---
name: goals-and-progress
version: 1.5.0
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

## Reading progress

```sh
argus summary                                   # today: goals, nutrition totals, meals count, latest metrics
argus summary --date 2026-08-24                 # any day
argus goals list                                # every goal with value/progress/met
argus trends --period 30d                       # goal metrics + body_mass, smoothed daily
argus trends --metrics dietary_protein,body_mass --period 90d
argus trends --metrics body_mass --period 365d   # Year view
argus today                                      # one-call day context
argus brief                                      # today's Brief: status, the insight, also-new lines
argus brief --date 2026-09-20                    # a past day: its review beside that day's insight
argus brief --week latest                        # the note on the newest settled week
argus brief timeline --limit 10                  # past insights, newest first
```

`summary.goals[].progress` is 0–1+ (1 = met for at_least; for at_most, 1 means within target). `trends` series carry `average` and `delta` (last − first) — the numbers to quote when the user asks "am I improving?".

## The Brief

`argus brief` (MCP tool `get_brief`) is the ledger's own editorial — what is new since yesterday — never a computation you should redo. Today's **daily Brief** is written once, when last night lands (else at 10:00 local), and then frozen: no read, write or hour changes it. The one exception is a Brief written without a night, which the night may rewrite once if it lands before noon. A read never writes it.

| `status` | meaning |
|---|---|
| `pending` | not written yet: last night has not landed and it is before 10:00 |
| `written` | one `insight`, and up to three `also_new` lines |
| `quiet` | nothing new since yesterday — a correct answer, not a failure |

The `insight` is one change against the person's own record, picked from candidates the ledger found:

| field | what it holds |
|---|---|
| `kind` | `outlier` (beyond their own usual: the median of the prior 28 days ± 2 × spread), `streak` (a goal met 7, 14, 30, 60 or 100 days running, or a run of 7+ broken), `record` (a 90-day best), `trend` (a 7-day average crossing a goal, sleep moving 30 min from usual, the body-mass trend turning, training load crossing 1.3 or falling under 0.8), `question` (new evidence on a followed Question, an Answer's verdict changing, a relationship newly found) or `experiment` (halfway, the last day, a verdict) |
| `metric` | the metric it is about, when it is about one |
| `label` | at most 3 words, the row voice: `Resting HR`, `Steps`, `Magnesium` |
| `headline` | at most 14 words — the notification, widget and watch line. Lead with it. |
| `sentence` | at most 32 words — the card's line |
| `figure` | exactly as Argus prints it (`58 bpm`, `14,210`, `Day 14 of 28`) or `null`. Quote it verbatim; never recompute or reformat it. |
| `ref` | `{ "kind": "goal" \| "metric" \| "meal" \| "workout" \| "experiment" \| "supplement" \| "medication", "id": … }` or `null`. A `question` insight names its question inside `id` (`question:<question id>:…`). |

Each `also_new` line is `{ id, kind, metric, label, figure, text, ref }`, `text` at most 24 words. `chart` is the ledger series behind the insight — `points` (`{ date, value }`, `null` where a day has no data), a `band` (the usual, labelled like `usual 50–54`), a `goal` line, the `marker` the insight is about and, for a streak, `marks` (met or missed per day). Every number in it comes from the ledger, never from a model: describe it, never re-derive it.

`dismissed_at` is set when the person put today's card away on any of their devices; every read carries it. It is their reading state, not a fact about their health — never re-surface a dismissed Brief unprompted.

A past date answers its **review** (`kind: "review"`, written once after the day settles — the night that followed landed, or 48 h passed) beside that day's own `status`, `insight`, `also_new` and `chart`. `--week latest` answers the note on the newest **settled** closed week (Mon–Sun; a week settles on the Wednesday after it) and `--week 2026-W35` a named one. A review or a week note keeps the older shape: a `headline` of at most 14 words, a `lead` paragraph and up to eight `notes`, each `{ section, label, figure, text, ref }` with `section` one of `night`, `readiness`, `yesterday`, `today`, `week`, `experiments`, `checklist`, `record`.

On the daily Brief, `headline`, `lead` and `notes` mirror the insight's headline, its sentence and the also-new lines for older readers — read `insight` and `also_new`. `paragraph` is a deprecated mirror of `lead`.

A null `lead` carries a `reason`: `nothing_to_say` (a pending or quiet day, or nothing stored under `--cached`), `unavailable` (a review or week note could not be written), `rate_limited` (the day's cap of four written editions is spent) or `generating` (a review or week note is being written now). Report it as it stands and never write a replacement Brief of your own.

`argus brief timeline` (MCP tool `list_brief_timeline`) lists the past daily Briefs that carried an insight, newest first — `date`, `kind`, `metric`, `label`, `headline`, `sentence`, `figure`, `dismissed` — paged with `--before` and `next_before` (`--paginate` follows it to the end). Quiet days are absent by design. Read it before restating something: the Brief never tells the same thing twice within 30 days unless it went further — worse than when told, a later milestone, a new record.

## Patterns

- "How am I doing today?" → `argus summary`, lead with unmet goals.
- "Log X and tell me where I stand" → `argus log "…"` then `argus summary` (projection is immediate).
- "What's new?" → `argus brief`; quote the insight's headline, then its sentence; say `pending` or `quiet` as it stands.
- Weekly review → `argus brief --week latest` for the note, then `argus trends --period 7d` for the figures.
- Year review → `argus trends --period 365d`; preserve missing days and do not infer causation.
- Readiness is against personal baselines, not a goal. Use `argus readiness` and preserve null/unknown contributors.

A review or a week note that has settled but was never written is written on the read that asks for it: pass `wait=1` (the CLI does unless `--cached`) to wait within a 90-second window. It can still return `generating`, so read again later without inventing a replacement note. Preparation and transport failures remain errors; they are not proof that a note is being written. `mode=cached` only reads storage, and today's Brief is only ever read.

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
