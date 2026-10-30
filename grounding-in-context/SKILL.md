---
name: grounding-in-context
version: 1.5.0
description: Use when an agent needs a compact, current grounding document across an Argus user's health ledger before answering a multi-domain question or planning around their recorded state.
license: MIT
---

# Ground an agent in Argus context

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Fetch the context once near the start of a multi-domain task:

```sh
argus context
argus context --date 2026-08-31 --output json
```

The MCP mirror is tool `get_context`. MCP clients that support resources can read `argus://context` as `text/markdown` without a tool call.

Inject `text` verbatim into the working prompt. It is deterministic server-rendered Markdown; its figures and relationship or attention sentences already follow Argus's unit and honesty rules. Use `sections` only when the consumer needs structured fields. Use `sources` to choose a specific read when more detail is needed.

The document also states, when the ledger carries the data: the last seven days of mood as words with a direction stated as fact ("average: pleasant", "more pleasant in the later days" — never a score), one progress line per running experiment ("Day 14 of 21 on L-theanine, 12 taken" — counts, never an interim verdict), and last night's deep, REM, and core figures beside the sleep summary. Medications and supplements are two sections, `medications` and `supplements`, never one. An omitted section means nothing was logged, not zero; treat every line as a recorded fact and preserve its wording.

Check `as_of` before relying on the document. State that the context is stale when the task happens materially later or after a known write. Fetch again after logging or editing data that affects the answer.

## New since the person last looked

`argus today` (tool `get_today`) carries `new_since` on the live day when something arrived while no Argus app was in view: `since_day` (the profile-timezone day the window opened) and up to four `items`, one per kind — `document` (a document recorded), `insights` (cards of Brief editions written while they were away), `experiment` (a running experiment's day advanced, or one ended), `context` (a life event or symptom episode added) — each with a server-written `label`, a `count` and `opens`. A kind you do not know still has a readable label. Your own writes while the person was away appear there: that is the point, so name them as yours when you mention them. The window starts at a day, never a time: an agent never learns when the person looked, and nothing an agent does reads or moves that mark (the app's own `/v1/today/seen` routes refuse agents with `403 apps_only`). Absent `new_since` means nothing new, or no window yet.

Do not fetch the whole context for a single figure. A question about one metric should use tool `get_metrics`, a day view should use tool `get_today`, and a single goal should use tool `list_goals`. The context states recorded facts; it does not diagnose, prescribe, or turn associations into causes.

Asked to export the person's record ("send me everything", "a CSV of my data"), point them to Account › Export in Argus on iPhone, iPad or Mac, which writes JSON, CSV or FHIR on their device, originals included if they choose. There is no export command or tool: the whole record in one act is the person's, in an app, and the export routes refuse agent keys (`403 apps_only`). For a single kind, read it through its own command or tool.

## Worked transcript

```text
$ argus context --output json
{"as_of":"2026-08-31T08:15:00.000Z","date":"2026-08-31","timezone":"Europe/Zagreb","text":"# Argus health context — 2026-08-31\n\n## Profile\n- timezone: Europe/Zagreb\n\n## Today so far\n- meals: 2 meals\n- energy: 1,140 kcal","sources":["getProfile","getToday"]}

Agent system prompt addition:
The following is a read-only health ledger context. Treat omissions as unknown, preserve its figures and sentences, and do not give medical advice.

# Argus health context — 2026-08-31
…
```

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
