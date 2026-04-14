# Runbook: Kafka Consumer Lag

**Severity:** SEV-2 (streaming features degraded) / SEV-1 (if alerting pipeline is affected)
**On-Call Team:** Data Platform SRE
**Escalation:** Data Platform Engineering Lead

---

## Overview

Consumer lag is the difference between the latest offset produced to a Kafka topic and the offset most recently committed by a consumer group. Sustained lag means downstream consumers are reading stale data, which can result in stale feature values in real-time recommendations, delayed anomaly detection alerts, and delayed operational dashboards.

**Acceptable lag thresholds:**
- Real-time feature consumers (Flink): <= 10,000 messages (~5 seconds)
- Alerting consumers (Flink): <= 5,000 messages (~2-3 seconds)
- Batch ingestion consumers (Spark): <= 10 million messages (~15 minutes)

---

## Detection

- Automated alert: consumer group lag exceeds threshold for 3 consecutive minutes
- Flink job backpressure indicator turns red in the Flink job manager UI
- Downstream alert: online feature store freshness SLA breach
- Manual observation during dashboard review

---

## Initial Assessment (< 5 minutes)

1. Open the **Kafka Consumer Lag Dashboard** and identify:
   - Which consumer group is lagging?
   - Which topic(s) and partition(s)?
   - Is lag growing, stable, or recovering?

2. Check whether the consumer is still running:
   ```
   kubectl get pods -l app=<consumer-name> -n data-platform
   ```

3. Check producer throughput: has the produce rate increased significantly (e.g., a traffic spike or a new producer)?

4. Check consumer throughput: is the consumer processing at its normal rate, or has processing slowed?

---

## Mitigation Steps

### Consumer Has Fallen Behind Due to Traffic Spike

1. Scale up the consumer group by increasing the number of consumer instances:
   ```
   kubectl scale deployment <consumer-name> --replicas=<target> -n data-platform
   ```
   Note: the number of consumer instances cannot exceed the number of topic partitions.

2. If the topic has insufficient partitions to support more consumers, increase the partition count:
   - Coordinate with the topic owner before changing partition counts on existing topics.
   - Adding partitions does not rebalance existing data; it only affects future messages.

### Consumer Is Processing Slowly (Backpressure)

3. Check whether the downstream sink (online feature store, Iceberg) is responding slowly:
   - Review feature store latency on the feature store dashboard.
   - Review Iceberg commit latency if Flink is writing to Iceberg.

4. If the downstream sink is the bottleneck, address the sink issue first (see relevant runbook).

5. If Flink is the consumer, check for a resource bottleneck (CPU, memory, network):
   ```
   kubectl top pods -l app=<flink-job-name> -n data-platform
   ```
   Increase task manager memory or CPU limits if resources are saturated.

### Consumer Process Has Failed

6. Check consumer pod logs for errors:
   ```
   kubectl logs -l app=<consumer-name> -n data-platform --tail=200
   ```

7. If the consumer is crashing due to a schema mismatch or deserialization error, identify the offending messages and either:
   - Fix the consumer schema and redeploy
   - Skip the offending offset range (only as a last resort; document the decision)

---

## Validation

1. Confirm consumer lag is decreasing consistently on the Kafka Consumer Lag Dashboard.
2. Confirm Flink job backpressure indicators return to green in the Flink UI.
3. Confirm online feature store freshness returns to within SLA.
4. Confirm alerting pipeline lag returns to acceptable thresholds.

---

## Post-Incident

1. Identify the root cause: traffic spike, slow consumer, slow sink, or process failure.
2. Review whether partition count is sufficient for the current peak throughput.
3. Evaluate whether consumer auto-scaling can prevent manual intervention in future.
4. Document any offset skips with justification in the data quality log.
