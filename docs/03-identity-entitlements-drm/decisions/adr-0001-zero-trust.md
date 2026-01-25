# ADR-0001: Zero Trust Service Architecture

## Status
Accepted

## Context
Traditional perimeter security models fail in large distributed systems. Any internal service may become compromised.

## Decision
All service-to-service communication requires mutual TLS, short-lived identity tokens, and explicit authorization policies.

## Consequences

### Positive
- Prevents lateral movement
- Strong service identity guarantees
- Limits breach impact

### Negative
- Increased operational complexity
- More certificate and secret management
