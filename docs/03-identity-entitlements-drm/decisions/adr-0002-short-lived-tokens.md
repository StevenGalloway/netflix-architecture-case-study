# ADR-0002: Short-Lived Playback Tokens

## Status
Accepted

## Context
Long-lived playback tokens increase the window for theft and replay attacks.

## Decision
All playback and manifest tokens are short-lived and bound to device, user, and region.

## Consequences

### Positive
- Limits replay attacks
- Improves content protection
- Enables precise policy enforcement

### Negative
- Requires token refresh mechanisms
- Slight increase in control-plane traffic
