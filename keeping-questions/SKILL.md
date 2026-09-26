---
name: keeping-questions
version: 2.0.1
description: Keep personal health questions, review descriptive answers and explicitly follow recorded evidence changes.
license: MIT
---

# Keeping questions

> **Source boundary:** Argus is the system of record for this workflow. A Question records an interest, not a finding. An Answer describes captured evidence, never a cause, diagnosis, optimal amount or treatment recommendation.

Read `argus questions list --state active` / MCP tool `list_questions` before saving a duplicate. Read one with `argus questions get inv_abc123` / `get_question`. Lists carry optional following, pending_check_in and latest_answer summaries. Preserve filters and next_cursor while paging; omitted state includes paused and archived questions.

Save only the person's explicit question: `argus questions create "Does walking relate to sleep?" --purpose "Understand my recorded days" --metric step_count,sleep_duration` / `create_question`. The contract body has title, state, timezone and evidence containing metrics, exact samples `{id,start}`, life_events and episodes. Use owned records read from tools. A stable client_ref deduplicates retries; different content with the same reference conflicts.

Edit with `argus questions update inv_abc123 --title "Walking and next-night sleep"` / `update_question`. The CLI reads the current revision and preserves unchanged fields; MCP supplies the full fields and expected_revision. On 409, reread and reconcile. Active, paused and archived are question states. Editing does not recompute answers or silently restart Following. Delete only on explicit intent: `argus questions delete inv_abc123 --confirm` / `delete_question`. Linked ledger records remain; the retry tombstone prevents recreation.

Read immutable wording with `argus questions revisions inv_abc123` / `list_question_revisions`, and `argus questions revision inv_abc123 2` / `get_question_revision`. Read the chronological feed with `argus questions history inv_abc123` / `get_question_history`. It contains answer_saved, evidence_changed, baseline, check_in_answered, question_edited and following_changed entries, newest first. Follow its cursor with the same question. Temporal adjacency never establishes provenance.

## Review an Answer

Read `argus questions answers inv_abc123` / `list_answers`, then `argus questions get-answer ivr_abc123` / `get_answer`. The same Answer shape carries kind, pinned question_revision, definition, result, quality, provenance, verdict and matched_days. Its recorded evidence remains immutable when later data arrives. Preserve insufficient, imbalanced and partial support before discussing an estimate. Unknown source quality or sync completion is unknown, not proof of a physical device change.

For a new answer, read the question and measured prefills first: `argus questions defaults inv_abc123 --kind compare_days --exposure step_count --outcome sleep_duration --from 2025-01-01 --to 2025-04-01` / `get_answer_defaults`. Percentiles include recorded zero and exclude missing closed days. Defaults are editable starting points, not selected thresholds or findings. Review the sentence and the full definition with the person before `argus questions answer inv_abc123 --kind compare_days --definition @definition.json --client-ref walking-sleep-1` / `create_answer`.

The file contains a definition, for example:

```json
{"kind":"compare_days","from":"2025-01-01","to":"2025-04-01","exposure":{"metric":"step_count","low_max":4000,"high_min":8000},"outcome":{"metric":"sleep_duration","lag_days":1},"same_weekday":true,"match":[],"source_policy":"same_sources"}
```

This is an example, not a recommended threshold. The window is profile-local, from inclusive and to exclusive. Ask one focused question about a material ambiguity before saving. Missing lagged boundary days remain excluded. Choose matching and exclusion rules before inspecting outcomes; never search definitions for a favorable answer.

- `compare_days`: declared low and high exposure days are matched without reuse using declared preceding factors, date distance, optional weekday and source sets. Report pairs, weeks, exclusions and balance. The middle 80 percent of pairs and leaving weeks out are descriptive sensitivity, never confidence intervals or significance. Pair counts are not independent observations.
- `by_amount`: exposure is `{ "metric":"step_count", "mode":"amount", "clock_origin_hour":0 }`. Preserve supported bands, recording groups and gaps. A null estimate is unavailable, not zero. Do not join unsupported segments.
- `by_timing`: exposure uses first_recorded_time or last_recorded_time and an explicit clock_origin_hour. Positive record starts are recorded timing, not verified consumption. Rotating the clock axis does not move calendar dates. Nearby P10–P90 spread and leaving fourteen-day blocks out are descriptive, never causal effects, optima or treatment advice.

MCP `create_answer` supplies expected_revision and the reviewed definition. Keep client_ref stable across retries. Supply preceding_answer_id or preceding_update_id only for explicit provenance; never infer them from nearby timestamps. A changed revision or profile timezone requires fresh review.

`argus questions freshness ivr_abc123` reads the latest stored check through `get_answer`. `argus questions check-freshness ivr_abc123 --client-ref check-1` / `check_answer_freshness` explicitly checks the original window without refitting. Unchecked does not mean changed; current evidence does not rewrite the saved answer. To save a new answer, review another definition explicitly.

## Following and updates

Read `argus questions following inv_abc123` / `get_following` and the profile before configuring a rule. Use `argus questions follow inv_abc123 --metric step_count,sleep_duration --window 30 --changed-days 3 --stops 2026-10-01T00:00:00Z` / `set_following` only after explicit review. The MCP request includes expected_question_revision, expected_revision, expected_timezone, rule, expires_at and stable client_ref. Suggested measurements and thresholds are reviewable choices, not medical targets.

Following has active, paused, stopped and expired states. Use `argus questions pause inv_abc123`, `argus questions resume inv_abc123`, or `argus questions stop inv_abc123` / `update_following`, preserving the current rule revision. An explicit check uses `argus questions refresh inv_abc123` / `refresh_following`. Replacement starts a quiet baseline and preserves historical receipts. A saved rule does not prove its first check has completed; read status again. No notification, automatic new answer or treatment action is implied.

Read `argus questions updates inv_abc123` / `list_updates`, then `argus questions update imu_abc123` / `get_update`. Evidence changed reports changed recorded dates, including late corrections, removals and source changes. It does not establish a health change or update an Answer. Keep zero and no record distinct. Read the exact changes before describing dates. A baseline is quiet. Historical receipts remain valid when Following stops or expires.

`argus questions update imu_abc123 --mark-reviewed` / `update_following` with action mark_reviewed marks the exact receipt after reading it and the current following revision. Chat replies do not mark updates reviewed.

## Ask Argus

`argus chat --question inv_abc123 --revision 2 "Explain this question"` pins the selected wording. Chat tools propose_question, propose_answer and propose_following prepare visible reviews; only the person's explicit save applies them. The applied receipt links to the saved Question, Answer or update. Stale proposals require fresh reads and review, never silent revision changes. A useful missing circumstance can be a neutral check-in; silence and dismissal never count as an answer.

Skills version 2.0.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
