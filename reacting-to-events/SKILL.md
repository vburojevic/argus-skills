---
name: reacting-to-events
version: 1.10.1
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

Use a webhook for a durable public receiver:

```sh
argus events subscribe https://agent.example/argus --kinds meal.logged,goal.met --confirm
argus events test evs_0123456789ABCDEF
argus events unsubscribe evs_0123456789ABCDEF --confirm
```

The signing secret is shown once. Store it as a credential. Deliveries are at-least-once and best-effort ordered: deduplicate on `X-Argus-Delivery`, then reorder by the Event `id` when ordering matters.

The v1 kinds are closed: `meal.logged`, `meal.edited`, `meal.deleted`, `workout.logged`, `workout.edited`, `workout.deleted`, `body.recorded`, `mood.logged`, `supplement.taken`, `sample.recorded`, `sync.landed`, `goal.met`, `goal.lost`, `attention.opened`, `attention.cleared`, `experiment.ended`, `recipe.created`, `recipe.edited`, `recipe.deleted`, and `medication_dose.logged`. A meal logged from a recipe still arrives as `meal.logged` — the recipe kinds cover the saved recipe itself, not its logs. An `experiment.ended` notice carries the experiment id and outcome metric names; fetch it with `argus experiments get` or MCP tool `get_experiment` and print its verdict sentences verbatim.

A `medication_dose.logged` notice is one confirmed dose recorded through Argus's own write path (`argus medication take`, MCP tool `take_medication_dose`); its payload carries the medication `concept_id` and the `taken_at` instant. Doses imported from Apple Health arrive inside `sync.landed` batches, never as per-dose events. React with counts and facts only — never score adherence or call an absent dose missed.

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

Skills version 1.7.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.


Question events `question.created`, `question.updated`, and
`question.deleted` contain only question_id and revision. Reread the
current question before acting; a deletion's revision follows its last stored
revision. These events signal question/evidence edits or state changes, not new
analysis findings. Paused and archived questions do not imply a clinical outcome.

Skills version 1.9.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.


Check-in events `check_in.created`, `check_in.edited` and `check_in.deleted`
contain only check_in_id and revision. Fetch the check-in before acting: an
edit can mean an explicit answer, correction, skip, snooze or reopen. It is
not a measurement or evidence of illness, adherence or continuous following.

Skills version 1.10.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.

Question lifecycle emits `question.created`, `question.updated`, and `question.deleted`. Saving an immutable answer emits `answer.saved`. Explicit following changes emit `following.started`, `following.paused`, `following.resumed`, `following.stopped`, and `following.expired`. A recorded receipt emits `update.recorded`; marking it reviewed emits `update.reviewed`. Read the referenced question or receipt before acting. These events do not establish causes, silently compute answers, or authorize a new write.

Skills version 1.10.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
