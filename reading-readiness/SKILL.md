---
name: reading-readiness
version: 1.5.0
description: Use when reading or explaining an Argus readiness score, its HRV, resting-heart-rate, sleep, and load contributors, missing inputs, personal baselines, confidence, readiness trend, or Today's What changed tiles against your usual, yesterday or last week.
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
- Name `confidence`, `missing` (with each input's `missing_reason`), and each contributor's `value`, `baseline_mean`, `delta_pct`, `score`, and `weight`.
- `baseline_days` shows how much personal history supports that contributor: days with a reading among the 28 before the scored day. Readiness generally needs seven nights plus heart data before it becomes useful.
- A contributor with `has_data: false` is **not counted**, and its `missing_reason` says why. `no_reading`: the day has no reading (`value` is null). `no_baseline`: the day has a reading — `value` carries it — but fewer than 7 of the 28 days before it do, so there is nothing yet to compare it with. Never call a `no_baseline` input missing or "no data"; say it is not counted yet and quote its value. The surfaces word it as "Sleep not counted: under 7 nights of history to compare" or "Sleep not counted: no reading last night" (HRV, resting HR and load count days, not nights).
- `missing_reason` is omitted on a counted contributor, never null.
- A null score means **unknown**, never zero, poor, or not ready.
- Treat the trend as descriptive. Do not turn readiness into medical advice or a training order.
- The apps' Readiness card reads the score against its own record from the trend, and so should you: its **usual** is the mean of the scores on the 28 days before the day, needing 7 ("8 above your usual of 74", the printed score minus the usual rounded to a whole point); its **usual band** is the median of those days ± 1.4826 × the median absolute deviation, never narrower than ±3 points, needing 14 ("usual 68–80"); its dots are the scores of the 13 days before. Null points are left out of all three, never zero. Read the 30-day trend for today or yesterday and the 90-day one further back; a day whose 28 days no trend reaches has no usual, so say so rather than estimate one.

## What changed

`argus today` (tool `get_today`) carries `changes.tiles` when the night's measures have readings: HRV, resting heart rate, sleep, respiratory rate, in that fixed order, with wrist temperature, then blood oxygen, taking only a place one of those four leaves empty. Each tile has the day's `value` (null = no reading that day), its canonical `unit`, a 7-day `spark` ending on the day (null = a gap, never zero), and up to three bases:

- `usual` — the mean of the 28 days before, needing 7 with a value. It is readiness's own `baseline_mean` for HRV and resting heart rate, and the attention line's "your usual" for respiratory rate: Argus prints one usual per measure. Sleep counts nights of at least three hours asleep, so a nap never enters the value or the usual.
- `yesterday` — the day before.
- `last_week` — the mean of the 7 days before, needing 4.

A basis is absent when there is too little behind it; say so ("no usual yet", "none yesterday", "too few last week"), never compute against it. Values are canonical and unrounded: round the value and the basis to the figure as printed (whole ms, bpm and minutes; tenths for breathing), then subtract, so 68 ms against a usual printed 59 is "9 ms above". A tile carries `display_unit` only when the person reads that measure in another unit than the canonical `unit` — today only `degF` for wrist temperature: convert the value and the basis to it first, then round and subtract, so 35.4 °C against 35.1 °C reads "95.7 °F, 0.5 °F above", never a °C difference converted. `attention_days` means an open attention episode on the metric ("Breathing · 2 nights"). A tile states which way the number moved, never better or worse: leave judgement to the Brief's words.

## Worked transcript

```text
$ argus readiness --output json
{"date":"2026-08-27","has_data":true,"score":74,"confidence":"reduced","missing":["sleep","load"],"contributors":[{"key":"hrv","value":58,"unit":"ms","baseline_mean":51,"baseline_days":24,"delta_pct":13.7,"score":82,"weight":0.3,"has_data":true},{"key":"resting_hr","value":56,"unit":"bpm","baseline_mean":54,"baseline_days":25,"delta_pct":3.7,"score":68,"weight":0.25,"has_data":true},{"key":"sleep","value":7.67,"unit":"h","baseline_mean":null,"baseline_days":4,"delta_pct":null,"score":null,"weight":0.3,"has_data":false,"missing_reason":"no_baseline"},{"key":"load","value":null,"unit":"kcal","baseline_mean":null,"baseline_days":26,"delta_pct":null,"score":null,"weight":0.1,"has_data":false,"missing_reason":"no_reading"}],"delta_from_yesterday":3}

The readiness score is 74 with reduced confidence. HRV is 58 ms versus a 51 ms personal baseline over 24 days; resting heart rate is 56 versus 54 over 25 days. Sleep is not counted yet: you slept 7:40, but there are only 4 nights of history to compare it with, and readiness needs 7. Load is not counted: no activity reading yesterday. The score is descriptive rather than a complete judgment.
```

If `score` is null, answer: “Readiness is unevaluated. Argus has fewer than two known contributors; collect seven nights plus heart data.”

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
