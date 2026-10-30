---
name: reacting-to-events
version: 1.18.0
description: Use when an agent needs to follow Argus ledger changes live, inspect event history, or configure and verify a signed webhook subscription.
license: MIT
---

# React to Argus ledger events

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Use the tail when the agent can hold a connection but has no public URL:

```sh
argus events list --kinds meal.logged,goal.met
argus events tail --kinds meal.logged,goal.met
```

`argus events tail` emits one JSON Event per line. Resume with `--cursor <event-id>`. The MCP read mirror is tool `list_events`; MCP has no SSE passthrough.

Kinds join in generations, so a released client is never sent a kind it cannot parse. A read that names no kinds (`GET /v1/events`, `GET /v1/events/stream`, or the CLI without `--kinds`) sends only the kinds of the generation the client declares in the `x-argus-events` header; a client that sends none is generation 1. Generation 2 added `supplement.created`, `supplement.updated`, `supplement.archived`, `supplement.untaken`, `regimen.*` and `medications.updated`; generation 3 added `document.read`, `document.recorded`, `document.key_reset` and `lab_result.recorded`, so a client that declares 2 is sent none of those four; generation 4 added `export.prepared`; generation 5 added `medication_dose.skipped`, so a client that declares 4 is never sent a skip. This CLI sends the current generation and sees every kind; a CLI through v1.41.0 sends none and a later one may declare an older generation, so update the CLI or name the kinds you want. A read that names kinds gets them whatever the header says. The current generation is 5: over raw HTTP, send `x-argus-events: 5` once your parser takes every kind listed below.

Use a webhook for a durable public receiver:

```sh
argus events subscribe https://agent.example/argus --kinds meal.logged,goal.met --confirm
argus events test evs_0123456789ABCDEF
argus events unsubscribe evs_0123456789ABCDEF --confirm
```

The signing secret is shown once. Store it as a credential. Deliveries are at-least-once and best-effort ordered: deduplicate on `X-Argus-Delivery`, then reorder by the Event `id` when ordering matters.

The v1 kinds are closed: `meal.logged`, `meal.edited`, `meal.deleted`, `workout.logged`, `workout.edited`, `workout.deleted`, `body.recorded`, `mood.logged`, `supplement.taken`, `sample.recorded`, `sync.landed`, `goal.met`, `goal.lost`, `attention.opened`, `attention.cleared`, `experiment.ended`, `recipe.created`, `recipe.edited`, `recipe.deleted`, `medication_dose.logged`, `supplement.created`, `supplement.updated`, `supplement.archived`, `supplement.untaken`, `regimen.created`, `regimen.updated`, `regimen.deleted` and `medications.updated`, plus the context and question kinds below. A meal logged from a recipe still arrives as `meal.logged` — the recipe kinds cover the saved recipe itself, not its logs. An `experiment.ended` notice carries the experiment id and outcome metric names; fetch it with `argus experiments get` or MCP tool `get_experiment` and print its verdict sentences verbatim.

## Supplements and medications

A supplement's own record emits `supplement.created`, `supplement.updated` and `supplement.archived` (payload: `supplement_id`, `name`); archiving an archived supplement again emits nothing. Its regimens emit `regimen.created`, `regimen.updated` and `regimen.deleted` (payload: `regimen_id`, `supplement_id`). `supplement.taken` is one serving recorded; `supplement.untaken` is that take undone (`argus supplement undo`, MCP tool `delete_supplement_intake`), with the same payload as the take it removed: `intake_id`, `supplement_id`, `regimen_id`, `servings`, `taken_at`. A checklist or a due count that follows events re-reads the supplement checklist on any of these; never adjust a count from a payload alone.

`medications.updated` says a medication snapshot from the iPhone changed the list: its payload holds three lists of `concept_id`s — `added` (new to the ledger, or back in Apple Health after being removed), `edited` (a name, form, schedule, coding or Health's own archive flag changed) and `archived` (missing from a complete snapshot, so removed from Apple Health). An archived medication keeps every dose it recorded; it only leaves the active list. A snapshot that changed nothing emits nothing. Re-read `argus medications list` (MCP tool `get_medications`) before naming a medication.

A `medication_dose.logged` notice is one confirmed dose recorded through Argus's own write path (`argus medication take`, MCP tool `take_medication_dose`); its payload carries the medication `concept_id` and the `taken_at` instant, and it always means taken. A dose the person recorded as skipped emits `medication_dose.skipped` instead (generation 5), with the `concept_id` and the `skipped_at` instant; a skip is never a taken dose, and never arrives as `medication_dose.logged`. Doses imported from Apple Health arrive inside `sync.landed` batches, never as per-dose events. React with counts and facts only — never score adherence or call an absent dose missed.

## Documents

A medical document emits `document.read` once its reading settles — `status` is `to_review` (with the counts of proposed results and facts) or `failed` — and `document.recorded` when the person records it (with the counts it filed). The payloads carry the document id and counts only, never a value, a name or a word of the document. On `document.read` with `to_review`, fetch it with `argus documents get` or MCP tool `get_document` and walk the person through it; record nothing until they agree. A failed read carries its reason on the document itself. `document.key_reset` means the person reset their document key after losing it (its payload is only `reset_at`): their originals were deleted first, their results and facts remain, and Records waits to be set up again in an app. Never treat it as something to undo.

`lab_result.recorded` is one result joining a test's history: one per result when a document is recorded (`source: document`, with its `document_id`), and one per new Apple Health lab result (`source: health_records`; a record rewritten later, or projected for the first time from an older sync, emits none). Its payload carries `result_id`, the `test` key, `loinc` when matched, the `name` as printed, `collected_at` and `flag` (`above`, `below` or `within` the lab's own range, absent without one) — never the value. Fetch the history with `argus labs get <test>` or MCP tool `get_lab_history` before saying anything about it, and state what the lab printed; never diagnose.

## Exports

`export.prepared` means the person started an export of their record in an Argus app (Account › Export): its payload is the `export_id`, the `format` (`json`, `csv` or `fhir`), the `from` and `to` days, how many sealed `originals` the device will include, and the `device_id` that made it (null for a web session). It never carries a word of the export, and Argus keeps no copy. Treat it as the record leaving by the person's own act: you may mention it when asked what happened, never act on it, and never try to fetch the export — no route serves one to an agent.

## Life events and recovery reports

The context kinds are `life_event.created`, `life_event.edited`,
`life_event.deleted`, `symptom_episode.created`, `symptom_episode.edited`,
`symptom_episode.deleted`, `episode_observation.created`,
`episode_observation.edited` and `episode_observation.deleted`.

These notices carry resource ids and revisions, never the reported note,
severity or diagnosis. Fetch the current life event/episode before using its
content. An observation notice also carries its parent episode id and new
`episode_revision`; any observation change advances that parent revision.
A deletion's revision is one beyond the last stored version. Episode deletion
invalidates all its observations in one notice, without per-observation spam.
A rejected edit or replayed creation emits no new notice. These events record
ledger changes; they are not medical alerts or a following promise.

## What produces `sync.landed`

`sync.landed` is emitted **once per uploaded batch**, not once per sync run, so
one device sync usually produces several events in quick succession. Treat
events less than a few minutes apart as one arrival rather than reporting each
as a separate sync.

The payload carries `source`. Only `source == "healthkit"` is a device sync;
manual and agent writes emit `sync.landed` too.

`argus sync request` asks the account's devices to upload now, which is one of
the things that produces these events — best-effort, and never a guarantee that
any will follow.

## Verify a signature

Verify the HMAC over the exact raw request body before parsing JSON. This Bun example is timing-safe:

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

const raw = await request.text();
const received = request.headers.get("x-argus-signature") ?? "";
const expected = "sha256=" + createHmac("sha256", secret).update(raw).digest("hex");
const valid =
  received.length === expected.length &&
  timingSafeEqual(new TextEncoder().encode(received), new TextEncoder().encode(expected));
if (!valid) return new Response("invalid signature", { status: 401 });
const event = JSON.parse(raw);
```

Events are compact change notices. Fetch the underlying resource before acting. Poll tool `get_today` instead when only current day state matters and no reaction latency is required.

Question events `question.created`, `question.updated`, and
`question.deleted` contain only question_id and revision. Reread the
current question before acting; a deletion's revision follows its last stored
revision. These events signal question/evidence edits or state changes, not new
analysis findings. Paused and archived questions do not imply a clinical outcome.

Check-in events `check_in.created`, `check_in.edited` and `check_in.deleted`
contain only check_in_id and revision. Fetch the check-in before acting: an
edit can mean an explicit answer, correction, skip, snooze or reopen. It is
not a measurement or evidence of illness, adherence or continuous following.

Question lifecycle emits `question.created`, `question.updated`, and `question.deleted`. Saving an immutable answer emits `answer.saved`. Explicit following changes emit `following.started`, `following.paused`, `following.resumed`, `following.stopped`, and `following.expired`. A recorded receipt emits `update.recorded`; marking it reviewed emits `update.reviewed`. Read the referenced question or receipt before acting. These events do not establish causes, silently compute answers, or authorize a new write.

Skills version 1.18.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
