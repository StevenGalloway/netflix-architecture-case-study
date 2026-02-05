# Observability & QoE SLOs

## Key SLIs

| Metric | Description |
|------|-------------|
TTFF | Time to first frame |
Rebuffer Ratio | % of time buffering |
Playback Error Rate | Failed sessions |
Alert Latency | Time from failure to alert |

## SLO Targets

| SLI | Target |
|-----|--------|
TTFF | ≤ 1.5 seconds |
Rebuffer Ratio | ≤ 0.3% |
Playback Error Rate | ≤ 0.1% |
Alert Latency | ≤ 60 seconds |

## Error Budget Policy
If SLOs breach, feature launches pause until stability is restored.
