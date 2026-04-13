# Runbook: QoE Degradation

**Severity:** SEV-1 (if global) / SEV-2 (if regional or device-scoped)
**On-Call Team:** Playback SRE, QoE Engineering
**Escalation:** Playback Engineering Lead, CDN Partnerships

---

## Detection

QoE degradation is confirmed when any of the following alerts fire:

- TTFF p95 exceeds 1.5 seconds sustained for >= 3 minutes
- Rebuffer Ratio exceeds 0.3% of total watch time for >= 5 minutes
- Playback Error Rate exceeds 0.1% of sessions for >= 2 minutes
- Customer support complaint rate increases > 50% above baseline within a 10-minute window
- Automated anomaly detection flags a deviation of >= 2 standard deviations on any key QoE signal

---

## Initial Assessment (< 5 minutes)

1. Open the **QoE Health Dashboard** and determine the scope:
   - Global, regional, or CDN-specific?
   - Specific device type or platform? (TV, mobile, web, partner devices)
   - Specific content type? (live events, new releases, back-catalog)
   - Specific bitrate tier? (4K, HD, SD)

2. Check the **Telemetry Pipeline Status** dashboard to confirm telemetry is flowing correctly. If telemetry is degraded, the QoE signals may be misleading (see `telemetry-outage.md`).

3. Correlate with recent deployments: check the deployment log for changes in the past 2 hours.

4. Correlate with CDN health: check CDN provider dashboards for any active provider incidents.

---

## Mitigation Steps

### CDN-Related Degradation

1. Review CDN Steering weights and shift traffic away from the impaired provider.
2. Reduce the bitrate ladder cap to 1080p to lower CDN throughput pressure if segments are slow to deliver.
3. If a specific CDN PoP (point of presence) is the source, disable it and allow the provider to route to the next closest PoP.

### Origin or Service Degradation

4. Scale the Playback Session Service if latency is elevated on the origin-side path.
5. Check the Manifest Service error rate; increase replica count if errors are elevated.
6. Verify the DRM License Service is healthy; a slowdown there will increase playback startup time.

### Device or Client-Side Degradation

7. If degradation is scoped to a specific client version, coordinate with the client team to force-push a hotfix or roll back via remote configuration.
8. Issue a remote bitrate cap or ABR policy change via the Client Configuration Service.

---

## Investigation

While mitigation is in progress:

1. Pull distributed traces for failed playback sessions from the Trace Explorer.
2. Identify the first point of failure in the request lifecycle (client, auth, manifest, CDN, origin).
3. Check Playback Error codes in the analytics store for patterns (e.g., license timeout, manifest parse failure).
4. Review server-side logs for the Playback Session Service and Manifest Service.
5. If rebuffering is the primary signal, inspect CDN bandwidth utilization; compare against historical patterns for the same time of day.

---

## Recovery Validation

1. Confirm TTFF p95 returns to <= 1.5 seconds.
2. Confirm Rebuffer Ratio returns to <= 0.3%.
3. Confirm Playback Error Rate returns to <= 0.1%.
4. Sustain monitoring for 30 minutes after metrics normalize.
5. Confirm customer complaint rate returns to baseline.

---

## Communication

- **At detection:** Post scope and initial assessment to `#incidents-sev2` (or `#incidents-sev1` if global).
- **Every 15 minutes:** Post updated metrics and active mitigation steps.
- **At resolution:** Post resolution summary with root cause, timeline, and user impact estimate.

---

## Post-Incident

1. File a detailed incident report within 24 hours.
2. Identify whether detection was fast enough; if not, propose new alerting rules.
3. Document any manual mitigation steps that could be automated.
4. Review whether the CDN Steering Service should have reacted sooner.
5. Schedule a postmortem within 5 business days if SLO was breached.
