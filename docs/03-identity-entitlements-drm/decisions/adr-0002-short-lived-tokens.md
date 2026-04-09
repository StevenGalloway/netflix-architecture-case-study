# ADR-0002: Short-Lived Playback Tokens

## Status
Accepted

## Context

Playback access is controlled through tokens that are issued at session initiation and presented at several points in the playback flow: when fetching the manifest, when requesting video segments from the CDN, and when requesting a DRM license. These tokens are the mechanism by which the platform enforces that only authorized users on authorized devices can access content.

The design of these tokens involves a fundamental tradeoff between security and operational overhead. Long-lived tokens are simpler: issue once, use indefinitely, no refresh logic needed. Short-lived tokens are more secure but require the client to refresh them before they expire, adding round trips and complexity to the session management layer.

Before adopting short-lived tokens, the platform used session tokens with validity periods measured in hours or days. Several failure modes and threat scenarios exposed the limitations of this approach:

**Token replay attacks.** A token extracted from one session could be replayed from a different device or location within its validity window. Because the token was not bound to the originating device's characteristics, the platform had no way to distinguish a legitimate session from a replayed one. Content protection audit findings flagged this as a persistent vulnerability.

**Credential sharing at scale.** Long-lived tokens facilitated systematic credential sharing. A token issued to one subscriber account could be distributed and used by many unrelated sessions simultaneously, each within the token's validity window. Detecting this required complex behavioral heuristics that were reactive rather than preventive.

**Difficulty revoking access mid-session.** When a subscriber's account was suspended for a policy violation or payment failure, long-lived tokens that had already been issued remained valid. Revoking access required maintaining a revocation list that every token-validating service had to check on every request. At scale, this revocation list became operationally expensive to maintain and distribute.

**Insufficient precision for policy changes.** Content licensing windows sometimes close unexpectedly (a rights dispute, a takedown request). With long-lived tokens, content that should no longer be accessible could continue streaming for hours until tokens expired naturally, because there was no efficient mechanism to invalidate tokens in flight.

## Decision

All playback tokens are short-lived, device-bound, and scoped to a specific content title and session.

**Token lifetime:** Playback session tokens have a maximum lifetime of 7 minutes. Manifest access tokens have a lifetime of 4 minutes. DRM license requests must be made within 5 minutes of token issuance. These windows are calibrated to exceed the typical session initiation flow duration while minimizing the replay window.

**Device binding:** Every token is bound to a device fingerprint derived from hardware attestation data. The token includes a cryptographic claim over the device identifier. Token validation rejects tokens presented from a device whose fingerprint does not match the claim in the token. This binding is enforced at the DRM License Service and the Manifest Service independently.

**Session scoping:** Tokens are scoped to a specific playback session identifier, a specific title, and a specific user account. A token issued for one title cannot be used to access a different title. A token issued for one session cannot be reused after that session ends.

**Continuous refresh for in-progress sessions:** The client player receives a refreshed set of tokens before the current tokens expire. Refresh requests are made 60 seconds before token expiry. The session management infrastructure issues new tokens only if the session is still active, the subscription is still valid, and the content license is still active at the time of refresh. If any of these conditions fails at refresh time, no new tokens are issued and playback naturally terminates when the current tokens expire.

**Token issuance rate limiting:** Token issuance for a given account is rate-limited. Generating a large number of tokens rapidly (behavior consistent with automated scraping or systematic credential sharing) triggers review and potential account action.

## Consequences

### Positive
- The replay window for a stolen token is at most 7 minutes, after which the token is worthless. This substantially reduces the value of extracting tokens from client traffic or device memory.
- Device binding means a token stolen from one device cannot be used on a different device. The attacker must also compromise the device's hardware attestation to fabricate a matching fingerprint claim.
- Revoking access to a subscriber account no longer requires a revocation list. Simply not issuing new tokens at the next refresh cycle is sufficient; the subscriber's access terminates naturally within the token lifetime window.
- When a content license expires or is revoked, the continuous refresh mechanism naturally terminates access at the next refresh cycle without requiring active token invalidation. The access termination latency is bounded by the token lifetime, not by cache TTL or revocation list propagation time.
- Session scoping prevents a token obtained for one title from being used to access the catalog more broadly.

### Negative
- Each token refresh requires a round trip from the client to the Session Management Service. For a long-form viewing session, this means several refresh calls per hour per active session. At hundreds of millions of concurrent sessions, this is a non-trivial load on the token issuance infrastructure. The Session Management Service must be dimensioned for this steady-state refresh load.
- Token expiry at an inopportune time (refresh fails due to a network error or service degradation) can interrupt an active playback session. The client must implement retry logic with sufficient buffer time before expiry to minimize user-visible interruptions.
- Device binding based on hardware attestation is only as strong as the attestation mechanism on each platform. Device platforms that do not support strong hardware attestation (some older or low-cost devices) require compensating controls or weaker binding guarantees.
- Debugging token validation failures requires correlating the token's issuance time, device binding claim, and session state across multiple services. The operational tooling to support this investigation must be purpose-built.

## Alternatives Considered

**Long-lived tokens with revocation lists:** Issue tokens with multi-hour validity and maintain a centralized revocation list that all validating services check. Reduces refresh load on the token issuance infrastructure but introduces the operational cost and consistency challenges of a distributed revocation list. Revocation latency (the time between a revocation decision and full enforcement) is bounded by revocation list propagation delay, which can be minutes. Rejected because it does not adequately address token replay or the precision of license revocation.

**Token-less signed URL approach:** Generate signed, time-limited URLs for manifest and segment access at the CDN layer. Simpler than a full token infrastructure but provides no mechanism for device binding, session scoping, or continuous validation during a session. The URL signature cannot carry the context needed for fine-grained policy enforcement. Rejected as insufficient for DRM compliance obligations.

**Opaque session cookies with server-side validation:** Issue opaque session identifiers that are looked up in a centralized session store on every request. Provides strong revocation capability but makes every content request dependent on a session store lookup, adding latency and creating a potential bottleneck. The session store becomes a high-throughput, high-availability critical dependency for every playback request in the platform.
