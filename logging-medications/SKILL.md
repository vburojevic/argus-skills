---
name: logging-medications
version: 1.8.0
description: Use when listing medications shared from Apple Health, recording a confirmed medication dose (taken or skipped) in Argus, undoing a manual medication dose entry, or explaining how dose receipts feed relationships, experiments, events, and the context document.
license: MIT
---

# Log medication doses with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## What medications are in Argus

Argus mirrors the medication list a person shares individually from Apple Health. A medication carries its Apple Health concept id, display text, optional nickname, general form, schedule presence, archive state, and clinical codings. An empty list means no medications were shared with Argus; it does not prove the person takes none. Apple Health shares medications one at a time, on a sheet of its own, and its settings list no medication toggles for Argus until Argus has opened that sheet. On iPhone, Argus offers it under the import offer when Apple Health is connected. If it has still not opened, Argus opens it once by itself the next time it comes to the foreground. After that it opens from Today's "Medications aren't shared" row or from Account → Medications → Choose medications to share. Opening the sheet once also gives Argus a switch per medication in Health → your profile → Apps → Argus → Medications, and Apple's sheet may not open again after that, so a person can share from there too. Argus reads whatever is switched on at its next sync, whichever way it was shared. When the list is empty, point the person to Argus → Account → Medications on their iPhone, or to those switches in Health. Before Argus has asked, Health shows no switches for it. The web and the Mac only read the list.

A medication removed from Apple Health leaves the active list on the iPhone's next sync. The phone reads every medication it may read on each sync and sends that list with `complete: true` (and `authorized: true`); a medication the ledger holds and a complete list lacks is archived (`is_archived: true`), and every dose it recorded stays readable. A list without `complete` — every release before this one — only adds and updates, and a complete empty list archives only when the device also reports medications authorization. A medication archived this way comes back if Health lists it again; one Health itself archived stays archived. `complete` and `authorized` are the device's claim and are accepted only from its own key (the one the Argus app mints at sign-in): from an agent's key, the CLI or a browser session, `PUT /v1/medications` refuses them with a `validation` problem naming each field, and nothing is archived. `argus medications sync` refuses them before sending, and MCP tool `upsert_medications` does not take them. An agent never archives a medication; to see one leave the list, wait for the iPhone's next sync.

Dose events reach the same ledger in two ways:

- Apple Health imports them as `healthkit` medication-dose samples.
- A confirmed dose recorded through Argus becomes one `manual` `medication_dose` sample with status `taken`, or `skipped` when the person says they skipped it.

The Argus write records the event in Argus only. Apple Health gives apps no way to write a dose or add a medication, so nothing recorded here reaches Health, and a person often confirms the same dose again at Health's reminder. A HealthKit dose recorded as taken for the same medication, logged or scheduled within two hours of a manual taken entry, supersedes it on reads: that dose counts once. A skipped HealthKit dose, another medication's, or one further off supersedes no manual taken entry; a manual taken beside a skipped HealthKit dose keeps counting as taken. A manual skip gives way to any HealthKit dose of the same medication in that window, taken or skipped, so Health's record of that dose decides. A skip never counts as taken anywhere: it adds to the day's dose events, never to the taken count or the per-medication series. Neither row is deleted.

## Record a dose

First list the medication mirror and use its exact `concept_id`:

```sh
argus medications list --output json
argus medication take 'x-apple-health://medication/concept/example' --output json
argus medication take 'x-apple-health://medication/concept/example' --taken-at 2026-09-02T08:15:00+02:00 --output json
argus medication take 'x-apple-health://medication/concept/example' --status skipped --output json
```

Record a dose only after the person confirms they took it, and a skip only after they say they skipped it. Never infer either from a schedule, notification, prior habit, or another health reading. Do not recommend a medication or amount. MCP tool `take_medication_dose` takes the same optional `status` (`taken` by default, or `skipped`).

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
- **The Daily Brief.** Today's Brief states what is new since yesterday, so today's dose counts are not in it — Argus's own Today strip counts the scheduled medications still to take, live, as a part of their own (`1 medication`), never added to the supplements. A day review may state a closed day's counts in the Today surface's grammar — "1 of 2 medication doses taken" — as a fact or leave them out. The denominator counts dose events (taken and skipped, plus scheduled doses not logged), so a taken as-needed dose earns the line too; a day with no dose events carries none. The Brief never instructs and never reads a count as adherence.
- **Events.** A dose recorded through the Argus write path emits `medication_dose.logged` with the `concept_id` and `taken_at` instant when it was taken, and `medication_dose.skipped` with the `concept_id` and `skipped_at` instant when it was a skip — never the taken kind; doses imported from Apple Health arrive inside `sync.landed` batches, never per dose. A snapshot that changed the medication list emits `medications.updated` with the `added`, `edited` and `archived` concept ids. The reacting-to-events skill holds the reaction rules.

In Ask Argus (chat), the model can list the medication mirror — name, schedule presence, last taken, today's taken count — and can propose recording a dose. A proposal is applied only by the person; applying it records the same honest `manual` row this skill describes.

## Tone

Argus is a ledger, not a nurse. Medications have **doses**; supplements have **servings**. State only what was recorded: never calculate adherence percentages, label an absent record "missed," score behavior, or turn a schedule into medical advice.

## Worked transcript

```text
$ argus medication take 'x-apple-health://medication/concept/example' --taken-at 2026-09-02T08:15:00+02:00 --output json
{"id":"smp_abc123","metric":"medication_dose","taken_at":"2026-09-02T06:15:00.000Z","name":"Example tablet"}

One confirmed dose was recorded at 08:15 local time. No adherence judgment was made.
```

MCP tools follow the same rules: `get_medications` reads the shared mirror, `take_medication_dose` records one confirmed dose, and `upsert_medications` inserts and updates medications from a snapshot the user shared; it never archives, and it refuses a snapshot that claims to be complete or authorized.

### Apple Health write restriction

Verified against the iOS 26.5 SDK: `HKMedicationDoseEvent` has read-only properties and no public initializer or save factory. Argus cannot create an Apple Health medication dose event through the supported API. A confirmed dose stays in Argus; do not substitute a symptom, nutrition sample, or private URL scheme. The general Apple Health mirror does not change this restriction.

Skills version 1.8.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
