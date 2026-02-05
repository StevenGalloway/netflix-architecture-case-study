# Runbook: Telemetry Pipeline Outage

## Detection
- Missing telemetry data
- Alerting gaps

## Mitigation
1. Switch to fallback metrics sources
2. Increase sampling on core services
3. Notify SRE & data teams

## Recovery
- Restore pipeline
- Backfill data if possible
- Update detection safeguards
