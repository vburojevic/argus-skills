---
name: recording-context
version: 1.3.0
description: Use when recording, correcting or reading reported life events, symptom episodes and dated recovery observations in Argus.
license: MIT
---

# Recording personal context

> **Source boundary:** Argus is the system of record for this workflow. Missing records mean unknown, never symptom-free. Do not infer a diagnosis, medication change or causal effect from an event or episode.

A life event is a named point or period: travel, a routine change, work, illness,
treatment, a milestone or other context. A symptom episode is the person's
reported experience, with an onset, optional resolution and dated observations.
An observation records severity (0–10), functional impact, a note, or a combination.
Zero means explicitly reported zero. Null means not reported. Omitted nullable
values default to null, including during a full-replacement update; preserve
existing values unless the person intends to clear them.

## Read first

Use `argus context events list --from 2026-09-01T00:00:00Z --to 2026-10-01T00:00:00Z`
or MCP tool `list_life_events`. Read one with `argus context events get lev_abc123`
or MCP tool `get_life_event`. Use `argus context episodes list` / MCP tool
`list_symptom_episodes` and `argus context episodes get sep_abc123` / MCP tool
`get_symptom_episode` for episodes. `argus context observations list sep_abc123`
or MCP tool `list_episode_observations` reads dated reports, newest first.

Lists carry `data` and `next_cursor`. Follow that cursor with identical window
filters to read beyond the first page. Life periods overlap [from,to); an episode
resolved exactly at from still intersects because its resolution observation
belongs to the episode. Points at to are excluded. Keep the recorded timezone;
never silently move a reported calendar date into the agent's timezone.

## Writes and retries

Only record what the person explicitly asks to save. Clarify missing dates and
meaningful ambiguities. Obtain permission before deleting records. Reports are
as stated, not measured samples or inferred diagnoses. Every create needs a
stable `client_ref`; reuse the exact payload and reference for transport retries.
A reused reference with different content is a conflict. A deleted record cannot
be resurrected by retrying its original create.

Create JSON files matching the contract and use `argus context events create --file event.json`
(MCP tool `create_life_event`) or `argus context episodes create --file episode.json`
(MCP tool `create_symptom_episode`). A point has `ends_at:null`; an open period
has `extent:"period"` and `ends_at:null`. An episode has `onset_at` and nullable
`resolved_at`. All instants include an offset and all records include an IANA
`timezone`. Optional `metrics` must use registry names; `samples` must contain
owned sample pairs `{id,start}` returned by Argus, never guessed ids.

For edits, fetch the latest record, preserve all fields the person did not
change, strip read-only identity fields and supply its `expected_revision`.
Use `argus context events update lev_abc123 --file event-edit.json` / MCP tool
`update_life_event` or `argus context episodes update sep_abc123 --file episode-edit.json`
/ MCP tool `update_symptom_episode`. A stale revision returns 409: reread and
reconcile the person's intended edit, never overwrite newer content blindly.

`argus context observations create sep_abc123 --file report.json` / MCP tool
`create_episode_observation` needs the current episode `expected_revision`.
Every report mutation advances the parent revision. Reports must fall within
the episode, including its exact resolution instant. Dates cannot be edited
to exclude reports; correct the report or episode intentionally first.

`argus context observations update sep_abc123 eob_abc123 --file report-edit.json`
/ MCP tool `update_episode_observation` needs both the report's
`expected_revision` and the current `episode_revision`.

For deletion use `argus context events delete lev_abc123 --revision 2 --confirm`
/ MCP tool `delete_life_event`, or `argus context episodes delete sep_abc123 --revision 3 --confirm`
/ MCP tool `delete_symptom_episode`. Deleting an episode also deletes its
observations. `argus context observations delete sep_abc123 eob_abc123 --revision 1 --episode-revision 3 --confirm`
/ MCP tool `delete_episode_observation` deletes one report. MCP deletion requires
the same current revisions and explicit authorization from the person.

## Worked example

Person: Record that my headache began at 9 this morning, Paris time. It is still going.
Use the person's actual date; this fixture assumes September 8, 2026.

```json
{"client_ref":"headache-2026-09-08-person-request","title":"Headache","onset_at":"2026-09-08T09:00:00+02:00","resolved_at":null,"timezone":"Europe/Paris","note":null,"metrics":[],"samples":[]}
```

```text
$ argus context episodes create --file episode.json
{"id":"sep_abc123","revision":1,"observation_count":0,...}
Person: At noon it was a 4, and I had to stop working. Save that too.
```

Create report.json with `observed_at:"2026-09-08T12:00:00+02:00"`, `severity:4`,
`impact:"limits_activity"`, `note:"Had to stop working."`, a new stable
`client_ref`, and `expected_revision:1`. Ask about the impact choice if the
person's meaning is unclear. After `argus context observations create sep_abc123 --file report.json`,
show the stored report and reread the parent before a later edit. The abbreviated
outputs above are synthetic examples, not real receipts.

Skills version 1.1.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.

In-app chat can prepare life events, episodes and observations as proposals.
Their receipt shows the reported details before Save. A proposal is not a
saved record; an episode observation pins the parent revision and may need a
new proposal if another device changes it. External agents continue to use
the explicit write operations above after the person authorizes the write.

Skills version 1.1.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.

Read a linked observation with `argus metrics sample smp_abc123 --start 2026-09-08T09:00:00Z`
or MCP tool `get_sample` with `id` and the exact `start` from the link. This is
an original measurement receipt, including provenance; daily totals may
supersede it. Do not replace a deleted/unavailable link with a nearby reading.

Skills version 1.1.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.


## Read a combined timeline

Use `argus context timeline --from 2026-09-01T00:00:00Z --to 2026-09-08T00:00:00Z --metric resting_heart_rate`
or MCP tool `get_personal_timeline`. The window is required and limited to
93 days. Omit metric for context only; choosing a metric adds original samples
without hiding unlinked life events or episodes. Readings may be superseded in
daily totals; do not sum the raw timeline to reconstruct those totals.

The result carries `data` and `next_cursor`. Preserve from/to/metric while
paginating. `at` positions an entry within this window: an earlier period
continuing into it sits at from, while its original onset remains in its
record. Observations retain the episode title and original zero/unknown fields.
Follow the embedded record or exact sample reference to inspect details.
Dates alongside one another provide context, not a causal conclusion.

Skills version 1.3.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
