---
name: answering-check-ins
version: 1.1.2
description: Use when saving one useful missing-context question for an Argus question, life event or episode, recording the person's answer, or managing its skip/snooze and correction history.
license: MIT
---

# Ask a purposeful check-in

> **Source boundary:** Argus is the system of record for this workflow. Missing context is a gap,
> not permission to infer a symptom, missed dose, non-wear or lifestyle event.

A conversational reply is not a saved check-in answer.
Use Review check-in in the apps for explicit answer review and saving. In Ask
Argus, propose_check_in and propose_check_in_answer prepare confirmation cards
with the exact question, context revision, availability and reported answer.
Save check-in / Save answer is the write; a pending card records nothing.
Corrections show the earlier answer and preserve its history. A stale review
must be rebuilt from the current record rather than silently advancing its
revision. View check-in opens the saved record after confirmation.

Read `argus check-ins list` / MCP tool `list_check_ins` first. Available questions
are returned by default; use state `all` or `answered` for records/history.
Optional `--target-kind question --target-id inv_abc123` scopes a list;
MCP uses target_kind and target_id. Keep filters when passing next_cursor as
cursor. Present one relevant question with its reason, allowing a free answer,
skip or snooze. No notification, badge, medical following or automatic answer
is authorized by saving a question.

Save only a question the person wants to keep, using `argus check-ins create
--file check-in.json` / MCP tool `create_check_in`. Read the linked record first.
The JSON has question (240 characters), reason (500), optional distinct choices
(up to five, 80 characters each), target `{kind,id,expected_revision}`, a stable
client_ref, and available_at / expires_at instants. Target kind is question,
life_event or symptom_episode. The window is at most 30 days, expires in the
future and starts no later than 30 days from now. The server pins the exact
owned label, revision and timezone. It permits one unanswered question for that
record/revision, with at most 20 unexpired open check-ins. Avoid leading choices;
include uncertainty where appropriate, and preserve a written-answer path.

Read `argus check-ins get cki_abc123` / MCP tool `get_check_in` before changing
it. Use `argus check-ins update cki_abc123 --file command.json` / MCP tool
`update_check_in` (MCP input is `{id,command}`). A command carries action,
expected_revision and a new stable client_ref for this distinct action:

```json
{
  "action": "answer",
  "expected_revision": 1,
  "client_ref": "answer-for-this-confirmation",
  "answer": { "choice": null, "text": "My evening routine was unchanged." }
}
```

Only send an answer the person explicitly reported and confirmed. choice must
exactly match an offered choice; text may stand alone or qualify it. Empty
answers fail. The text "0" is a real answer. This saves reported context with a
recorded_at timestamp; it never projects a measurement or modifies the parent.
For a correction, answer again against the latest revision. Original wording
and earlier answers remain in the history even if the parent question changes.

Other actions: `skip` records no answer; `snooze` requires until strictly after
now and before expiry; `reopen` restores a skipped question only while its
original context/window is valid and no other question for that record is open.
Snoozed and scheduled questions cannot be answered until available. A pending
question becomes context_changed when its linked revision changes, or paused
when its question stops being active. Read status and prepare a fresh
question instead of reinterpreting the old one. Expired questions do not renew
themselves. An answered question cannot be turned into a dismissal.

The same client_ref and command safely retries without a second write. Changed
content conflicts. A revision conflict requires rereading and reconciling, not
blindly repeating with a new revision. Read immutable snapshots with `argus
check-ins history cki_abc123` / MCP tool `list_check_in_revisions`; pass its
next_cursor as before (`--before` in CLI). Each page has up to 20 revisions.
Historical state describes that write, not current availability.

Delete only with explicit intent: `argus check-ins delete cki_abc123 --revision
3 --confirm` / MCP tool `delete_check_in`. This removes the prompt, answer and
revision history; linked measurements and context remain. Deleting a parent
also removes its check-ins. Retry tombstones keep no question or answer text
and prevent a deleted record from reappearing through a retry.

`check_in.created`, `check_in.edited` and `check_in.deleted` carry only check_in_id
and revision. Fetch the record before acting; these are record changes, not
new medical findings. An edit can be a dismissal or snooze rather than an answer.

Skills version 1.1.2 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
