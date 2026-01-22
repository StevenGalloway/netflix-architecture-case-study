# ADR-0002: Graceful Degradation Strategy

## Status
Accepted

## Context
Personalization failures must never block user access to content.

## Decision
If personalization components fail, clients fall back to:
- cached recommendations
- precomputed global trending rails
- minimal default UI

## Consequences

### Positive
- Protects core user experience
- Reduces blast radius of failures

### Negative
- Less optimal personalization during incidents
