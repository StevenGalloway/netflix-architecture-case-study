# Identity, Entitlements and DRM Platform

## Purpose

This domain governs who can watch what, on which device, in which location, and under what conditions, while ensuring content protection and regulatory compliance.

Security must be uncompromising, yet latency must remain imperceptible to users. Playback availability remains the highest priority, but security must fail closed: unauthorized playback is never acceptable, while brief availability degradation is preferable to a content breach.

---

## User Journey

1. Client authenticates the user and obtains an auth token
2. Client requests a playback session for a specific title
3. Entitlement service validates subscription, plan, region, and concurrency rules
4. Playback session token is issued (short-lived, device-bound)
5. Client requests a DRM license using the playback token
6. DRM service validates the token and retrieves encryption keys from the key vault
7. License is issued to the device; secure playback begins

---

## Architecture Diagram

See `diagrams/drm-license-flow.mmd`

---

## Key Design Decisions

| Decision | Rationale |
|---------|-----------|
| Zero Trust internal architecture | Prevent lateral movement and minimize breach impact; no implicit trust between services |
| Short-lived playback tokens | Tokens expire in minutes, limiting the window for replay or theft attacks |
| Device-bound DRM licenses | Keys are bound to device attestation; cannot be transferred to unauthorized devices |
| Policy-driven entitlements | Support regional availability, plan tiers, concurrent stream limits, and promotional grants in a unified policy engine |
| Fail-closed security model | When the entitlement or DRM service is ambiguous, deny playback rather than allow unauthorized access |

---

## Pros / Cons

### Pros
- Strong content protection posture aligned with studio licensing requirements
- Fine-grained entitlement control supporting complex plan and regional rules
- High auditability: every license issued is recorded with device, user, and content identifiers
- Regulatory compliance for geofenced and rights-limited content

### Cons
- Latency sensitivity: each additional hop in the auth/DRM path adds to TTFF; services must be deployed close to playback origin
- Complex policy lifecycle: entitlement rules must be versioned, tested, and rolled back safely
- Certificate and secret management overhead increases with service count under Zero Trust

---

## Failure Modes and Mitigation

| Failure | Mitigation |
|---------|-----------|
| License service outage | Active-active multi-region deployment; licenses cached on device for in-progress sessions |
| Entitlement service degradation | Read-through cache with short TTL; stale-while-revalidate pattern for active sessions |
| Key vault unavailability | HSM cluster with regional replicas; keys pre-fetched and cached in DRM service memory |
| Token replay attack | Short token TTL (< 5 minutes); token binding to device fingerprint; replay detection log |
| Credential stuffing | Rate limiting, CAPTCHA, device attestation challenges, and anomaly detection on login patterns |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| DRM license issuance latency (p99) | <= 200 ms | 30-day rolling |
| Entitlement validation latency (p99) | <= 50 ms | 30-day rolling |
| Auth/DRM service availability | >= 99.99% | 30-day rolling |
| License issuance success rate | >= 99.95% | 30-day rolling |

---

## Security Considerations

- All service-to-service calls use mutual TLS with short-lived identity certificates
- Encryption keys are stored in HSM-backed key vaults; application services never access raw keys
- PII minimization: playback tokens carry only the minimum claims required for DRM policy evaluation
- Audit log of all license issuances retained for 90 days minimum
- Regular penetration testing of the token issuance and DRM license flows

---

## Operational Artifacts

- `decisions/adr-0001-zero-trust.md`
- `decisions/adr-0002-short-lived-tokens.md`
- `threat-model/threat-model-identity-drm.md`
- `threat-model/abuse-cases.md`
