# ADR-0001: Apache Iceberg as the Lakehouse Table Format

## Status
Accepted

## Context

The data platform must store and serve petabytes of event data, feature tables, and analytical datasets across multiple teams. Historical approaches relying on Hive-style partitioned directories on object storage have several limitations that have caused operational pain at scale:

- Schema changes require full table rewrites or partition-level workarounds
- No atomicity: concurrent readers can observe partially written partitions during Spark job execution
- No time-travel: debugging a data quality issue requires preserving external snapshots manually
- Partition management is manual and error-prone, leading to data skew and query performance degradation
- No row-level deletes or updates: removing a user's data for GDPR compliance requires full partition rewrites

A unified table format is needed that supports ACID semantics, schema evolution, and efficient query planning across both the Spark batch layer and the Flink streaming layer.

## Decision

Adopt Apache Iceberg as the standard table format for all managed datasets in the data platform lakehouse.

All new analytical tables, feature store offline partitions, and event archive tables will be written using Iceberg. Existing Hive-format tables will be migrated on a domain-by-domain schedule prioritizing the most frequently modified tables first.

Key Iceberg capabilities relied upon:
- **ACID transactions:** Concurrent writes from multiple Spark jobs do not create partial reads for downstream consumers
- **Hidden partitioning:** Queries do not require partition predicates; Iceberg computes partition pruning automatically
- **Schema evolution:** Adding, renaming, or dropping columns does not require rewriting existing data files
- **Time-travel:** Any table can be queried at any historical snapshot, enabling debugging and compliance audits
- **Row-level deletes:** GDPR delete requests can be executed as row-level deletes, committed as delete manifests, and compacted on a schedule

## Consequences

### Positive
- Teams can evolve table schemas without coordinating with downstream consumers on a migration window
- Data quality investigations can query the exact state of any table at any past point in time
- GDPR and compliance deletions are first-class operations, not batch rewrite jobs
- Concurrent Spark jobs writing to the same table do not create consistency hazards for downstream readers

### Negative
- Iceberg metadata (manifests, snapshot logs) accumulates over time and requires active compaction and expiry jobs to prevent performance degradation
- Teams must learn Iceberg-specific concepts (snapshot isolation, manifest files, hidden partitioning) that differ from their Hive experience
- Flink-to-Iceberg streaming writes require careful configuration of commit intervals and checkpoint alignment to maintain exactly-once semantics

## Alternatives Considered

**Apache Hudi:** Provides upsert semantics and incremental processing, but has a steeper operational curve and less mature Flink integration at the time of adoption.

**Delta Lake:** Strong ecosystem integration with Databricks but introduces a proprietary dependency that conflicts with the platform's preference for cloud-neutral, open-source tooling.

**Hive-partitioned Parquet:** Lowest migration cost but does not solve ACID, schema evolution, or time-travel requirements. Ruled out as a long-term foundation.
