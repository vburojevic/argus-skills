---
name: goals-and-progress
version: 1.4.1
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
argus brief                                      # the day's Brief: headline, lead, rows — may be null
argus brief --week latest                        # the note on the newest closed week
```

`summary.goals[].progress` is 0–1+ (1 = met for at_least; for at_most, 1 means within target). `trends` series carry `average` and `delta` (last − first) — the numbers to quote when the user asks "am I improving?".

## The Brief

`argus brief` (MCP tool `get_brief`) is the ledger's own editorial, never a computation you should redo. It answers in four **kinds**: `morning` (the live day until 17:00 local), `evening` (the live day from 17:00), `review` (any past date, written in the past tense) and `week` (`--week latest` or `--week 2026-W35`, the closed ISO week Mon–Sun).

Each answer opens with a `headline`: one sentence of at most 14 words, the one thing to know. It is the whole line Argus's own teaser, widgets, watch and menu bar show, so lead with it when you summarize. Behind it sits the `lead`, one 40–60 word paragraph, and up to eight `notes`. The headline is never the lead's first sentence repeated — say one or the other, not both.

A note is one ledger row, not a sentence with a link:

| field | what it holds |
|---|---|
| `section` | `night`, `readiness`, `yesterday`, `today`, `week`, `experiments`, `checklist` or `record` — the part of the record the fact came from. `record` covers relationships, reference bands and anything the other seven do not name. Argus's own screen groups the rows in that order. |
| `label` | at most 3 words, the row voice: `Sleep`, `Protein`, `Resting HR`, `L-theanine`. |
| `figure` | the figure exactly as Argus prints it — `7:46`, `115 / 140 g`, `65 ms`, `Day 14 of 21` — or `null` when the fact has none. Quote it verbatim; never recompute or reformat it. |
| `text` | the sentence, at most 24 words. |
| `ref` | `{ "kind": "goal" \| "metric" \| "meal" \| "workout" \| "experiment" \| "supplement" \| "medication", "id": … }`, or `null` when the fact has no single record. |

Quote a label with its figure ("Sleep 7:46"); follow a ref to the record before elaborating on it. `paragraph` is a deprecated mirror of `lead` and will be removed — read `lead`.

A null `lead` always carries a `reason`. `nothing_to_say` means the day holds no facts worth a note — or, under `--cached`, that none was stored; `unavailable` means the model call failed; `rate_limited` means the person passed the 24-hour generation ceiling. Report the reason as it stands and never write a replacement brief of your own. `--cached` (`mode=cached`) answers from storage and never waits on generation — use it whenever a slow answer would be worse than no answer.

## Patterns

- "How am I doing today?" → `argus summary`, lead with unmet goals.
- "Log X and tell me where I stand" → `argus log "…"` then `argus summary` (projection is immediate).
- Weekly review → `argus brief --week latest` for the note, then `argus trends --period 7d` for the figures.
- Year review → `argus trends --period 365d`; preserve missing days and do not infer causation.
- Readiness is against personal baselines, not a goal. Use `argus readiness` and preserve null/unknown contributors.

Skills version 1.4.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.

A cold note takes about a minute to write. `GET /v1/brief` answers at once with what is on the record — the previous note, or `lead: null` with `reason: "generating"` — and finishes out of band; pass `wait=1` (the CLI does unless `--cached`) to block for the fresh note instead of reading again.

Skills version 1.4.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
