---
name: reading-readiness
version: 1.1.1
description: Use when reading or explaining an Argus readiness score, its HRV, resting-heart-rate, sleep, and load contributors, missing inputs, personal baselines, confidence, or readiness trend.
license: MIT
---

# Read readiness with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Readiness is an auditable summary against the user's own 28-day baselines. It is not a diagnosis and does not compare the user with a population.

```sh
argus readiness
argus readiness --date 2026-08-27
argus readiness --trend 30
```

MCP keeps the daily and series questions separate: tool `get_readiness` reads one day, while tool `get_readiness_trend` returns exactly 7, 30, or 90 dated points. Preserve null trend points as unevaluated rather than filling them with zero.

## Explain the result

- Quote `score` only when it is non-null. Argus requires at least two known contributors.
- Name `confidence`, `missing`, and each contributor's `value`, `baseline_mean`, `delta_pct`, `score`, and `weight`.
- `baseline_days` shows how much personal history supports that contributor. Readiness generally needs seven nights plus heart data before it becomes useful.
- A null score means **unknown**, never zero, poor, or not ready.
- Treat the trend as descriptive. Do not turn readiness into medical advice or a training order.

## Worked transcript

```text
$ argus readiness --output json
{"date":"2026-08-27","has_data":true,"score":74,"confidence":"reduced","missing":["load"],"contributors":[{"key":"hrv","value":58,"unit":"ms","baseline_mean":51,"baseline_days":24,"delta_pct":13.7,"score":82,"weight":0.3,"has_data":true},{"key":"resting_hr","value":56,"unit":"count/min","baseline_mean":54,"baseline_days":25,"delta_pct":3.7,"score":68,"weight":0.25,"has_data":true}],"delta_from_yesterday":3}

The readiness score is 74 with reduced confidence. HRV is 58 ms versus a 51 ms personal baseline over 24 days; resting heart rate is 56 versus 54 over 25 days. Load is missing, so the score is descriptive rather than a complete judgment.
```

If `score` is null, answer: “Readiness is unevaluated. Argus has fewer than two known contributors; collect seven nights plus heart data.”

Skills version 1.1.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
