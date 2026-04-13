# Observability & QoE Service Level Objectives

## SLI Definitions

| Signal | SLI | Measurement Method |
|--------|-----|-------------------|
| Time to First Frame | Duration from play intent to first decoded frame | Client-side event telemetry |
| Rebuffer Ratio | Percentage of total watch time spent buffering | Client-side session telemetry |
| Playback Error Rate | Percentage of play attempts that fail to start or abort | Client-side + server-side event correlation |
| Alert Latency | Time from anomaly onset to alert delivery | Telemetry pipeline + alerting system timestamps |
| Telemetry Completeness | Percentage of expected events received within 60 seconds | Pipeline monitoring |
| Trace Coverage | Percentage of requests with end-to-end distributed traces | Sampling configuration audit |

---

## SLO Targets

| SLI | Target | Measurement Window | Error Budget (30-day) |
|-----|--------|-------------------|----------------------|
| TTFF (p95) | <= 1.5 seconds | 30-day rolling | 0.1% of play sessions |
| TTFF (p50) | <= 0.8 seconds | 30-day rolling | 0.1% of play sessions |
| Rebuffer Ratio | <= 0.3% of watch time | 30-day rolling | 0.02% excess buffering |
| Playback Error Rate | <= 0.1% of play attempts | 30-day rolling | 0.1% of sessions |
| Alert Latency (p99) | <= 60 seconds | 7-day rolling | < 1% of alerts delayed |
| Telemetry Completeness | >= 99.9% | 24-hour rolling | 0.1% event loss |

---

## Error Budget Policy

| Error Budget Remaining | Action |
|-----------------------|--------|
| > 50% | Normal operations; feature launches permitted |
| 25% - 50% | Increased monitoring cadence; new launches require SRE review |
| 10% - 25% | All non-critical launches paused; SRE on active watch |
| < 10% | Freeze all feature launches; incident declared; leadership notified |
| 0% (exhausted) | Full incident response; mandatory postmortem; 2-week stability window |

---

## Alerting Thresholds

| Metric | Warning | Critical | Response |
|--------|---------|----------|----------|
| TTFF p95 | > 1.2 s | > 1.5 s | Page on-call SRE |
| Rebuffer Ratio | > 0.2% | > 0.3% | Page on-call SRE + CDN team |
| Playback Error Rate | > 0.07% | > 0.1% | Page on-call SRE |
| Alert Latency | > 45 s | > 60 s | Page observability team |
| Telemetry Completeness | < 99.95% | < 99.9% | Page data platform team |

---

## SLO Review Cadence

| Review Type | Frequency | Participants |
|-------------|-----------|-------------|
| SLO Health Review | Weekly | SRE, Product, Engineering leads |
| Error Budget Review | Monthly | Engineering, Product, Leadership |
| SLO Target Review | Quarterly | Architects, SRE, Product |
| Postmortem Review | After each SLO breach | All stakeholders |

---

## Exclusions

The following conditions are excluded from SLO calculations:

- Planned maintenance windows with >= 72 hours advance notice
- Force majeure events (BGP-level internet outages, major cloud provider failures)
- Client-side failures caused by user device constraints or network conditions below minimum thresholds
- Events during declared infrastructure chaos exercises with active flags
