---
name: reading-the-body
version: 1.0.6
description: Use when reading or explaining the Argus body — its eleven systems, what moved against the person's own history, muscle load from workouts, symptom and episode pins on body sites, the body timeline, tagging a workout's muscles, placing an episode on the body, or the standard-figure setting.
license: MIT
---

# Read the body with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

The body is the person's record drawn on a figure in their own proportions. It is a map of the ledger, not a verdict on the person.

**Law 1 — the body never diagnoses.** No health scores, grades, population norms or "this organ is unwell". A lit system says two things only: a record lives here, and a record here moved against the person's own history. Say what the record shows ("resting heart rate 55, 7 below your usual"), never what is wrong, never a cause, never advice.

```sh
argus body
argus body --date 2026-09-20
argus body timeline
argus body timeline --span all
```

MCP tools: `get_body` reads one day (default today, profile timezone); `get_body_timeline` reads a span (default 30d). Both read a daily rollup Argus keeps as records land, so every span answers in a few seconds at most. Right after an account's first large import the rollup may still be building; the longer spans then read the raw record and can take longer. The figures are the same either way.

## The eleven systems

Every metric belongs to exactly one system (or to none, with a reason: a setting, a dose, the water's depth). Each system lights where its records live.

| id | Name | Leads with |
|---|---|---|
| `heart` | Heart & circulation | resting heart rate, HRV, VO₂ max |
| `lungs` | Lungs & breath | respiratory rate, blood oxygen |
| `mind` | Mind & sleep | sleep duration, state of mind, mindful minutes |
| `senses` | Senses | headphone and environmental audio |
| `digestion` | Digestion & nutrition | energy, water, caffeine, protein |
| `metabolism` | Blood & metabolism | blood glucose, basal energy |
| `muscles` | Muscles & training | active energy, exercise minutes, training load |
| `mobility` | Walking & mobility | steps, walking speed, steadiness |
| `skin` | Skin & temperature | wrist temperature, body temperature |
| `composition` | Body composition | body mass, body fat, waist |
| `reproductive` | Pelvic & reproductive health | menstrual flow, basal body temperature |

Per system, `GET /v1/body` gives:

- `freshness`: `today`, `week` (within 7 days), `month` (within 30), `older` (within a year) or `none`. It says how recent the system's records are. It lights nothing and says nothing about health.
- `has_records`: any record within 365 days. The reproductive organs are drawn only when this is true.
- `moved` and `moved_metrics[]` (largest |z| first): each with `value`, `baseline_mean`, `baseline_sd`, `z` and `direction`.
- `headline[]`: up to three lead figures with the instant they were recorded; `label` names a category value ("Medium", "Pleasant").
- `symptoms`: how many symptoms were logged that day.

## What "moved" means

- **Daily metrics:** the day's value against the 28 days that end the day before, needing at least 7 days with values. σ is the population deviation, floored at 5 % of the mean and at the metric's smallest meaningful step — except the metrics that sit far from zero on their own scale (temperatures, blood oxygen, weight, lean mass, waist, BMI, body fat, VO₂ max), whose floor is a fixed amount instead (0.1 °C of wrist temperature, 0.3 kg of weight). Moved when |z| > 1.5.
- **Measurements** (body mass, body fat, waist, VO₂ max, blood pressure, steadiness, lung function, six-minute walk): the latest value within 7 days against the up-to-eight before it within 180 days, needing at least 4.
- **Always moved:** a symptom logged that day, or a workout that day that works muscles.
- Too little history means **unknown**, never "fine" and never "moved". On a day with meals logged, a missing nutrient or drink is a real zero.

Quote the figures (`value`, `baseline_mean`, `z`) and say "against your own last 28 days". Never call a moved figure good, bad, high-risk or abnormal.

**Light is movement, and nothing else.** A record alone earns no light; a system with records but no move is plain on the figure. A moved system's light grows with its largest |z| (`bodyMovedLight`): about a third at 1.6σ, full from 3σ. A move without a z (a symptom logged, a workout that worked muscles) lights at the floor, and its pin or the muscle light says the rest. In words, a moved figure is "above" or "below" your usual under 3σ, and "well above" or "well below" from 3σ. Never a colour for direction, and never a number on the body.

## Pins, sites and episodes

- `pins[]` carry symptoms logged in the last 14 days (one per symptom, its latest logging; severity 1–3) and episodes that are open or resolved within 30 days (severity 0–10, `trend` from the last two rated observations). An episode belongs to its linked metric's system, else to its site's.
- A site is one of 37 surface regions — head, face, eyes, ears, nose, mouth, throat, neck, chest, breasts, upper and lower abdomen, pelvis, upper and lower back, and left and right shoulder, upper arm, elbow, forearm, hand, hip, thigh, knee, lower leg, ankle and foot (`left_knee`, `right_foot`) — or `whole_body` for records with no one place (fever, fatigue).
- Place an episode only where the person said they feel it. Never infer a site from a symptom's name or a measurement:

```sh
argus context episodes create --file episode.json --site left_knee
argus context episodes update sep_… --file episode.json --site none
```

The same field is `body_site` on the episode tools. An update that omits it keeps the stored site.

## Muscles and tagging

- `muscles[]` lists each muscle group with load above 0: Σ weight × intensity × 0.5^(hours since the workout ended ÷ 36) over the last 7 days, capped at 1. Intensity is minutes ÷ 60 (at most 1) × a heart-rate factor from the session's average heart rate, resting heart rate and age (0.6 when unknown). A group lights on the figure at 0.15.
- Weights come from the activity's table, or from the person's tags, which override it. Untagged strength training lights every group at 0.35 × intensity; an untagged `other` lights nothing.
- `untagged[]` lists strength and `other` workouts from the last 14 days that have no tags yet.
- Tag only what the person tells you they worked:

```sh
argus workouts tag wkt_… --muscles chest,triceps
argus workouts tag wkt_… --muscles legs,core
argus workouts tag wkt_… --clear
```

Tool `tag_workout_muscles` takes the same names. The seven chips expand to fine groups: chest → chest; back → upper back, lats, lower back; shoulders → shoulders, neck; arms → biceps, triceps, forearms; core → abs, obliques; glutes → glutes, hip flexors; legs → quadriceps, hamstrings, adductors, calves, shins. A name that is both a chip and a group ("chest", "shoulders") is the fine group. Never guess tags from an activity's name; ask.

Load describes the record. It is not readiness, fatigue or advice to rest.

## Vitals, goals and the Brief

- `vitals.resting_heart_rate` and `respiratory_rate` are the latest within 7 days (null stops the figure's heartbeat or breathing). `wrist_temperature_delta` is last night's wrist temperature minus its 28-day mean, in °C.
- `goals[]` are the standing (`ongoing`) level goals as journeys, one per metric (a goal is keyed by its metric): `baseline` (the first reading of the last 14 days), `current` (within 90 days) and `target`. The waist sits at `waist`; the rest at `whole_body`.
- `brief_mark` (today only) names the system of the day's written Brief insight.

## The figure

`shape.weights` and `shape.scale` come from height, weight, body fat, waist and lean mass in effect on the day, with profile sex. Age never shapes it. Missing measurements fall back to standard proportions. The shape reads a measurement only inside human bounds (height 50–272 cm, waist 30–250 cm, body and lean mass 2–650 kg, body fat 0–100 %); a reading outside is absent for the shape, which keeps the newest one inside, while the metric itself still shows it. The person may choose a standard figure; only they change it:

```sh
argus profile set --body-figure standard
argus profile set --body-figure yours
```

## The timeline

`argus body timeline` returns buckets: days for `30d` and `90d`, weeks ending Sunday for `1y` and `all` (the running week ends today). Each bucket is the body at its last day — systems on the record by then with their freshness, whether each moved within the bucket, muscle load at its end, and the symptom sites logged in it — plus `shape_keyframes` at measurement days to interpolate between. Each keyframe's `measured` names the body measurements taken that day, and its `values` carries those readings (`{metric, value, unit}`, canonical units), the figures the timeline's label prints ("14 Sep · 71.2 kg · 18 %"). A bucket's `moved` reads each system's lead metrics, plus its measurements, symptoms and workouts; the day's own `argus body` reads every metric, so a day can show a moved figure its bucket does not. In `all`, weeks older than a year mark movement from measurements, symptoms and workouts only.

## Worked transcript

```text
$ argus body --output json
{"date":"2026-09-27","systems":[{"id":"heart","freshness":"today","moved":true,"moved_metrics":[{"metric":"resting_heart_rate","value":55,"unit":"count/min","baseline_mean":62,"baseline_sd":3.1,"z":-2.26,"direction":"down"}], …}], "muscles":[{"group":"calves","load":0.44, …}], "untagged":[{"workout_id":"wkt_5Fs1Ne7c","activity":"traditional_strength_training", …}], …}

Your heart is lit: resting heart rate is 55 bpm today, 7 below your own last 28 days (z −2.3). Your calves carry this morning's run. Thursday's strength session has no tags yet — which muscles did it work?
```

Skills version 1.0.6 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
