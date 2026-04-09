# Runbook: CDN Degradation

**Severity:** SEV-1 (if global) / SEV-2 (if regional)
**On-Call Team:** Playback SRE, CDN Partnerships
**Escalation:** Playback Engineering Lead, VP of Infrastructure

---

## Detection

This incident is typically triggered by one or more of the following signals:

- Automated alert: TTFF p95 exceeds 1.5 seconds for 3 consecutive minutes
- Automated alert: Playback Error Rate exceeds 0.1% for 2 consecutive minutes
- Automated alert: CDN provider reports degraded status via health check API
- Customer support spike: elevated complaint volume correlating with region or device type
- CDN provider incident notification via status page or direct channel

---

## Initial Assessment (< 5 minutes)

1. Open the **Playback Health Dashboard** and confirm the scope:
   - Is degradation global or region-specific?
   - Which CDN provider(s) are affected? (CDN-A, CDN-B, CDN-C)
   - Which content segments are impacted? (manifest, video segments, audio-only)

2. Check CDN provider status pages and internal CDN health APIs.

3. Confirm whether the CDN Steering Service has already initiated automatic failover:
   - Look for steering weight changes in the CDN control plane audit log.
   - If automatic failover is in progress, monitor for stabilization before manual intervention.

---

## Immediate Mitigation

**If automatic steering has NOT triggered:**

1. Manually adjust CDN traffic weights in the CDN Steering Console:
   - Reduce weight on degraded provider to 0% or minimum threshold.
   - Redistribute traffic to healthy providers proportionally.

2. Verify the steering weight update has propagated (TTL: 30 seconds).

3. Increase origin shield capacity if origin request volume rises due to cache misses:
   ```
   # Scale origin shield fleet in the affected region
   kubectl scale deployment origin-shield --replicas=<target> -n playback
   ```

4. If all CDN providers show degradation in a region, escalate to regional evacuation (see `05-global-resilience/runbooks/region-evacuation.md`).

**Bitrate ladder reduction (if segments are slow but not failing):**

5. Issue a bitrate cap directive via the Quality Control Service to reduce peak resolution from 4K to 1080p or 720p. This reduces CDN throughput pressure during degraded conditions.

---

## Validation

After mitigation steps:

1. Confirm TTFF returns below 1.5 seconds p95 on the Playback Health Dashboard.
2. Confirm Playback Error Rate returns below 0.1%.
3. Confirm Rebuffer Ratio returns below 0.3%.
4. Verify CDN health check scores normalize across all affected edge nodes.
5. Sustain monitoring for 15 minutes to confirm stability before declaring resolution.

---

## Communication

- **At detection:** Post initial assessment to `#incidents-sev1` with scope and impact estimate.
- **Every 15 minutes:** Post status update to `#incidents-sev1`.
- **At resolution:** Post resolution summary including: root cause, mitigation taken, user impact duration, and ticket link.
- **If SEV-1:** Notify on-call Playback Engineering Lead and open customer-facing status page incident.

---

## Post-Incident

1. Document incident timeline in the incident tracker.
2. Identify whether CDN Steering should have triggered automatically and, if not, why.
3. Review whether CDN provider SLA was breached and initiate vendor escalation if applicable.
4. Schedule a postmortem within 5 business days.
5. Update this runbook with any learnings or gaps discovered during response.
