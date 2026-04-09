# ADR-0001: Event-Driven Pipeline Architecture

## Status
Accepted

## Context

The content supply chain transforms studio-delivered mezzanine files into globally streamable content. The pipeline spans ingest, quality control, transcoding, packaging, DRM encryption, and global distribution. Each of these stages has different compute profiles, failure modes, and scaling requirements.

The previous synchronous, service-call-based pipeline architecture had several structural problems that became more acute as the catalog and release cadence grew:

**Cascading failures.** A transient failure in the transcoding layer caused ingest jobs waiting upstream to stall or time out. A single slow transcode run could hold up everything behind it in the queue because each stage called the next synchronously and waited for a response.

**Poor elasticity.** The pipeline was provisioned for average load. During global release events, when tens of titles might be published simultaneously, the synchronized call chain created a thundering herd problem. Scaling one stage without scaling all downstream stages offered no relief.

**Large blast radius.** A failure in packaging caused retries that propagated backward through the pipeline. A job that had successfully transcoded all renditions could be forced to restart from ingest because the packaging stage failed and there was no checkpoint to resume from.

**No natural backpressure.** If the transcoding farm was saturated, there was no mechanism for upstream stages to slow their output. Ingest continued accepting new jobs at full rate while the queue behind transcoding grew unbounded.

The pipeline needed to be restructured so that each stage was independently scalable, failures were isolated to the stage where they occurred, and the system could apply backpressure naturally without explicit orchestration.

## Decision

Adopt an event-driven pipeline architecture where each stage emits an immutable event to a durable message bus upon completion, and the next stage is triggered by consuming that event.

Each stage operates as an independent worker pool:
- Workers consume events from the bus when they have capacity
- Workers process their assigned work unit idempotently, so any job can be safely retried if a worker fails mid-execution
- Upon successful completion, a worker emits a completion event to trigger the downstream stage
- Upon failure, the event is returned to a retry queue with exponential backoff; it does not block other jobs

The message bus (Apache Kafka) provides durability and ordering guarantees. Events are not acknowledged until the downstream worker has successfully written its output to the content artifact store, ensuring no work is silently lost on worker failure.

Stages in the pipeline:
1. Studio delivery received
2. Ingest and integrity validation complete
3. Quality control (QC) check passed
4. Transcoding complete (one event per rendition, per codec)
5. Packaging complete
6. DRM encryption complete
7. Distribution to CDN origin complete
8. Global publish gate cleared
9. Playback available

Each event carries a content artifact identifier, a job identifier, and a checksum of the output. The event log is the authoritative record of pipeline progress for every title.

## Consequences

### Positive
- Each stage scales independently. During a peak release window, additional transcoding workers can be provisioned without any changes to the ingest or packaging stages.
- Failures are contained to the stage where they occur. A packaging failure does not affect in-progress transcode jobs; the transcode outputs are preserved and packaging retries without re-doing prior work.
- Natural backpressure. If packaging workers are at capacity, unconsumed events accumulate in the message bus. Upstream stages can observe consumer lag and self-throttle or trigger worker auto-scaling.
- The event log provides a complete audit trail of every pipeline stage for every title, which satisfies content liability and forensic investigation requirements.
- Reprocessing is straightforward. A bugfix to the packaging stage can be applied and a subset of titles can be reprocessed by replaying their packaging-input events without touching earlier stages.

### Negative
- The system is eventually consistent. There is no single coordinator that can answer "what is the current state of title X" without querying the event log or a derived state store. Operational tooling must account for this.
- Debugging a pipeline failure requires correlating events across multiple stages. Without strong observability tooling (distributed tracing keyed to the content artifact identifier), this is difficult.
- Idempotency must be designed into every stage explicitly. Workers that write to object storage or databases must handle the case where they are retried after a partial write.
- The message bus itself becomes a critical dependency. Kafka cluster health must be actively monitored and its operational footprint adds to the platform's SRE burden.

## Alternatives Considered

**Synchronous service call chain:** Each stage calls the next directly and waits for a response. Simpler to reason about locally but creates tight coupling, poor elasticity, and cascading failure behavior under load. Rejected because it reproduces the structural problems that motivated this decision.

**Workflow orchestration with a central DAG engine:** A single orchestrator tracks pipeline state and issues work to each stage. Provides better visibility into pipeline state than a pure event-driven approach, but the orchestrator becomes a bottleneck and a single point of failure. Considered as a hybrid where a lightweight orchestrator tracks aggregate title state without controlling individual stage execution.

**Batch processing (nightly jobs):** Acceptable for low-velocity catalogs, but cannot meet the release SLAs required for same-day global title availability. Rejected.
