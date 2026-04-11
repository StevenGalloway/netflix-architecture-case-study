# ADR-0001: Control Plane Isolation from the Playback Data Plane

## Status
Accepted

## Context

Netflix's product surface spans two fundamentally different operational domains that happen to be presented to users as a single application. The first is the **playback data plane**: the infrastructure that delivers video segments from CDN to the player. This system has extremely strict availability and latency requirements; a failure here means a subscriber cannot watch content. The second is the **control plane**: the infrastructure that determines what content is shown on the homepage, in what order, in what visual configuration, and under what experimental treatment. This system has high but less extreme requirements; a failure here means the homepage shows a less personalized view, but the subscriber can still watch content.

Before explicitly isolating these domains, they were architecturally entangled. The client application made a single session initiation request that combined the entitlement check, the manifest URL, and the homepage content configuration. The backend services that fulfilled this request shared infrastructure, service dependencies, and deployment pipelines.

This entanglement created cascading failure scenarios that violated the principle that playback should be the most protected capability in the platform:

**Personalization service outages blocked playback.** When the recommendation ranking service experienced elevated latency, the combined session initiation request timed out, preventing users from starting playback even though the video delivery infrastructure was fully operational. From the subscriber's perspective, the service was completely down, despite only the non-critical personalization layer being degraded.

**Experiment deployments affected playback availability.** The personalization and experimentation systems operate at high deployment velocity; new model versions and experiment configurations are deployed multiple times per week. Because deployment pipelines were shared with playback services, a bad personalization deployment could propagate to the playback path. Rollback of a personalization change required rolling back playback services, adding latency to the recovery.

**Capacity planning was conflated.** Scaling the personalization system to handle more simultaneous homepage requests required scaling playback infrastructure in lockstep, even when playback itself was not under pressure. Resource allocation became difficult to optimize because the two workloads had different load profiles and different scaling triggers.

**SLO ownership was ambiguous.** When a combined endpoint degraded, it was unclear whether the root cause was in the playback path or the personalization path. The on-call response was fragmented across multiple teams, each suspecting the other's domain as the cause.

## Decision

Strictly separate the control plane (personalization, recommendations, experimentation, UI configuration) from the data plane (playback session initiation, manifest delivery, DRM licensing) at every layer: network, service, data, and deployment.

**Separate client API calls.** The client application makes distinct, independent requests for playback session data and for UI/recommendation data. These requests have no shared dependencies at the application layer. A failure in either path does not affect the other at the API level.

**Separate service fleets.** No service instance or cluster serves both control plane and data plane traffic. Separate auto-scaling groups, separate Kubernetes namespaces, separate load balancers. A capacity event in personalization does not consume resources from the playback service pool.

**Separate data stores.** The feature stores, session stores, and caches used by the personalization system do not share infrastructure with the session and token stores used by the playback path. A slow query or a cache eviction storm in one domain does not affect the other.

**Separate deployment pipelines.** Personalization models, experiment configurations, and homepage layout changes are deployed through a pipeline that has no intersection with the playback service deployment pipeline. A bad personalization deployment can be rolled back without touching playback services, and vice versa.

**Separate SLO ownership.** The playback data plane has its own SLO set, its own error budget, and its own on-call rotation. The personalization control plane has its own SLO set, its own error budget, and its own on-call rotation. Incidents in each domain have a clear primary owner.

The client is responsible for handling the case where control plane data is unavailable. The client must implement a fallback rendering path (see ADR-0002: Graceful Degradation) that allows playback to proceed even when the control plane response is absent.

## Consequences

### Positive
- A complete personalization system outage does not prevent subscribers from watching content. Playback continues from the cached or default UI state while the control plane recovers.
- High-velocity personalization and experiment deployments are decoupled from the stability-focused playback deployment cadence. Neither side needs to slow down to accommodate the other's risk tolerance.
- Resource consumption in the personalization layer (which can be substantial during model serving or A/B test evaluation) does not compete with playback infrastructure.
- SLO attribution is clear. When an incident occurs, the responsible domain is identifiable from the first alert. On-call response is faster because the scope is bounded.
- Each domain can be capacity-planned and scaled independently based on its own load characteristics. The personalization layer may scale with the number of active browse sessions; the playback layer scales with the number of concurrent streams. These are different demand curves.

### Negative
- The client application must implement and maintain two independent request paths, two independent retry strategies, and two independent fallback behaviors. Client-side complexity increases.
- Data that was previously consistent within a single response (e.g., a homepage tile and the playback session for the content behind it) may now arrive from two separate responses with a small timing gap between them. Client-side state management must handle this gracefully.
- Isolation requires operational discipline to maintain. As the platform evolves, teams may be tempted to add shared dependencies between the two planes for convenience. Architectural governance is needed to enforce the boundary.
- Testing end-to-end behavior requires simulating both planes independently and in combination. Integration test coverage for the interaction between control plane content and playback session initiation is more complex to design.

## Alternatives Considered

**Combined endpoint with circuit breaker on personalization.** A single endpoint handles both playback and personalization. A circuit breaker on the personalization dependency allows the endpoint to return a degraded response (playback session only, no personalization data) when the personalization service is unavailable. Simpler client implementation, but the circuit breaker must be implemented and tuned at every service that calls personalization, creating distributed complexity. Also does not address the shared deployment pipeline risk.

**Separate services, shared deployment pipeline.** Split the API surface but continue using a shared deployment system. Reduces some failure coupling but retains the deployment risk: a bad configuration pushed through the shared pipeline can still affect both planes simultaneously.

**Monolithic combined service with feature flags.** A single service handles all requests. Feature flags control whether personalization is included in a given response. Does not address any of the failure isolation, scaling, or deployment decoupling goals. Rejected without extensive consideration.
