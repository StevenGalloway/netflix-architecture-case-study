# ADR-0002: Immutable Content Artifacts

## Status
Accepted

## Context

The content supply chain produces a large number of intermediate and final artifacts for each title: mezzanine files, per-resolution encodes, CMAF packaged segments, DRM-encrypted outputs, and manifests. These artifacts are referenced by downstream systems, cached by CDN infrastructure, and consumed by playback clients.

Before adopting an immutability model, artifacts were written to named paths that could be overwritten. This created several operational and correctness problems:

**Silent corruption during reprocessing.** When a title was reprocessed due to a quality issue or pipeline bug, the new outputs were written over the existing files at the same storage path. CDNs and playback clients that had cached the old artifact continued serving it. Invalidating CDN caches is slow and not guaranteed to complete before the content is accessed again. In some cases, a partially overwritten artifact was served mid-reprocessing.

**Audit gaps.** When a question arose about which encode was active at a specific point in time, there was no way to answer it definitively. The storage layer only retained the most recent version of any artifact. This created liability exposure when content quality disputes arose with studio partners.

**Rollback complexity.** Rolling back to a previous version of a packaged artifact required restoring from a separately maintained backup, which had its own consistency and timing issues. There was no first-class concept of "version N of this artifact."

**Race conditions during concurrent pipeline runs.** When two pipeline jobs for the same title were running concurrently (which happened during retries or reruns), their outputs could interleave at the same storage path, producing a corrupted artifact that was a mix of outputs from two different pipeline executions.

## Decision

All content artifacts produced by the supply chain pipeline are immutable. Once written, an artifact is never overwritten or modified in place.

Each artifact is identified by a content-addressable key that includes the title identifier, the pipeline run identifier, the stage name, and a content hash of the output. Reprocessing a title produces new artifacts with new identifiers; it does not touch existing artifacts.

The content artifact store maintains a manifest of which artifact version is the current "active" version for each title and rendition. Updating the active version is an atomic metadata operation, not a file overwrite. CDN cache invalidation targets the manifest, not the artifact files themselves; artifact files are cached with long TTLs because they never change.

Artifact lifecycle management:
- Artifacts from the most recent pipeline run per title are retained indefinitely while the title is active in the catalog.
- Artifacts from prior pipeline runs are retained for 90 days after being superseded, to support rollback and forensic investigation.
- Artifacts for titles removed from the catalog are transitioned to cold storage and retained for the contractually required period before deletion.

## Consequences

### Positive
- Concurrent pipeline runs for the same title cannot corrupt each other's outputs; each run writes to its own uniquely keyed namespace.
- Rolling back to a prior version of a title's encodes is a metadata operation: update the active pointer to the previous artifact version. No file restoration required.
- The full history of all artifact versions for any title is available for audit and forensic purposes for the retention period.
- CDN caching becomes simpler and more aggressive. Artifact files are immutable and can be cached indefinitely; only the manifest has a short TTL.
- Content-addressable keys mean that if the same mezzanine input produces the same output (common during reruns with no pipeline change), duplicate artifacts can be detected and deduplicated at the storage layer.

### Negative
- Storage costs increase proportionally to the number of pipeline reruns per title. A title that is reprocessed five times retains five copies of all intermediate and final artifacts for the retention period. Active management of lifecycle policies is required to prevent unbounded storage growth.
- The artifact store metadata layer (the active version manifest) becomes a critical dependency. A failure there does not corrupt artifacts, but it can prevent playback from resolving the correct version.
- Content-addressable naming requires the pipeline to compute output hashes, which adds a small amount of CPU overhead per stage.

## Alternatives Considered

**Versioned object storage with server-side versioning:** Object storage versioning maintains prior versions automatically on overwrite. Simpler to implement but harder to operate: version proliferation in object storage namespaces degrades listing performance at scale, and server-side versioning does not provide the explicit pipeline-run-level lineage tracking needed for audit requirements.

**Mutable artifacts with explicit cache invalidation:** Overwrite artifacts in place and issue CDN invalidations on update. Simpler storage model but does not resolve the concurrent-write race condition, introduces CDN invalidation latency as a consistency gap, and provides no rollback capability without external backup infrastructure.
