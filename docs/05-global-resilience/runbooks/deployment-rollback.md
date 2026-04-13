# Runbook: Deployment Rollback

**Severity:** SEV-2 (automatic rollback) / SEV-1 (if automatic rollback fails)
**On-Call Team:** Release Engineering, Service-Owning Team
**Escalation:** Engineering Lead, Deployment Platform Team

---

## Detection

A rollback is warranted when one or more of the following conditions are met:

- Automated canary analysis fails: error rate or latency exceeds baseline by configured threshold
- Guardrail breach detected: TTFF, Rebuffer Ratio, or Playback Error Rate exceeds SLO bounds
- Automated health checks fail for the new version during progressive rollout stage
- On-call engineer observes clear regression correlated with the deployment timestamp
- Customer support volume spikes following a release without another identified cause

---

## Pre-Rollback Assessment (< 3 minutes)

1. Confirm the deployment timeline: identify which service, which version, and when it was deployed.
2. Check the Canary Analysis Dashboard for the deployment. Note:
   - Which metrics triggered the failure
   - What percentage of traffic was on the new version at the time of detection
3. Confirm the previous stable version tag or image digest (available in the deployment record).
4. Assess whether the failure is in the deployed code, configuration, or a downstream dependency.

---

## Rollback Procedure

### Automatic Rollback (Preferred)

The deployment platform should initiate rollback automatically when canary failure thresholds are crossed. Verify this is occurring:

1. Check the deployment pipeline dashboard for rollback status.
2. Confirm traffic is being shifted back to the previous version.
3. Monitor rollback progress; confirm completion within 5 minutes.

### Manual Rollback (If Automation Fails)

1. Halt the active deployment pipeline to stop further rollout:
   ```
   deploy halt --service <service-name> --env production
   ```

2. Pin the service to the last known stable image digest:
   ```
   deploy rollback --service <service-name> --version <stable-tag> --env production
   ```

3. Monitor the rollback: confirm all pods are running the stable version:
   ```
   kubectl get pods -l app=<service-name> -o jsonpath='{.items[*].spec.containers[*].image}'
   ```

4. If the service uses feature flags, revert any flags enabled with the deployment:
   - Access the Feature Flag Console and revert to the pre-deployment snapshot.

---

## Validation

After rollback completes:

1. Confirm service health checks pass for the stable version.
2. Verify SLO metrics return to pre-regression baselines:
   - TTFF back below 1.5 seconds (p95)
   - Playback Error Rate below 0.1%
   - Service error rate within normal bounds
3. Validate dependent services have recovered from any cascading effects.
4. Sustain monitoring for 15 minutes before declaring the incident resolved.

---

## Communication

- **At rollback initiation:** Post to `#deployments` and `#incidents` with service name, version, and reason.
- **At completion:** Confirm in both channels with metric recovery status.
- **If SEV-1:** Page engineering lead and notify affected product team.

---

## Post-Incident

1. Preserve all deployment artifacts, logs, and canary analysis reports for the postmortem.
2. Identify the root cause: code defect, configuration error, infrastructure change, or dependency regression.
3. Add a test case or guardrail to prevent the same regression from reaching production.
4. Review whether automatic rollback triggered correctly; if not, file a platform defect.
5. Do not re-deploy the failed version without a documented fix and review approval.
6. Complete a postmortem within 5 business days if customer impact occurred.
