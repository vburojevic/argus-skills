---
name: tracking-metrics
version: 1.5.5
description: Use when recording or querying body/health metrics and workouts in Argus — weight, steps, heart rate, sleep, water, or any of the 195 HealthKit metric types — including aggregates, derived figures, population reference, and unit conversion.
license: MIT
---

# Tracking metrics with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Every HealthKit metric type exists in Argus (195: quantity, category, characteristic). Names are snake_case HK identifiers: `body_mass`, `step_count`, `heart_rate`, `dietary_water`, `sleep_analysis`, `vo2_max`…

```sh
argus metrics list                              # the full registry with units + styles
```

## Recording

```sh
argus metrics record body_mass 82.5 --unit kg
argus metrics record body_mass 182 --unit lb    # converted server-side
argus metrics record dietary_water 0.5 --unit l # water is a metric, not a resource
argus metrics record step_count 8500 --start 2026-08-25T00:00:00+02:00 --end 2026-08-25T23:59:00+02:00
argus mood pleasant --label calm --label grateful --about self_care
```

- `--unit` accepts lb, kg, st, oz, ml, l, fl_oz, °F, kJ… — storage is always HK-canonical.
- Samples are intervals; a point sample omits `--end`.
- Syncing external data? Pass `--external-id <uuid>` — duplicates dedupe silently.
- State of Mind accepts a valence from `-1` to `1` or one of the seven words
  (`very_unpleasant` … `very_pleasant`). Labels and `--about` associations use
  Apple Health's snake_case names.

## Querying

```sh
argus metrics get body_mass                     # raw samples, newest first
argus metrics get body_mass --paginate          # every cursor page, one JSON array
argus metrics overview                          # every metric with latest + last-day figure
argus metrics get step_count --agg sum --bucket day    # aggregates (validated per metric style)
argus metrics get heart_rate --agg avg --bucket hour
argus metrics get step_count --representation timeline --start 2026-09-06T00:00:00+02:00 --end 2026-09-07T00:00:00+02:00 --paginate
argus metrics get sleep_analysis --sessions     # derived sleep sessions with per-stage minutes
argus trends --metrics body_mass --period 30d   # smoothed series + delta
argus trends --metrics body_mass --period 365d  # Year view
argus medications                               # per-medication Apple Health grants
argus medications sync --data '{"medications":[...]}'  # complete HealthKit snapshot
```

Medication sync clients use MCP tool `upsert_medications` with the complete user-shared snapshot. Do not infer a prescription or schedule, and do not interpret an empty result from tool `get_medications` as proof that the user takes none.

Aggregation must match the metric's style (summing heart rate is a 400 — the error's `remediation` explains). `sum` fits cumulative metrics (steps, dietary_*); `avg/min/max/latest` fit discrete ones (heart_rate, body_mass).

For intraday shape use `representation=timeline` on `GET /v1/metrics/{name}`, or `representation: "timeline"` with MCP `get_metrics`. Supply both bounds, at most seven days apart, and follow every `next_cursor` without changing them. Timeline reads include intervals crossing the start boundary, exclude merged HealthKit day totals, and apply raw source precedence. Clip intervals to the displayed profile-timezone day before charting. A merged step total cannot describe when steps happened; never distribute it across 24 hours or add it to raw observations. Timeline shapes are not authoritative daily totals: read those from summary/trends. The default raw representation remains a stored-row inventory. Timeline cannot be combined with `agg` or sleep sessions.

Apple-resolved historical step totals import newest first in resumable calendar-day pages. A confirmed raw HealthKit deletion invalidates the affected stored resolved total, including older days; summaries then use the remaining source-precedence rows until a fresh Apple total lands. An empty HealthKit query alone does not authorize deleting totals or reporting zero. Never add resolved totals to their raw samples.

## Workout records

```sh
argus workouts list --paginate
argus workouts get workout_abc
argus workouts log --data '{"activity":"running","start":"2026-08-30T07:00:00+02:00","end":"2026-08-30T07:30:00+02:00"}'
argus workouts edit workout_abc --data '{"activity":"walking"}'
argus workouts delete workout_abc --confirm
```

Inspect a workout before editing or deleting it. Do not duplicate a HealthKit-synced session or invent distance, energy, zones, or route detail. MCP tools: `get_workouts`, `get_workout`, `log_workout`, `edit_workout`, `delete_workout`.

## Derived figures

Sleep projections land at each wake moment: `sleep_duration`, and — when the night carries staged data — `sleep_deep`, `sleep_rem`, and `sleep_core`, each stage's minutes the union of its intervals. A night with only unstaged sleep writes no stage samples: absence, not zero. All four are quantity metrics and behave like any other in goals, trends, charts, and correlations.

Four more Argus-owned samples behave like every other metric in goals, trends, and detail reads:

- `sleep_debt` — accumulated shortfall across the last seven nights against the person's trailing 28-night sleep baseline.
- `sleep_consistency` — the 14-night population spread of sleep midpoints; lower is steadier.
- `social_jetlag` — the 14-night gap between median weekend and workday sleep midpoints.
- `training_load_ratio` — the 7-day mean of active energy divided by the 28-day mean.

These projections are deliberately absent until their minimum history is present. Never translate a missing derived sample into zero; say that the figure is not evaluated yet.

## Population reference

Reference context exists only for `vo2_max`, `resting_heart_rate`, `heart_rate_variability_sdnn`, `body_fat_percentage`, and `body_mass_index`:

```sh
$ argus metrics reference vo2_max
{"metric":"vo2_max","evaluated":true,"value":48,"unit":"mL/(kg·min)","percentile":85,"cohort":"men 35–44","sentence":"VO₂ max 48 — better than about 85 % of men 35–44.","source":"…"}
```

The API renders `sentence`; print it verbatim. For `evaluated: false`, report `no_birth_date`, `no_biological_sex`, or `no_reading` and do not infer the missing fact. Do not request a reference for metrics outside the closed list, and never present cohort context as diagnosis or advice.


## When the ledger looks stale

Argus stores what a device has already uploaded. Apple Health lands on the
server only when the person's iPhone syncs, so a reading can be older than your
question needs — and no amount of re-reading changes that on its own.

```sh
argus sync request                    # ask this account's devices to sync now
# {"nudged":1,"throttled":0}
```

Wait a few seconds, then re-read. `nudged` counts requests APNs accepted,
not devices that uploaded new data. `throttled` counts recently claimed
devices skipped by the floor. Zero may also mean push is unavailable or
APNs rejected the requests.

**A nudge is a request, never a guarantee.** A phone that is closed, offline,
in Low Power Mode or out of background budget may never answer. Report what
the ledger actually holds and how old it is — never state that data is current
because a nudge was sent.

## Gotchas

- `dietary_*` samples sourced from meals are **projections** — edit the meal, not the sample (`delete-sample` refuses them).
- Delete only an identified user-authored sample with `argus metrics delete-sample <sample_id> --confirm`; source projections must be undone at their meal, workout, supplement, or sleep record (`sleep_duration` and the stage metrics are projected from `sleep_analysis`).
- Daily numbers (`summary`, day buckets) follow the **profile timezone**; after the person confirms their timezone, record it with `argus profile set --timezone Europe/Zagreb --confirm-timezone`. A stored timezone alone does not prove confirmation; read `timezone_confirmed` from the profile, including for UTC.
- An empty medications response is normal: access is granted per medication
  inside the Health app, and denied items are deliberately invisible.
- `metrics overview` returns both `latest` (one sample) and `day` (the last profile-timezone day's style-aware figure). Do not substitute one for the other.

## Automatic Apple Health mirror

With Apple Health write permission, the connected iPhone/iPad mirrors Argus-authored meals, supplement nutrient contributions, body measurements, mood, water, sleep stage intervals and workouts. Writes accepted from web, Mac, Watch, CLI, MCP and chat reach the same path on the phone's next sync; they do not require a second agent write. Edits and deletes follow through a durable device cursor. The mirror is best effort while the phone is offline or permission is denied. Do not claim a Health write succeeded from an Argus receipt alone.

Measured HealthKit data never echoes through this path. HealthKit dietary imports remain excluded except water; mirrored water carries Argus sync identity and is skipped on import. Sleep duration/stage totals remain derived, never extra sleep sessions. A workout carries its recorded interval and energy; Apple Exercise Time is not directly writable and Activity ring credit remains Apple's decision. Medication dose events have no public HealthKit write API in the iOS 26.5 SDK and remain Argus-only.

Skills version 1.5.5 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
