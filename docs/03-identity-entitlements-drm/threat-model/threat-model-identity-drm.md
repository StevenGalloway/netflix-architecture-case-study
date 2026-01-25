# Threat Model: Identity, Entitlements & DRM

## Assets
- User credentials
- Playback tokens
- DRM keys
- License policies
- Encryption infrastructure

## Entry Points
- Authentication endpoints
- License issuance APIs
- Operator consoles
- CI/CD pipelines

## Trust Boundaries
- Client → Auth perimeter
- Internal services
- DRM provider boundary
- Key vault / HSM boundary

## Threats (STRIDE)

### Spoofing
Fake clients requesting licenses.

### Tampering
Modification of entitlement policies.

### Repudiation
Disputes over license issuance actions.

### Information Disclosure
Leakage of keys or tokens.

### Denial of Service
Flooding auth or DRM services.

### Elevation of Privilege
Unauthorized policy changes.

## Mitigations
- mTLS & strong identity verification
- HSM-backed key storage
- RBAC & least privilege access
- Rate limiting & anomaly detection
- Comprehensive audit logging

## Residual Risk
Zero-day vulnerabilities and insider threats remain.
