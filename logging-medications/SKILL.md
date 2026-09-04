---
name: logging-medications
version: 1.2.1
description: Use when listing medications shared from Apple Health, recording a confirmed medication dose in Argus, undoing a manual medication dose entry, or explaining how dose receipts feed relationships, experiments, events, and the context document.
license: MIT
---

# Log medication doses with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## What medications are in Argus

Argus mirrors the medication list a person shares individually from Apple Health. A medication carries its Apple Health concept id, display text, optional nickname, general form, schedule presence, archive state, and clinical codings. An empty list means no medications were shared with Argus; it does not prove the person takes none.

Dose events reach the same ledger in two ways:

- Apple Health imports them as `healthkit` medication-dose samples.
- A confirmed dose recorded through Argus becomes one `manual` `medication_dose` sample with status `taken`.

The Argus write records the event in Argus only. It does not create a dose event in Apple Health. When an Apple Health dose event overlaps a manual entry, the HealthKit event supersedes the manual one on reads. Neither row is deleted.

## Record a dose

First list the medication mirror and use its exact `concept_id`:

```sh
argus medications list --output json
argus medication take 'x-apple-health://medication/concept/example' --output json
argus medication take 'x-apple-health://medication/concept/example' --taken-at 2026-09-02T08:15:00+02:00 --output json
```

Record a dose only after the person confirms they took it. Never infer one from a schedule, notification, prior habit, or another health reading. Do not recommend a medication or amount.

The concept id belongs in the JSON body and may be URI-shaped. Do not build a path from it. Omitting `--taken-at` records the current instant; daily attribution follows the profile timezone.

Each command invocation records one honest row. Version 1 has no dose deduplication: invoking it twice at the same instant records two rows. Do not retry after an ambiguous network result without first checking the ledger.

## Undo a manual entry

Use the returned sample id with the existing sample deletion command:

```sh
argus metrics delete-sample medication_dose smp_abc123 --confirm
```

Only user-authored manual rows are deletable this way. Imported HealthKit rows remain HealthKit's record.

## How doses feed the ledger's thinking

The ledger reads its own dose receipts; none of this asks anything extra of the person:

- **Relationships.** Each medication earns a per-day `medication:<slug>` series counting its taken doses. A skipped dose contributes nothing, and a day without a taken dose is absent from the series, never zero. These receipt series join the correlation universe in one lane beside the `supplement:<slug>` and `supplement_ingredient:<slug>` series — the ten densest receipt series across all three kinds together, never ten of each. The per-metric read accepts receipt names: `argus insights --metric medication:levothyroxine`.
- **Experiments.** An N=1 run can bind to a medication's dose receipts via `medication_concept_id` — at most one binding per run, a supplement or a medication, never both. The running-experiments skill teaches the bound windows.
- **The context document.** The medications section states each active medication as facts — `Levothyroxine — scheduled; last taken 3 h ago; today 1 taken`. The last-taken clause reads a 90-day window; an older dose renders no clause, never a judgment.
- **The Daily Brief.** The fact sheet carries today's dose counts in the Today surface's grammar — "1 of 2 medication doses taken today" — and the paragraph may state them as a fact or leave them out. The denominator counts dose events (taken and skipped, plus scheduled doses not logged), so a taken as-needed dose earns the line too; a day with no dose events carries none. The Brief never instructs and never reads a count as adherence.
- **Events.** A dose recorded through the Argus write path emits `medication_dose.logged` with the `concept_id` and `taken_at` instant; doses imported from Apple Health arrive inside `sync.landed` batches, never per dose. The reacting-to-events skill holds the reaction rules.

In Ask Argus (chat), the model can list the medication mirror — name, schedule presence, last taken, today's taken count — and can propose recording a dose. A proposal is applied only by the person; applying it records the same honest `manual` row this skill describes.

## Tone

Argus is a ledger, not a nurse. Medications have **doses**; supplements have **servings**. State only what was recorded: never calculate adherence percentages, label an absent record "missed," score behavior, or turn a schedule into medical advice.

## Worked transcript

```text
$ argus medication take 'x-apple-health://medication/concept/example' --taken-at 2026-09-02T08:15:00+02:00 --output json
{"id":"smp_abc123","metric":"medication_dose","taken_at":"2026-09-02T06:15:00.000Z","name":"Example tablet"}

One confirmed dose was recorded at 08:15 local time. No adherence judgment was made.
```

MCP tools follow the same rules: `get_medications` reads the shared mirror, `take_medication_dose` records one confirmed dose, and `upsert_medications` replaces the complete HealthKit sync snapshot.

Skills version 1.2.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
