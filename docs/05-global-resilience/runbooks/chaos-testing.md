# Chaos Engineering Program

**Owner:** Resilience Engineering Team
**Cadence:** Quarterly global events, monthly service-level tests
**Coordination Required:** SRE, Playback Engineering, CDN Partnerships, Data Platform

---

## Objectives

- Validate fault tolerance assumptions before production failures expose them
- Improve incident response readiness through deliberate failure practice
- Discover unknown failure modes in distributed system interactions
- Measure and enforce recovery time against RTO/RPO targets
- Build organizational confidence in automated recovery mechanisms

---

## Test Categories

### Tier 1: Service-Level Tests (Monthly)

Scoped to a single service or dependency. Run during off-peak hours with limited blast radius.

| Test | Description | Success Criteria |
|------|-------------|-----------------|
| Dependency blackhole | Drop all traffic from service A to service B | Circuit breaker triggers within 5 seconds |
| Memory pressure | Inject OOM conditions into a replica set | Auto-restart completes within 60 seconds |
| CPU saturation | Pin CPU to 95% on a target deployment | Horizontal auto-scaler triggers within 90 seconds |
| Disk I/O degradation | Throttle storage throughput | Service degrades gracefully without data loss |
| Pod eviction | Force-evict 30% of pods in a deployment | Traffic redistributes within 10 seconds |

### Tier 2: Regional Tests (Quarterly)

Scoped to a single cloud region. Run during declared maintenance windows with customer impact disclosure.

| Test | Description | Success Criteria |
|------|-------------|-----------------|
| Regional DNS blackout | Simulate DNS resolution failure in a region | GSLB reroutes traffic within 2 minutes |
| CDN brownout | Reduce CDN capacity by 50% in target region | Steering adjusts; TTFF stays below 2 seconds |
| Network latency injection | Add 200ms to all inter-service calls | No cascading failures; graceful degradation active |
| AZ outage simulation | Shut down all services in one availability zone | Regional traffic shifts within 5 minutes |

### Tier 3: Global Chaos Events (Quarterly)

Full-scale exercises with cross-region failure scenarios. All major teams on active standby.

| Test | Description | Success Criteria |
|------|-------------|-----------------|
| Region evacuation drill | Simulate complete regional failure and evacuate | RTO <= 5 minutes; no data loss |
| Multi-CDN failure | Take 2 of 3 CDN providers to zero | Playback error rate stays below 1% |
| Data corruption simulation | Inject invalid data into pipeline | Corruption detected and blocked before publish |
| Control plane outage | Shut down personalization + experiment systems | Playback continues from cached configuration |

---

## Execution Protocol

### Pre-Test Checklist

- [ ] Chaos test ticket approved by SRE lead and affected team leads
- [ ] Maintenance window communicated with >= 48 hours notice
- [ ] On-call rosters confirmed and all escalation paths active
- [ ] Rollback mechanism tested and documented in the test ticket
- [ ] Customer-facing status page updated if any customer impact is anticipated
- [ ] Blast radius estimate documented and accepted by risk owner

### During Test

1. Assign a dedicated **Test Commander** to coordinate communication.
2. Assign a **Safety Officer** with authority to abort the test at any time.
3. All participants on a dedicated incident bridge with Slack channel.
4. Record all observations in real time in the test ticket.
5. Abort immediately if any unintended system enters an impaired state outside test scope.

### Post-Test

1. Restore all injected failures and confirm full system recovery.
2. Validate SLOs returned to within normal bounds on all dashboards.
3. Document test results: what passed, what failed, any surprises.
4. File action items for gaps discovered; assign owners and due dates.
5. Present findings in next resilience review.

---

## Schedule (Rolling 12-Month)

| Quarter | Tier 3 Focus | Tier 2 Focus |
|---------|-------------|-------------|
| Q1 | Region evacuation drill | CDN brownout simulation |
| Q2 | Multi-CDN provider failure | AZ outage simulation |
| Q3 | Control plane outage | Regional DNS blackout |
| Q4 | Data corruption injection | Network latency injection |

---

## Abort Criteria

Halt the test immediately if any of the following occur:

- Unintended impact to non-participating regions or services
- SLO breach exceeding 3x the expected impact window
- On-call team unable to reach the chaos test commander
- Rollback mechanism fails to execute
- Any sign of data loss in production systems
