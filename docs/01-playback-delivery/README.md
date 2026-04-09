# Playback Delivery Platform

## Purpose

Deliver globally low-latency, high-availability streaming with predictable startup times and minimal buffering. This domain is the most latency-sensitive system in the platform and carries the highest availability requirement.

Every second of startup time and every fraction of a percent of rebuffering is measurable in engagement and retention data. Engineering decisions here are guided by player telemetry and continuously validated through QoE SLOs.

---

## User Journey

Launch App → Request Playback Session → Verify Entitlement → Receive Signed Manifest → Fetch Segments from CDN → Begin Streaming → Send QoE Telemetry Continuously

---

## Architecture Diagrams

- `diagrams/playback-sequence.mmd` - end-to-end playback session initiation sequence
- `diagrams/multi-cdn-routing.mmd` - CDN steering and origin shield topology

---

## Key Design Decisions

| Decision | Rationale |
|---------|-----------|
| Multi-CDN with real-time steering | Eliminate CDN as a single point of failure; enable geographic performance optimization |
| Tokenized manifests and segments | Prevent unauthorized access without per-request server-side validation |
| Origin shielding and cache hierarchy | Protect origin from cache-miss storms during CDN failover or traffic shifts |
| Closed-loop QoE optimization | Feed player telemetry back into CDN steering and bitrate adaptation decisions |
| Separation from control plane | Prevent personalization and experiment system failures from affecting playback |

---

## Pros / Cons

### Pros
- Extremely fast startup times achievable at global scale
- High resilience through provider and geographic redundancy
- Reduced vendor lock-in and commercial leverage through multi-CDN contracts
- Real-time responsiveness to CDN degradation without manual intervention

### Cons
- Operational complexity of managing multiple CDN relationships and steering logic
- Cache fragmentation across CDN providers reduces overall cache efficiency
- Signed manifest infrastructure adds latency to the session initiation path if not carefully optimized

---

## Failure Modes and Mitigation

| Failure | Blast Radius | Mitigation |
|---------|-------------|-----------|
| CDN provider outage | Regional to global | Automatic traffic shift to healthy providers via CDN Steering Service |
| Origin overload | All playback globally | Origin shield absorbs cache-miss load; request collapsing limits origin fan-out |
| Manifest signing service degradation | New session starts | Multi-region signing service deployment; circuit breaker with cached token fallback |
| Regional cloud outage | All sessions in region | Global traffic evacuation; see `05-global-resilience/runbooks/region-evacuation.md` |
| DRM license service degradation | Secure content playback | Multi-region DRM cluster; cached licenses honored for active sessions |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| TTFF (p95) | <= 1.5 seconds | 30-day rolling |
| TTFF (p50) | <= 0.8 seconds | 30-day rolling |
| Rebuffer Ratio | <= 0.3% of watch time | 30-day rolling |
| Playback Error Rate | <= 0.1% of sessions | 30-day rolling |
| CDN Cache Hit Rate | >= 95% | 7-day rolling |

---

## Security

- Signed manifests and segment URLs: time-limited tokens prevent hotlinking and unauthorized access
- DRM enforcement: all premium content requires valid license from the DRM License Service
- Zero Trust service authentication: mTLS between all internal services in the playback path
- No PII in CDN-routed paths: all user-specific context resolved server-side before token issuance

---

## Operational Artifacts

- `decisions/adr-0001-multi-cdn-steering.md`
- `runbooks/cdn-degradation.md`
