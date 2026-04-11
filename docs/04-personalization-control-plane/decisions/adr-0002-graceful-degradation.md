# ADR-0002: Graceful Degradation Strategy for the Control Plane

## Status
Accepted

## Context

The control plane isolation decision (ADR-0001) establishes that the personalization system must not block playback. This creates a design obligation: the client and the serving infrastructure must define precisely what happens when personalization components are unavailable, and that behavior must be implemented as a first-class feature, not an afterthought.

Graceful degradation is easy to state as a principle and hard to implement well. The failure modes that need to be handled span a wide range of severity:

- The recommendation ranking model serving fleet is completely unavailable
- The feature store is returning stale data from 30 minutes ago
- The experiment evaluation service is timing out for 5% of requests
- The homepage layout service is returning error responses for a specific device type
- A specific A/B test variant has a bug that causes it to render incorrectly on smart TVs
- The entire personalization control plane is down during a region evacuation

Each of these scenarios requires a different response from the client and the serving infrastructure. Some warrant falling back to cached data; others warrant falling back to a precomputed default; others require the experiment to be halted immediately while the rest of the page renders normally.

Before a formal degradation strategy was defined, client teams handled each of these scenarios in different ways. Some clients showed blank recommendation rails when personalization failed. Some clients showed an error state that blocked navigation. Some clients retried indefinitely, causing latency spikes. The behavior was inconsistent across device platforms and was not tested systematically. Incidents in the personalization system produced a wide range of subscriber experiences, from mildly degraded to completely broken, depending on which device the subscriber was using.

## Decision

Define a tiered degradation hierarchy for the personalization control plane, with explicit fallback states at each tier. All client applications and the serving infrastructure implement this hierarchy consistently.

The degradation hierarchy is ordered from least to most severe:

**Tier 1: Stale cache.** Serve personalization data from the local response cache if it is within the maximum stale TTL (15 minutes for homepage rails). This is invisible to the subscriber; the content is slightly less fresh but the experience is unaffected. This is the first fallback and is applied automatically by the serving layer when the upstream service is slow or returning errors for a given request.

**Tier 2: Precomputed nearline recommendations.** If the local cache is expired or missing, serve from a precomputed set of recommendations that is refreshed on a 4-hour cycle and stored in a durable, highly available store separate from the online serving infrastructure. The nearline recommendations are computed for each subscriber using a batch model and represent a reasonable but less real-time personalized experience. This fallback is applied server-side when the online recommendation service is degraded.

**Tier 3: Global trending rails.** If nearline recommendations are also unavailable (the nearline store is degraded), serve a set of globally trending content rails that are precomputed for each catalog region. These rails are not personalized but they represent coherent, relevant content for the subscriber's region. They are stored with very high availability and very aggressive caching at the CDN edge. This fallback represents a noticeably degraded personalization experience but a fully functional browsing and playback experience.

**Tier 4: Minimal default UI.** If no content rails can be served from any source, the client renders a minimal homepage UI that includes the subscriber's continue-watching list (fetched from a separate, highly available watch history service that is not part of the personalization control plane) and a static "popular titles" section compiled manually by the content team. This fallback requires no real-time personalization infrastructure to be available.

**Experiment handling during degradation:** Experiments are applied at the serving layer before personalization data is returned to the client. If the experiment evaluation service is unavailable, the serving layer falls back to the control arm (no treatment applied) rather than blocking the response or applying a random treatment. Partially applied experiments are not acceptable; if the experiment variant cannot be fully evaluated for a given request, that request receives the control experience.

Fallback state transitions are automatic and applied by the serving infrastructure without requiring client-side knowledge of which tier is active. The client receives a response that indicates the data source tier (for logging purposes) but renders the experience without branching on the tier value.

All fallback data sources are tested during chaos engineering exercises on the same schedule as the primary serving path.

## Consequences

### Positive
- The subscriber experience during personalization system failures degrades gradually and predictably. At no tier does a failure cause an inability to browse or watch content.
- Blank or error states in the recommendation rails are eliminated as a failure mode. There is always a valid content set to display, even if it is not personalized.
- Experiment failures do not cause experience breakage. Falling back to the control arm means the subscriber receives a known-good experience, which is the correct behavior for an experiment failure.
- The degradation behavior is tested explicitly in the chaos engineering program, so the team has production-validated evidence that each fallback tier works as designed.
- The precomputed fallback data sources (nearline recommendations, global trending rails) serve as a natural load shedding mechanism. During an incident, traffic that would have hit the online personalization infrastructure is absorbed by the precomputed stores, reducing load on a system that is already struggling.

### Negative
- Maintaining four distinct fallback data sources adds storage cost and operational complexity. Each fallback store must be treated as a production system with its own health monitoring, capacity planning, and update pipeline.
- The 4-hour refresh cycle for nearline recommendations means that during extended outages, the fallback data becomes increasingly stale. For short outages (< 4 hours), the nearline data is acceptably fresh. For longer outages, the global trending rails become the effective experience for most subscribers.
- Fallback state transitions must be carefully instrumented. If the system silently falls back to a degraded tier without detection, the incident may go unnoticed for longer than necessary. Metrics that track which tier is actively serving each request must be monitored and alerted on.
- Testing the interaction between the degradation tiers and the client rendering layer across all supported device platforms is a continuous engineering investment. Each platform has different caching behavior and must be tested independently.

## Alternatives Considered

**Single fallback (static default page):** When personalization fails, display a fixed default page that does not change. Simpler to implement and operate, but represents a more severe degradation than necessary for most failure scenarios. Also does not leverage the precomputed data infrastructure that can serve a reasonably personalized experience even when the online serving path is down.

**Client-side fallback with local storage:** Cache the most recent personalization response in the client application's local storage. On failure, serve from local cache with no TTL limit. Simpler server-side behavior but creates inconsistent experiences across clients that have different local cache states (a new device, a cleared cache, a first-time login). Also does not handle the case where the client has no prior cached response.

**No explicit degradation strategy (rely on retries).** When the personalization service is unavailable, the client retries until it succeeds. Acceptable for very short transient failures but produces timeouts and blocked UI for anything beyond a few seconds. Rejected because it conflates "personalization service is slow" with "the subscriber cannot browse," which is the failure mode this entire decision is designed to prevent.
