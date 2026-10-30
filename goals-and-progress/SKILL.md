---
name: goals-and-progress
version: 1.9.0
description: Use when setting or clearing nutrition/health targets in Argus, checking daily progress against goals, reading the daily summary dashboard, or analyzing trends over time.
license: MIT
---

# Goals and progress with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## Setting targets

```sh
argus goals set dietary_protein 140             # at_least, per day (defaults)
argus goals set body_mass 80 --direction at_most
argus goals set body_mass 176 --unit lb --direction at_most   # converted to kg at the edge
argus goals set active_energy_burned 3500 --period week
argus goals clear dietary_sugar                 # exits 4; re-run with --confirm
```

Any registry metric can be a goal. Direction: `at_least` (hit the floor) or `at_most` (stay under). Period: `day` or `week` (ISO week, profile timezone).

Two category metrics evaluate on their own scales rather than a raw sum: a `state_of_mind` goal scores the window's average valence rescaled to 0–100 (very unpleasant 0, neutral 50, very pleasant 100) — set its target on that 0–100 score, not on raw −1…1 valence — and stays unevaluated when no mood is logged — never zero; a `medication_dose` goal counts doses recorded as taken — a skipped dose adds nothing, and a window with none taken is a real 0.

### A goal on a lab test

```sh
argus goals set lab:13457-7 100 --unit mg/dL --direction at_most --period ongoing   # LDL under 100 mg/dL
```

A lab goal is `lab:<loinc>`, always `ongoing` (`lab_goal_ongoing` otherwise). Its target stays in the unit it was set in — the test's latest unit when `--unit` is omitted, and only a unit the test's results convert to (`invalid_unit`; `unit_required` when the test has no result yet). It is scored on the latest result collected within 365 days, converted into the goal's unit; with none it is unevaluated (`has_data: false`), never met. Its progress carries `label` (the test's name as printed) and `baseline` (the oldest result in that year): read it as a journey from baseline to target, and call it off course only when the latest moved away from the target. Read the test's own history with `argus labs get <loinc>`.

## Reading progress

```sh
argus summary                                   # today: goals, nutrition totals, meals count, latest metrics
argus summary --date 2026-08-24                 # any day
argus goals list                                # every goal with value/progress/met
argus goals adherence step_count --week 2026-W43 # one goal met or missed over a dated window
argus goals adherence dietary_protein --start 2026-10-01 --end 2026-10-28
argus trends --period 30d                       # goal metrics + body_mass, smoothed daily
argus trends --metrics dietary_protein,body_mass --period 90d
argus trends --metrics body_mass --period 365d   # Year view
argus today                                      # one-call day context
argus brief                                      # today's Brief: status, the insight, also-new lines
argus brief --date 2026-09-20                    # a past day: its review beside that day's insight
argus brief --week latest                        # the note on the newest settled week
argus brief timeline --limit 10                  # past insights, newest first
```

`argus goals adherence <metric>` (MCP tool `get_goal_adherence`) answers "did I meet my steps goal last week": pass `--week` with last week's ISO week, or `--start`/`--end` (local dates, inclusive, at most 98 days asked — a week goal's then widen to whole weeks); neither reads the 28 days ending today. A day goal is marked per day; a week goal per ISO week, Monday to Sunday, dated its Sunday — a closed week is judged whole, never seven days counted back from today, so a week goal's `start` and `end` widen to whole weeks. Each cell's `mark` is `met`, `missed`, `partial` (the bucket holding today: plotted, never judged the wrong way) or `none` (nothing to judge: an at-most day with nothing recorded proves nothing, while an at-least day with nothing recorded is `missed`). `met` of `total` counts the judged buckets; nothing past the profile's today is read. A lab goal has no days to adhere on (`goal_not_found`), and an ongoing goal is a journey scored on its latest measurement, never met or missed by day or week (`goal_ongoing`, 409): read it with `argus goals list`.

`summary.goals[].progress` is 0–1+ (1 = met for at_least; for at_most, 1 means within target). `trends` series carry `average` and `delta` (last − first) — the numbers to quote when the user asks "am I improving?".

## The Brief

`argus brief` (MCP tool `get_brief`) is the ledger's own editorial — what is new since yesterday — never a computation you should redo. Today's **daily Brief** is written once, when last night lands (else at 10:00 local), and then frozen: no read, write or hour changes it. The one exception is a Brief written without a night, which the night may rewrite once if it lands before noon. A read never writes it.

| `status` | meaning |
|---|---|
| `pending` | not written yet: last night has not landed and it is before 10:00 |
| `written` | an `opening` over one to four cards in `insights` |
| `quiet` | nothing moved outside the person's usual — a correct answer, not a failure |

**The edition (brief v4).** `opening.sentence` (at most 16 words) ties the day together — Today's card and the notification — and `opening.lead` (30–60 words) is the page's paragraph. A quiet day's sentence is the fixed line "A quiet day: nothing moved outside your usual overnight." and its lead is `null`. `insights` holds up to four cards — at most three changes and one pattern — in the server's rank, the one the opening leads with first. Every new key (`opening`, `points`, `insights`, `offer`, `feedback`) is absent while empty, never `null` or `[]`.

**Key points** (`points`, up to three) sit under the opening: each `{ card, metric, figure, text, short }`, templated by the server from the cards on the night's measures — HRV, resting HR, sleep, breathing, wrist temperature, blood oxygen — and frozen with the edition, never written by the model and never about a lab result. `text` reads "HRV 68 ms, your highest in 90 days" or "Slept 7 h 48, 36 min over your usual" (at most 48 characters), written in the person's display units when the edition was written; `short` is the figure alone for the glances. Every "usual" in a point is the one usual Argus prints for that measure (the 28-day mean, needing 7 days). Quote them as written; a quiet or pending day has none.

Each card is `{ id, kind, metric, label, headline, sentence, figure, evidence?, source, opens }`:

| field | what it holds |
|---|---|
| `kind` | `recovery` (HRV, resting HR and time asleep: at least two of the three ran beyond their usual over the week, on the same side; better and worse follow each measure's own direction — a lower resting HR is better), `changed` (a value beyond the person's own usual, a 7-day average crossing a goal, sleep moving 30 min from usual, the body-mass trend turning, training load crossing 1.3 or falling under 0.8, or a lab result outside the lab's range or moved a tenth), `record` (a 90-day best), `streak` (a goal met 7, 14, 30, 60 or 100 days running, or a run of 7+ broken), `pattern` (a relationship newly found in the person's own record, or one whose band rose), `question` (new evidence on a followed Question, or an Answer's verdict changing) or `experiment` (halfway, the last day, a verdict). The eyebrow reads `Recovery · 3 measures`, `Changed · Resting HR`. |
| `metric` | the lead metric; `null` for a question or a lab result |
| `label` | at most 3 words: the eyebrow's second part |
| `headline` | at most 14 words — Today's card. Lead with it. |
| `sentence` | at most 32 words — the Brief page's line |
| `figure` | exactly as Argus prints it (`58 bpm`, `+18 min`, `Day 14 of 28`) or `null`. Quote it verbatim; never recompute or reformat it. |
| `evidence` | what the card draws, from the ledger: `series` (points, the usual band, a goal line, the marker), `panels` (Recovery: one 14-day panel per measure, never two scales on one chart), `marks` (a streak's met or missed days), `groups` (a pattern: the outcome on days at or past the cause's `cut` against all other days — each day's value, the mean and the count, with the `lag`) or `progress` (an experiment's day count). Describe it; never re-derive it. |
| `source` | `{ window, devices, attention? }`: the window the figure covers (`30 nights`, `Last 14 days`, `180 days`), the devices behind the readings (`Apple Watch`), and `attention: true` when the card restates an attention line |
| `opens` | `{ kind, id }` — `metric`, `goal`, `pattern`, `question`, `experiment` or `lab` — where its figure came from, or `null` |

`offer`, when present, is one experiment the person may start, templated by the server from one of their own patterns (see `running-experiments`). `feedback` is the person's *Useful* or *Not for me* on the edition: it orders what the Brief picks next within each rank tier, and never hides an attention line or an escalation. Agents read `offer` and `feedback`; only the person's apps write feedback or decline an offer.

`dismissed_at` is set when the person put today's card away on any of their devices; every read carries it. It is their reading state, not a fact about their health — never re-surface a dismissed Brief unprompted.

A past date answers its **review** (`kind: "review"`, written once after the day settles — the night that followed landed, or 48 h passed) beside that day's own `status`, `insight`, `also_new` and `chart`; a review carries no cards. `--week latest` answers the note on the newest **settled** closed week (Mon–Sun; a week settles on the Wednesday after it) and `--week 2026-W35` a named one. A review or a week note keeps the older shape: a `headline` of at most 14 words, a `lead` paragraph and up to eight `notes`, each `{ section, label, figure, text, ref }` with `section` one of `night`, `readiness`, `yesterday`, `today`, `week`, `experiments`, `record` — or `checklist` on a note written before 2026-10-21, when supplements and medications shared one heading. A pending supplement or a medication's doses now read under their day, each in its own note.

**The older shape, for released readers.** On the daily Brief, `insight` is the first card in the v3 shape (`kind` one of `outlier`, `streak`, `record`, `trend`, `question`, `experiment`: Recovery and Changed read `outlier`, a crossing `trend`, a Pattern `question`), `also_new` holds the other cards as lines whose `text` is the card's headline, and `chart` is the first card's evidence (Recovery's HRV panel; a Pattern's outcome series with no `goal`). Its `ref` is set only for a metric, goal or experiment; a lab result keeps `metric: null`. `headline`, `lead` and `notes` mirror those again for older readers. Read `opening` and `insights` when present; a day written before brief v4 carries only the older keys. `paragraph` is a deprecated mirror of `lead`.

A null `lead` carries a `reason`: `nothing_to_say` (a pending or quiet day, or nothing stored under `--cached`), `unavailable` (a review or week note could not be written), `rate_limited` (the day's cap of four written editions is spent) or `generating` (a review or week note is being written now). Report it as it stands and never write a replacement Brief of your own.

`argus brief timeline` (MCP tool `list_brief_timeline`) lists the past daily Briefs that carried an insight, newest first — `date`, `kind`, `metric`, `label`, `headline`, `sentence`, `figure`, `dismissed` (the first card), and `count` when the edition held more than one card — paged with `--before` and `next_before` (`--paginate` follows it to the end). Quiet days are absent by design. Read it before restating something: the Brief never tells the same card twice within 30 days unless it went further — worse than when told, a later milestone, a new record, a pattern's band rising. A Recovery card counts as told for each of its measures, so the next day's HRV change stands alone only when it went further.

## Patterns

- "How am I doing today?" → `argus summary`, lead with unmet goals.
- "Log X and tell me where I stand" → `argus log "…"` then `argus summary` (projection is immediate).
- "What's new?" → `argus brief`; quote the opening's sentence, its points, then each card's headline in order; say `pending` or `quiet` as it stands.
- A trend's points are whole profile-local calendar days, today − (N − 1) through today; today holds what is recorded so far. A `7d` trend is seven dates, never a partial eighth.
- Weekly review → `argus brief --week latest` for the note, then `argus trends --period 7d` for the figures.
- Year review → `argus trends --period 365d`; preserve missing days and do not infer causation.
- A newly recorded lab result outside the lab's range, or one that moved at least a tenth from the previous result of its test, can be a `changed` card (`metric` null, the label the test's name, `opens` the lab test); its evidence is the test's history, banded by the lab's own range. On such a day the notification's body is a generic sentence, never the result.
- Readiness is against personal baselines, not a goal. Use `argus readiness` and preserve null/unknown contributors.

A review or a week note that has settled but was never written is written on the read that asks for it: pass `wait=1` (the CLI does unless `--cached`) to wait within a 90-second window. It can still return `generating`, so read again later without inventing a replacement note. Preparation and transport failures remain errors; they are not proof that a note is being written. `mode=cached` only reads storage, and today's Brief is only ever read.

Skills version 1.9.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
