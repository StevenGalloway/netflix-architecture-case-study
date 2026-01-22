# Experiment Guardrails

## Core Guardrails

| Metric | Threshold |
|------|-----------|
TTFF | +5% over baseline |
Rebuffer Ratio | +0.2% over baseline |
Playback Error Rate | +0.1% |
Session Abandonment | +1% |
Crash Rate | +0.5% |

## Enforcement
If any guardrail is violated for more than 5 minutes, the experiment is automatically rolled back.
