# Runbook: Telemetry Pipeline Outage

**Severity:** SEV-2
**On-Call Team:** Data Platform SRE, Observability Engineering
**Escalation:** Data Platform Engineering Lead, SRE Lead

---

## Overview

A telemetry pipeline outage causes loss of operational visibility across the platform. While playback itself may remain unaffected, the inability to observe the system creates risk of undetected degradation. This runbook should be executed promptly to restore monitoring fidelity.

---

## Detection

This incident is typically triggered by:

- Automated alert: telemetry event completeness drops below 99.9% for 5 consecutive minutes
- Alerting gap detected: monitoring systems stop firing alerts that are expected to be active
- Dashboard flatlines: QoE, latency, and error rate panels show no data or stale data
- Data team reports Kafka consumer lag exceeding thresholds on telemetry topics
- Flink streaming jobs report checkpoint failures or backpressure

---

## Initial Assessment (< 5 minutes)

1. Open the **Telemetry Pipeline Health Dashboard** and identify where in the pipeline the failure is occurring:
   - **Ingest layer:** Are events arriving at the Kafka brokers? Check producer metrics.
   - **Streaming processing:** Are Flink jobs running and processing events? Check job manager UI.
   - **Storage layer:** Are events landing in the metrics, log, and trace stores?
   - **Query layer:** Are dashboards able to query the stores, or is it a presentation issue?

2. Check Kafka broker health: confirm broker count, topic health, and replication factor status.

3. Check Flink job manager and task manager status for the telemetry processing jobs.

4. Determine scope: Is the outage affecting all telemetry, or just a specific signal type (metrics, logs, traces, or QoE events)?

---

## Mitigation Steps

### Kafka Ingestion Failure

1. If brokers are unavailable, check Kafka cluster health and restart failed brokers:
   ```
   kubectl rollout restart statefulset/kafka -n telemetry
   ```
2. If a specific topic is misconfigured or has a retention issue, inspect and correct topic configuration.
3. Verify that telemetry producers (client agents, service sidecars) are still sending events; check producer error rates.

### Flink Job Failure

4. Check the Flink job manager for failed jobs and their error logs.
5. Restart the failed Flink job from the latest successful checkpoint:
   ```
   flink cancel <job-id>
   flink run -s <checkpoint-path> <job-jar>
   ```
6. If checkpoint recovery fails, restart from the beginning of the retention window and accept data loss for the gap period.

### Storage Layer Failure

7. If the metrics store (e.g., Prometheus, Atlas) is degraded, switch dashboards to the secondary metrics store.
8. Enable fallback sampling mode: reduce telemetry event volume by 50% and send only critical QoE and error signals to the secondary pipeline.

### Dashboard Query Failure

9. If the underlying data is available but dashboards are broken, restart the query service:
   ```
   kubectl rollout restart deployment/grafana -n observability
   ```
10. Activate pre-configured backup dashboards that query the secondary store.

---

## Operating in Degraded Visibility

While the telemetry pipeline is being restored, apply the following measures to maintain situational awareness:

1. **Increase synthetic monitoring cadence:** Boost synthetic playback probes from 1-minute to 30-second intervals across all regions and device types.
2. **Monitor CDN provider dashboards directly:** Check CDN error rates and hit/miss ratios in provider consoles.
3. **Activate server-side log sampling:** Enable elevated log verbosity on the Playback Session Service, Manifest Service, and DRM Service to capture signals without the streaming pipeline.
4. **Brief the on-call team:** Inform all active on-call engineers that telemetry is impaired and manual checks are required.

---

## Data Recovery

After the pipeline is restored:

1. Assess the data gap: determine the start and end timestamps of the outage.
2. If Kafka retained the events during the outage (within the configured retention period), replay the events through the pipeline to backfill historical data.
3. If events were lost, document the gap with exact timestamps and notify the data governance team.
4. Validate backfilled data against expected baselines before re-enabling automated alerting rules.

---

## Recovery Validation

1. Confirm telemetry completeness returns to >= 99.9% on the pipeline health dashboard.
2. Confirm all dashboards are showing live, up-to-date data.
3. Confirm automated alerts are firing correctly by triggering a test alert.
4. Sustain monitoring for 30 minutes before declaring full resolution.

---

## Post-Incident

1. Document the data gap with precise timestamps; file a data quality incident report.
2. Identify whether the outage would have been detectable sooner with additional pipeline health monitoring.
3. Review Kafka retention configuration; confirm it provides sufficient buffer for pipeline recovery.
4. Evaluate whether the fallback metrics path was effective; improve if not.
5. Schedule a postmortem within 5 business days.
