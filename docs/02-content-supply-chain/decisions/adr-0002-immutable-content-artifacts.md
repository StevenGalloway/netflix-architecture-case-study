# ADR-0002: Immutable Content Artifacts

## Status
Accepted

## Context
Frequent reprocessing, retries, and versioning of content require full traceability and auditability.

Mutable content objects increase the risk of silent corruption and complicate forensic analysis.

## Decision
All content artifacts (mezzanine, encodes, packages) are immutable. New versions are created for any change.

## Consequences

### Positive
- Strong audit trail and reproducibility
- Safe rollback and replay
- Simplifies debugging and compliance

### Negative
- Increased storage costs
- Requires strict lifecycle management
