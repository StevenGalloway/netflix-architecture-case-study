# ADR-0001: Zero Trust Service Architecture

## Status
Accepted

## Context

At the scale of Netflix's microservices architecture, hundreds of distinct services communicate with each other across shared compute infrastructure. The traditional security model for internal service communication assumed that traffic within the data center perimeter was implicitly trusted: if a request arrived from an internal IP address, it was assumed to originate from a legitimate service and was processed without further authentication.

This perimeter trust model has a fundamental structural weakness. Any compromised internal service becomes a pivot point for lateral movement across the entire platform. An attacker who gains code execution in a low-privilege service can freely call high-privilege services as long as those calls originate from within the trusted network perimeter. The blast radius of any single compromise is bounded only by network segmentation, which is operationally difficult to enforce at granular service-to-service levels in a dynamic microservices environment.

Several specific incidents across the industry illustrated this failure mode concretely:

**Supply chain compromises** targeting build pipelines and dependencies have produced compromised internal services that appeared legitimate from a network perspective but were executing attacker-controlled code.

**Credential theft** from configuration files, environment variables, and secrets management systems gave attackers long-lived credentials that could be replayed from arbitrary network locations or from within the compromised service.

**Insider threat scenarios** where engineers with network access could craft requests to internal services that lacked caller authentication, bypassing application-level authorization checks.

The system needed a security model that did not assume any service was trusted by virtue of its network location. Trust must be established explicitly on every request, based on verified service identity, and authorization must be enforced at the destination service regardless of the caller's network origin.

## Decision

Adopt Zero Trust as the security model for all service-to-service communication within the platform.

Zero Trust in this context means three things apply to every internal service call:

**Mutual TLS (mTLS) for every connection.** Both the calling service and the receiving service present cryptographic certificates that prove their identity. Neither side is trusted solely because of its IP address or network location. Certificates are short-lived (24-hour TTL) and issued by an internal certificate authority. Services that cannot present a valid certificate cannot establish a connection.

**Short-lived identity tokens for every request.** In addition to the transport-layer identity established by mTLS, each request carries a signed identity token that identifies the calling service and the specific action being requested. These tokens are issued by a central identity service, are scoped to a specific service pair and action, and expire within minutes. A token that is stolen from in-flight traffic is unusable after expiry and cannot be reused for a different action or a different target service.

**Explicit authorization policy for every service.** Each service maintains an authorization policy that declares which other services are permitted to call which of its endpoints. Calls from services not in the policy are rejected at the service boundary, regardless of whether they present a valid certificate and token. Authorization policies are stored as code, reviewed, and version-controlled.

No exceptions to these three requirements are made for internal traffic. The assumption is that any service may be compromised, and the security model must limit the damage a compromised service can do.

## Consequences

### Positive
- A compromised service cannot escalate its access by calling other services it is not authorized to call. Lateral movement requires compromising the authorization policy itself, which is version-controlled and requires review.
- The blast radius of any single service compromise is bounded to the services that service is explicitly authorized to call. An attacker who controls one service cannot freely pivot to every other service in the platform.
- mTLS provides strong proof of service identity, eliminating the class of attacks based on IP spoofing or network-level impersonation.
- Short-lived tokens significantly reduce the value of credential theft. A stolen token is valid for minutes, not months.
- Authorization policies are explicit and auditable. The access control model for every service pair is documented in code and can be reviewed, diffed, and audited.

### Negative
- Certificate and token management adds operational complexity. Every service must participate in the certificate rotation lifecycle. Certificate expiry without renewal is an availability risk that must be actively monitored.
- mTLS adds latency to every service call. For most internal calls this is negligible (< 1ms once the connection is established and the session is reused), but connection establishment overhead matters for services that make many short-lived connections.
- Implementing Zero Trust requires retrofitting the entire service fleet. Services built before this model was adopted must be migrated to use mTLS and identity tokens. This is a multi-year migration effort, not a single deployment.
- Debugging service communication failures requires understanding the certificate and authorization layers. A misconfigured authorization policy produces the same symptom as a network partition from the caller's perspective: a connection refused or a 403 response.
- The internal certificate authority and identity service are themselves high-criticality infrastructure. Their availability must be maintained at the same level as the highest-criticality service in the platform.

## Alternatives Considered

**Network segmentation with per-service VLANs:** Restricts which services can communicate at the network layer. Provides isolation but does not provide service identity verification (any compromised host in the allowed network segment can call the target service). Does not scale operationally as the number of services and communication paths grows.

**API gateway with centralized authentication:** All internal service calls route through a central gateway that authenticates the caller. Provides a single enforcement point but the gateway becomes a bottleneck and a single point of failure. Does not address the lateral movement problem once a service has authenticated through the gateway.

**Perimeter trust with enhanced monitoring:** Retain implicit internal trust but invest in behavioral monitoring to detect lateral movement after the fact. Reduces preventive protection and accepts that breaches will occur, focusing on detection speed. Rejected because the platform's content protection obligations require preventive controls, not just detection.
