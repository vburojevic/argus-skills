---
name: acting-on-attention-lines
version: 1.1.0
description: Use when reading, explaining, inspecting, or following up an Argus attention line from the anomaly sentinel, including its metric, baseline, duration, evidence, and clearing behavior.
license: MIT
---

# Act on an attention line with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

An attention line is a restrained observation from the anomaly sentinel. It is not an alert badge, diagnosis, prediction, or medical advice.

```sh
argus attention
argus attention --date 2026-08-27
argus metrics get resting_heart_rate --start 2026-08-20T00:00:00Z --paginate
```

## Workflow

1. Quote the server-rendered `sentence`; it already states the evidence plainly.
2. Name `metric`, current `value`, `baseline_mean`, `unit`, and `days`.
3. Inspect that metric's raw samples or appropriate aggregates before adding context.
4. Never prescribe treatment or present the line as a medical conclusion. If symptoms or concern exist, suggest appropriate professional care without using Argus as the diagnosis.
5. The line clears when the underlying monitor no longer satisfies its open-episode rule. There is no manual dismissal command.

`body_signals` is a composite observation that opens when at least two
illness-direction signals are concurrently outside their personal baselines.
Its sentence names only the signals currently contributing. It subsumes their
single-metric lines while open and clears after fewer than two signals remain
for two consecutive days. It is not a medical alert, and Argus gives no
medical advice.

An empty array means no line is open. It is not proof that everything is normal: each monitor needs at least seven baseline days before it can open a line.

## Worked transcript

```text
$ argus attention --output json
[{"monitor":"resting_heart_rate","metric":"resting_heart_rate","opened_at":"2026-08-25T06:00:00Z","days":3,"value":67,"baseline_mean":56,"unit":"count/min","sentence":"Resting heart rate has run above your baseline for 3 days: 67 versus 56 count/min."}]

Argus has one observational attention line: resting heart rate has been 67 count/min versus a 56 baseline for three days. This is not a diagnosis. I can inspect the underlying resting-heart-rate samples to check timing and provenance.
```

Skills version 1.1.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
