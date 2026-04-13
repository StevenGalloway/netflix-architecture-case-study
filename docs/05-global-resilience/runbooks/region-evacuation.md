# Runbook: Regional Evacuation

**Severity:** SEV-1
**On-Call Team:** SRE, Playback Engineering, CDN Partnerships, Data Platform
**Escalation:** VP of Infrastructure, CTO staff

---

## Overview

Regional evacuation is the controlled migration of all production traffic out of an impaired cloud region into healthy regions. This procedure is executed when a region is confirmed degraded beyond the ability to recover within acceptable SLO bounds.

**RTO target:** <= 5 minutes from decision to full traffic evacuation
**RPO target:** 0 for all critical data systems (replication must be confirmed active)

---

## Detection

Evacuation is warranted when two or more of the following are true:

- Regional health check failure rate exceeds 20% for 3 consecutive minutes
- Playback Error Rate for region-sourced sessions exceeds 1%
- Cloud provider has issued an active incident affecting core services (compute, networking, or storage)
- Cross-region replication lag exceeds 5 minutes on critical data stores
- SRE lead makes a judgment call based on rate of degradation and trend

---

## Pre-Evacuation Checklist (< 2 minutes)

- [ ] Confirm the impaired region and identify all services running there
- [ ] Verify receiving regions have sufficient capacity headroom (>= 40% headroom on compute)
- [ ] Confirm cross-region data replication is active and lag is within acceptable bounds
- [ ] Alert all relevant on-call leads via PagerDuty and the `#incidents-sev1` Slack channel
- [ ] Open a customer-facing status page incident if user impact is already confirmed

---

## Evacuation Steps

### Step 1: Freeze All Deployments

Halt all active deployment pipelines to prevent new changes from complicating recovery:

```
deploy freeze --env production --reason "Regional evacuation in progress"
```

### Step 2: Shift Traffic at the Global Load Balancer

Update the GSLB weights to route 0% of traffic to the impaired region:

```
gslb update-weights --region <impaired-region> --weight 0
gslb update-weights --region <healthy-region-1> --weight 50
gslb update-weights --region <healthy-region-2> --weight 50
```

Verify propagation: GSLB weight changes take effect within 30-60 seconds. Monitor the Traffic Distribution Dashboard for confirmation.

### Step 3: Scale Receiving Regions

Pre-emptively scale receiving regions to absorb the additional traffic load:

```
kubectl scale deployment playback-api --replicas=<target> -n production --context <healthy-region-1>
kubectl scale deployment playback-api --replicas=<target> -n production --context <healthy-region-2>
```

Allow 60-90 seconds for new pods to become ready and pass health checks.

### Step 4: Validate CDN Routing

Confirm CDN origin routing has updated to point away from the evacuated region:

- Check CDN provider origin configuration for the impaired region.
- If CDN has not automatically updated, manually update origin endpoints in the CDN control plane.

### Step 5: Confirm Playback Recovery

- Monitor Playback Health Dashboard for TTFF, Rebuffer Ratio, and Error Rate normalization.
- Target: metrics returning to SLO bounds within 5 minutes of evacuation start.

---

## Data Consistency Verification

After evacuation:

1. Confirm all active session tokens are still being honored by healthy regions.
2. Verify cross-region replication lag has not grown beyond acceptable bounds on:
   - Entitlements store
   - DRM license cache
   - User session store
3. If replication lag is elevated, apply read-from-local policies in receiving regions until lag clears.

---

## Communication

- **At T+0 (decision):** Post to `#incidents-sev1`: region, scope, and evacuation start.
- **At T+2 (traffic shifted):** Confirm traffic shift complete; provide early metrics.
- **At T+5 (stable or not):** Provide full status: metrics, capacity, and next steps.
- **Every 10 minutes thereafter:** Ongoing status until resolution.
- **At resolution:** Full summary with RTO achieved, RPO status, and impact estimate.

---

## Return-to-Region Protocol

Do not return traffic to the evacuated region until all of the following are confirmed:

- Cloud provider has declared the incident resolved
- All services in the region pass health checks for >= 15 continuous minutes
- Cross-region replication has fully caught up
- SRE lead approves return-to-traffic

Return traffic gradually: 5% -> 20% -> 50% -> 100%, with 5-minute observation windows between each step.

---

## Post-Incident

1. Measure achieved RTO and RPO against targets; document any gap.
2. Identify whether evacuation could have been initiated sooner.
3. Review capacity in receiving regions; adjust auto-scaling thresholds if needed.
4. Update GSLB configuration if default weights contributed to slower evacuation.
5. Schedule a postmortem within 3 business days given SEV-1 customer impact.
