# ADR-0003: Policy-Based Global Publishing

## Status
Accepted

## Context

Making a title available for playback is not a single boolean switch. The act of publishing involves coordinating across a matrix of constraints that vary by title, by geography, by device type, by subscription plan, and by calendar date. A title may be licensed for streaming in the United States but not in Canada. A theatrical window may prohibit streaming until 90 days after a film's release date. A specific audio track may not be ready in time for a title's initial launch. A promotional launch may require simultaneous availability in 190 countries at a precise date and time.

The previous publishing model used a combination of manual configuration steps and ad-hoc service calls to enable availability on a per-title basis. This approach failed in several ways:

**Race conditions on global release.** Coordinating simultaneous availability across multiple regions required operators to manually enable each region in sequence. The window between the first and last region being enabled was measurable in minutes. During that window, users in different regions had inconsistent access to the same title, and some users were able to initiate playback before the title was fully ready in their region.

**Premature exposure.** Because enabling availability required multiple discrete steps, it was possible for a title to become partially available before all checks were complete. A manifest could be accessible before DRM keys had been distributed, or a title could appear in search results before its encodes had passed quality validation.

**Configuration errors on contractual constraints.** Regional licensing constraints were applied manually per title. Human error in this process resulted in titles being made available in regions where they were not licensed, which created legal liability. Discovering these errors required reactive auditing after the fact.

**No atomic rollback.** If a publishing error was discovered after a title went live, reverting required the same sequence of manual operations in reverse, applied region by region, while the title was actively being watched. There was no operation equivalent to "unpublish this title globally and atomically."

## Decision

All title availability is governed by an explicit publishing policy document that defines the complete set of conditions that must be satisfied before a title becomes playable in any context.

A publishing policy is a versioned, structured document associated with each title. It declares:
- **Availability windows:** The date range during which the title is available for streaming, per region
- **Geographic entitlements:** The set of countries and regions where the title is licensed for streaming
- **Device entitlements:** Which device tiers and DRM platforms are authorized to play the title
- **Plan entitlements:** Which subscription plans include access to the title (standard, premium, ad-supported)
- **Content readiness gates:** The list of pipeline artifacts that must be present and validated before the title can be published (e.g., all required encodes present, all required audio tracks packaged, DRM keys distributed)
- **Release coordination flags:** Whether a global simultaneous release is required (meaning no region becomes available until all regions are ready)

The Global Publish Service evaluates the policy document atomically before making a title available in any context. All conditions must pass; a partial policy pass does not result in partial availability. If any condition fails, the title remains unavailable and an alert is raised to the content operations team.

Publishing a title is the act of committing a policy document and having it pass evaluation, not the act of flipping individual service switches. Unpublishing is the act of retracting the policy commitment, which the publish service enforces across all downstream systems within its propagation SLA.

Policy documents are version-controlled. Prior versions are retained for audit purposes.

## Consequences

### Positive
- Global simultaneous release is a first-class operation. The publish service coordinates across all regions and confirms they are ready before making any region available, eliminating the partial-availability window.
- Regional licensing constraints are declared in the policy document, not applied manually. The policy is reviewed and approved before publishing; errors are caught at review time, not discovered reactively after the title goes live.
- Unpublishing is atomic. Retracting a policy document takes the title offline across all regions and all systems that respect the publish service, within the propagation SLA.
- The policy document provides a complete, auditable record of every availability decision made for every title, including who authored and approved the policy.
- Content readiness gates in the policy mean a title cannot go live until all its required artifacts have been validated. Premature exposure due to incomplete pipeline execution is prevented by design.

### Negative
- The Global Publish Service is on the critical path for every title launch. Its availability directly affects the ability to release content. It must be operated with high reliability and its own multi-region failover strategy.
- Policy document authoring requires training. Content operations teams must understand the policy schema and its implications. Incorrect policy configuration, while less catastrophic than the previous system, can still result in incorrect availability.
- Policy evaluation adds latency to the publish flow. For most titles this is acceptable, but for live events where millisecond-level publish timing matters, the policy evaluation path must be optimized or pre-evaluated.
- Complex contractual constraints that change frequently (e.g., rights that vary by subscriber tier or expire mid-catalog-period) require active policy document updates and re-evaluation. Without tooling support, this creates ongoing operational burden for the content operations team.

## Alternatives Considered

**Entitlement flags per service (status quo):** Each downstream service (playback, search, browse, CDN) maintains its own availability state, updated through direct API calls. Simple per-service but requires coordinating many independent systems for every publish and unpublish operation, with no atomicity guarantee across them.

**Feature flags with a release management tool:** General-purpose feature flag infrastructure can govern title availability with percentage rollouts and targeting rules. More flexible than a bespoke publish system but lacks the domain-specific concepts (geographic entitlements, content readiness gates, simultaneous global release) needed to support the contractual and operational requirements of the content supply chain.

**Manual operations with a pre-flight checklist:** Retain manual publishing but enforce a structured checklist that an operator must complete and approve before proceeding. Reduces errors but does not eliminate the race conditions or the lack of atomicity that drove this decision.
