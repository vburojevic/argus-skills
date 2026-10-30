---
name: tracking-metrics
version: 1.11.1
description: Use when recording or querying body/health metrics and workouts in Argus — weight, steps, heart rate, sleep, water, or any of the 198 HealthKit metric types — including aggregates, derived figures, population reference, and unit conversion.
license: MIT
---

# Tracking metrics with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Every HealthKit metric type exists in Argus (198, the iOS 27 catalog: quantity, category, characteristic). Names are snake_case HK identifiers: `body_mass`, `step_count`, `heart_rate`, `heart_rate_variability_rmssd`, `dietary_water`, `sleep_analysis`, `menopausal_state`, `vo2_max`… The six characteristics (birth date, sex, blood type, skin type, wheelchair use, Move mode) are profile fields: the iPhone fills them from Apple Health where the profile has none, and never overwrites a value the person typed in Argus.

Beside the registry, Apple Health also brings kinds that are not plain samples, each as an Argus metric with its own meta:

- **Apple's ring goals**: `active_energy_burned_goal` (or `apple_move_time_goal` in Move Time mode), `apple_exercise_time_goal`, `apple_stand_hours_goal` — one row a day, the goal Apple set that day. Not goalable.
- **GAD-7 and PHQ-9**: `gad7_assessment`, `phq9_assessment` — the value is the score, `meta.risk` its band (`none_to_minimal` … `severe`). The answers are kept on the record and never served by any read. State the score and band as recorded; never interpret them, never infer an answer, and never set a goal on them.
- **Heartbeat series**: `heartbeat_series` — the value is the series' mean heart rate; `meta.beats` holds each beat's offset in seconds from the start (`meta.gaps`: indices of beats after a gap).
- **ECGs**: `electrocardiogram`, an event whose value is Apple's classification code (1 sinus rhythm, 2 atrial fibrillation, 3–6 inconclusive, 100 unrecognized); `meta` carries the average heart rate and whether symptoms were reported. The voltage trace is stored apart: `argus metrics trace electrocardiogram <external-id>` (MCP `get_sample_trace`) returns microvolts at `hz` from the sample's start, by the sample's `provenance.external_id`. Report Apple's classification as recorded; never read a diagnosis into a trace.
- **Audiograms**: `audiogram` — the value is the better ear's average hearing threshold over 500, 1,000, 2,000 and 4,000 Hz in dB HL; `meta.points` holds every tested frequency, each ear.
- **Vision prescriptions**: `vision_prescription`, one event per prescription; `meta` holds glasses or contacts, each eye's sphere, cylinder, axis and add, and the expiry. Health shares them one at a time, like medications.

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
argus metrics get headache --agg count --bucket day     # a category's events per day
argus metrics get step_count --representation timeline --start 2026-09-06T00:00:00+02:00 --end 2026-09-07T00:00:00+02:00 --paginate
argus metrics get step_count --representation timeline --agg sum --bucket hour --start 2026-10-04T00:00:00+02:00 --end 2026-10-18T00:00:00+02:00
argus metrics get sleep_analysis --sessions     # newest derived sleep sessions with per-stage minutes, one page
argus metrics get sleep_analysis --sessions --start 2026-09-28T00:00:00+02:00 --end 2026-10-26T00:00:00+01:00 --paginate
argus trends --metrics body_mass --period 30d   # smoothed series + delta
argus trends --metrics body_mass --period 365d  # Year view
argus medications                               # per-medication Apple Health grants
argus medications sync --data '{"medications":[...]}'  # inserts and updates; never archives
```

Medication sync clients use MCP tool `upsert_medications` with the user-shared snapshot; it inserts and updates and never archives (only the Argus app's own key may mark a list complete). Do not infer a prescription or schedule, and do not interpret an empty result from tool `get_medications` as proof that the user takes none.

Aggregation must match the metric's style (summing heart rate is a 400 — the error's `remediation` explains). `sum` fits cumulative metrics (steps, dietary_*); `avg/min/max/latest` fit discrete ones (heart_rate, body_mass). A category metric's value is a code — a symptom's is its severity (2 mild, 3 moderate, 4 severe) — so `count` is its figure: how many events, with rows that record absence (`not_present`, an idle stand hour, no bleeding) left out. Never sum a category: one moderate headache sums to 3. Trends, the overview's day figure, goals and a Question's daily evidence on a category already count events (the evidence row reads unit `event` under its label `sum`). State of Mind is the one category that never counts: its day is a 0–100 mood score (valence −1…1 rescaled), read bare, never "N events".

For intraday shape use `representation=timeline` on `GET /v1/metrics/{name}`, or `representation: "timeline"` with MCP `get_metrics`. Supply both bounds, at most seven days apart, and follow every `next_cursor` without changing them. Timeline reads include intervals crossing the start boundary, exclude merged HealthKit day totals, and apply raw source precedence. Clip intervals to the displayed profile-timezone day before charting. A merged day total cannot describe when the activity happened; never distribute it across 24 hours or add it to raw observations. Timeline shapes are not authoritative daily totals: read those from summary/trends. The default raw representation remains a stored-row inventory. Timeline cannot be combined with sleep sessions.

Sleep sessions (`--sessions`, MCP `get_sleep_sessions`, `GET /v1/metrics/sleep_analysis/sessions/pages`) are nights gap-stitched from the stage segments, newest first, as a `{ data, next_cursor }` page like every other metrics read (`--limit` 1–200, default 50). `--start`/`--end` select sessions by their `date`, the wake date: on or after the start's day and before the end's day, both read in the profile timezone, so a night that began before midnight belongs to the morning it ended. Without `--start` the window reaches 400 days back; without either bound you get the newest sessions. A non-null `next_cursor` means the window may hold more (a page can even come back empty on a very dense history): pass it with `--cursor` and the same bounds until it is null, or use `--paginate` for the whole window as one array. Never treat one page as the full list. A window's session count can exceed its days with sleep when a day holds a nap. The older `GET /v1/metrics/sleep_analysis/sessions` (a bare array, `limit` ≤ 100, no window) stays for released apps.

For hourly shape across several days, do not page and sum the timeline yourself: ask for `representation=timeline&agg=sum&bucket=hour` (MCP `aggregate_metrics` with `representation: "timeline"`, `agg: "sum"`, `bucket: "hour"`). It takes a quantity metric and both bounds, at most fourteen days apart (337 hours, so fourteen local days hold across a daylight-saving change), and answers in one response with `representation: "timeline"`: each point is one hour's sum of the same rows the timeline lists. An interval is shared across the hours it overlaps in proportion to the overlap and clipped to the window; a point lands whole in its hour. Hours count from `start`, so pass the profile timezone's local midnight to get local clock hours. An hour with no observation is omitted, never a measured zero, and a day with no points recorded no intervals at all. These sums are shape — a pace, a usual-by-this-hour — never day totals; a day's points need not add up to its summary total. Any other `agg` or `bucket` over the timeline is a 400.

Apple-resolved day totals are the daily truth for `step_count`, `active_energy_burned`, `apple_exercise_time`, `apple_stand_time`, `distance_walking_running` and `flights_climbed`: the iPhone writes them for recent days and the Apple Watch for the running day, so today keeps moving while the phone is locked. While a day runs its total only rises; once it ends the iPhone's settled total stands. Historical step totals import newest first in resumable calendar-day pages. A confirmed raw HealthKit deletion invalidates the affected stored resolved total, including older days; summaries then use the remaining source-precedence rows until a fresh Apple total lands. An empty HealthKit query alone does not authorize deleting totals or reporting zero. Never add resolved totals to their raw samples, and never write `meta.device: "resolved"` yourself — it is the devices' sync claim.

## Workout records

```sh
argus workouts list --paginate
argus workouts get workout_abc
argus workouts log --data '{"activity":"running","start":"2026-08-30T07:00:00+02:00","end":"2026-08-30T07:30:00+02:00"}'
argus workouts edit workout_abc --data '{"activity":"walking"}'
argus workouts tag workout_abc --muscles chest,triceps
argus workouts delete workout_abc --confirm
```

A HealthKit workout recorded on iOS 27 or later carries `zones` on its detail: for each zoned quantity (`heart_rate`, cycling power…), the configured zones' bounds in that metric's canonical unit and the seconds spent in each, with `source` saying whether Apple, the person or an app set them. The field is absent when a workout has none. Inspect a workout before editing or deleting it. Do not duplicate a HealthKit-synced session or invent distance, energy, zones, or route detail. MCP tools: `get_workouts`, `get_workout`, `log_workout`, `edit_workout`, `tag_workout_muscles`, `delete_workout`.

Every workout carries `muscle_groups`: the fine muscle groups the person tagged, or null when untagged. Tags light those muscles on the body in place of the activity's own table (see the reading-the-body skill). `argus workouts tag <id> --muscles legs,core` takes the seven chips (chest, back, shoulders, arms, core, glutes, legs) or fine groups (triceps, lats, quadriceps…) and sends the expanded fine groups; `--clear` removes them. Tag only what the person says the session worked — never guess from the activity name. Tagging changes no statistics, energy or Activity projection, and it is not written to Apple Health.

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
- Delete only an identified user-authored sample with `argus metrics delete-sample <metric> <sample_id> [--start <sample start>] --confirm` — pass the start you read with the sample (MCP: `start` on `delete_metric_samples`) so the delete touches one time slice instead of the whole history; source projections must be undone at their meal, workout, supplement, or sleep record (`sleep_duration` and the stage metrics are projected from `sleep_analysis`). A delete rewrites what its metric feeds, as a write does: deleting a height or body mass re-projects `body_mass_index`, and with no height left none remains.
- `body_mass_index` projects only from a height (0.5–2.72 m) and a body mass (2–650 kg) inside human bounds. An impossible reading is stored and shows on its own metric, but projects nothing, and the height before it stays in force.
- Nothing changes a sample's value in place. A wrong reading or symptom severity is deleted, then logged again.
- Daily numbers (`summary`, day buckets) follow the **profile timezone**; after the person confirms their timezone, record it with `argus profile set --timezone Europe/Zagreb --confirm-timezone`. A stored timezone alone does not prove confirmation; read `timezone_confirmed` from the profile, including for UTC.
- A sample's or workout's own `timezone` is an IANA name. Offset spellings (`GMT+01:00`, `UTC+01:00`) are stored as their `Etc/GMT-1` zone; an unreadable one is refused as `invalid_timezone` — per item in a batch receipt, never the whole batch.
- An empty medications response is normal: access is granted per medication
  inside the Health app, and denied items are deliberately invisible.
- `metrics overview` returns both `latest` (one sample) and `day` (the last profile-timezone day's style-aware figure). Do not substitute one for the other. Its `count` is the samples of the last 90 days, not all history: a metric with only older rows lists with `count: 0` and its `latest`.

## Automatic Apple Health mirror

With Apple Health write permission, the connected iPhone/iPad mirrors Argus-authored meals, supplement nutrient contributions, body measurements, mood, water, sleep stage intervals and workouts. Writes accepted from web, Mac, Watch, CLI, MCP and chat reach the same path on the phone's next sync; they do not require a second agent write. Edits and deletes follow through a durable device cursor. The mirror is best effort while the phone is offline or permission is denied. Do not claim a Health write succeeded from an Argus receipt alone.

Measured HealthKit data never echoes through this path. HealthKit dietary imports remain excluded except water; mirrored water carries Argus sync identity and is skipped on import. Sleep duration/stage totals remain derived, never extra sleep sessions. A workout carries its recorded interval and energy; Apple Exercise Time is not directly writable and Activity ring credit remains Apple's decision. Medications have no public HealthKit write API (iOS 26.5 or 27.0): neither a dose nor a medication can be added to Health, so doses logged in Argus remain Argus-only.

Skills version 1.11.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
