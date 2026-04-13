# Observability and Quality of Experience (QoE)

## Purpose

Observability and QoE provide real-time insight into system health and customer experience, enabling rapid detection, diagnosis, and correction of issues across the entire platform.

This domain closes the feedback loop between architecture, operations, and user satisfaction. The primary measure of success is not server uptime but whether customers are having good playback experiences.

---

## Design Objectives

- Measure what users experience, not just what servers report
- Detect anomalies before customers report them or open support tickets
- Correlate signals across the entire request lifecycle from client to origin
- Enable fast root cause analysis by reducing mean time to diagnosis
- Drive SLO-based engineering decisions: error budgets inform deployment and feature risk

---

## Core Telemetry Signals

| Signal | Examples | Collection Method |
|--------|---------|------------------|
| QoE Metrics | TTFF, rebuffer ratio, bitrate, error codes | Client-side SDK telemetry |
| Infrastructure Metrics | Latency, error rate, throughput, CPU, memory | Service-side instrumentation |
| Distributed Traces | End-to-end request traces across service boundaries | OpenTelemetry / trace sampling |
| Structured Logs | Application events, access logs, audit logs | Log shipper sidecars |
| Business Signals | Plays started, engagement rate, session abandonment | Analytics event pipeline |

---

## Architecture Diagrams

- `diagrams/telemetry-pipeline.mmd` - event flow from producers to storage and dashboards
- `diagrams/qoe-feedback-loop.mmd` - closed-loop optimization from player telemetry to CDN steering

---

## Telemetry Pipeline Architecture

All signals flow through a unified telemetry ingest layer backed by Apache Kafka. Streaming processing (Flink) handles real-time anomaly detection and alerting. Batch processing (Spark) handles historical analytics and SLO burn rate calculations.

```
Clients + Services
       |
  Telemetry Ingest (Kafka)
       |
  +----+----+
  |         |
Flink     Spark
(stream)  (batch)
  |         |
Metrics   Analytics
Store     Warehouse
  |
Dashboards + Alerts
```

---

## Failure Detection and Mitigation

| Scenario | Detection | Response |
|---------|-----------|----------|
| QoE degradation (TTFF, rebuffer) | Automated alert within 60 seconds | CDN traffic shift; bitrate ladder reduction; see `runbooks/qoe-degradation.md` |
| Service latency spike | p99 latency alert | Traffic shaping, horizontal scaling |
| Playback error rate increase | Error rate alert + error code grouping | Root cause by error code; CDN or service rollback |
| Telemetry pipeline failure | Completeness drop alert | Switch to fallback metrics; see `runbooks/telemetry-outage.md` |
| Alert delivery failure | Synthetic alert test miss | Escalate to observability on-call |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| Alert latency (p99) | <= 60 seconds | 7-day rolling |
| Telemetry completeness | >= 99.9% | 24-hour rolling |
| Dashboard data freshness | <= 30 seconds | Continuous |

Full SLO definitions are in `slos/slos.md`.

---

## Operational Artifacts

- `slos/slos.md`
- `runbooks/qoe-degradation.md`
- `runbooks/telemetry-outage.md`
