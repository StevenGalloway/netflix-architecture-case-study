# ADR-0001: Multi-CDN Steering Strategy

## Status
Accepted

## Context

Streaming at Netflix's scale means that a single CDN provider creates a single point of failure for the entire global playback experience. CDN outages are not rare events; they happen multiple times per year across any provider at global scale.

Beyond resilience, single-CDN architectures also create leverage imbalance in commercial negotiations and limit the platform's ability to optimize for performance in specific geographies where one provider has a stronger edge network.

The playback system needs to deliver signed manifests and video segments to hundreds of millions of devices globally, often simultaneously during peak release windows, with TTFF SLOs of <= 1.5 seconds at p95.

## Decision

Adopt an active multi-CDN model with real-time health-based traffic steering. All CDN providers serve live production traffic simultaneously. A centralized CDN Steering Service continuously evaluates provider health, latency, and error rate signals and adjusts routing weights in real time.

Key implementation points:
- Maintain contracts with a minimum of three tier-1 CDN providers
- Operate a CDN Steering Service that evaluates health signals every 30 seconds
- Use origin shielding behind all CDN providers to protect the origin cluster from cache-miss storms during steering changes
- Signed manifests and segment URLs are CDN-agnostic; the steering service substitutes the base URL at the edge without re-signing

## Consequences

### Positive
- Eliminates CDN as a single point of failure; any single provider can go to zero without a customer impact event
- Enables geographic optimization by routing to the strongest provider per region
- Reduces commercial risk through competitive pricing pressure across providers
- Enables controlled A/B testing of CDN providers by splitting traffic at the steering layer

### Negative
- Operational complexity increases significantly: three CDN relationships, billing reconciliations, and provider-specific debugging
- Cache fragmentation: each CDN maintains an independent cache, reducing overall cache hit rates compared to a single-CDN model
- Steering logic must handle edge cases: oscillation prevention, minimum weight floors, and cache warm-up periods after traffic shifts
- Origin shield costs increase proportionally to the number of CDN providers

## Alternatives Considered

**Primary + Failover CDN:** Simpler operationally, but failover is reactive (response time measured in minutes, not seconds) and the standby CDN cache is always cold, degrading performance during failover events.

**Single CDN with redundant PoPs:** Relies entirely on one provider's internal resilience; does not protect against provider-wide outages or commercial disputes.
