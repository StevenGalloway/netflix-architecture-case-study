# ADR-0001: Control Plane Isolation

## Status
Accepted

## Context
Personalization systems change rapidly and contain high experimental risk. Playback systems require extreme stability.

## Decision
Strictly separate control plane (UI, recs, experimentation) from data plane (playback delivery).

## Consequences

### Positive
- Protects playback reliability
- Enables high-velocity experimentation

### Negative
- Requires more sophisticated client logic
- Additional system boundaries
