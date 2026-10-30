---
name: argus-cli-usage
version: 1.15.0
description: Use when driving the argus CLI (personal health/nutrition tracker) — output contract, auth resolution, exit codes, confirmation semantics, pagination, and discovery conventions every argus command follows.
license: MIT
---

# argus CLI conventions

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

`argus` is the agent-first CLI for Argus, a personal health ledger: meals, nutrition, workouts, sleep, goals, and every HealthKit metric type.

## Output contract

- **stdout carries exactly one JSON payload** (compact) in non-TTY contexts — parse it directly. Humans at a TTY get a rendered view (a table for lists, a laid-out report for `brief` and `coverage`); force either with `--output json|table`.
- Everything auxiliary (prompts, cursor hints, progress, debug) goes to **stderr**.
- Errors are one RFC 9457 problem JSON line on stderr: `{"title","status","code","detail","remediation"}` — **read `remediation`; it says what to do next.**

## Exit codes (frozen)

| code | meaning | react |
|---|---|---|
| 0 | success | parse stdout |
| 1 | general/network | retry or report |
| 2 | auth (401/invalid key) | run `argus auth status`, then login |
| 3 | validation (400/422) | fix arguments per the problem body |
| 4 | confirmation required | re-run with the exact `--confirm` command printed on stderr |

Destructive commands (`delete`, `revoke`, `clear`, `delete-sample`) always exit 4 first; nothing is destroyed until `--confirm`.

## Auth resolution order

1. The serving environment's key variable — `ARGUS_API_KEY` for production, and a per-environment variable elsewhere (best for agents/CI)
2. The serving environment's stored credentials under `~/.config/argus/` — `credentials.json` for production, a separate file per environment (written by `argus auth login`)

Check with `argus auth status` / `argus auth whoami`. Interactive humans: `argus auth login` (device flow). Non-interactive: `argus auth login --with-key` (reads the key from stdin) or export the environment-specific key above. Never copy a key between profiles.

## Conventions

- **Idempotency is automatic**: every mutating command sends an `Idempotency-Key`; safe to retry on network failure.
- **Pagination**: paged list responses carry `next_cursor`; pass `--cursor <value>` for one page or `--paginate` to follow every cursor and emit one JSON array. Current cursor-paged commands: `meals list`, `workouts list`, and raw `metrics get`.
- **Units**: writes accept `--unit` (lb, kg, ml, °F…) and convert to HealthKit-canonical units server-side; reads return canonical units.
- **Timezones**: samples record your local timezone; daily summaries use the profile timezone (`argus profile set --timezone Europe/Zagreb --confirm-timezone`). Use `--confirm-timezone` only after the person confirms that value; a timezone string or device setting is not confirmation.
- Discovery: `argus --help`, `argus <group> --help`; raw escape hatch `argus api GET /v1/... [--data '{...}']`; `argus docs` prints spec pointers.
- Diagnostics: `argus doctor` prints a read-only human report; `argus doctor --json` reports API/auth/profile readiness and why each higher-level capability is evaluated or unevaluated. A `database_unavailable` problem (HTTP 503, exit 1) means the API answered but its database is not ready; retry after readiness recovers. `api_unreachable` identifies a transport/discovery failure instead.
- Skills: `argus skills check` reports stale guidance with exit 1; use `argus skills update` or `npx skills add vburojevic/argus-skills` to refresh it.
- Environment safety: a project containing an `.argus-environment` marker selects that environment — its own host and its own credential profile, so keys never cross environments. Outside a marked project the CLI is production. `ARGUS_ENVIRONMENT=prod` is the explicit production override; `ARGUS_API_URL` may select only the serving environment's host. `ARGUS_DEBUG=api` logs request metadata to stderr.

## The two commands that matter daily

```sh
argus log "2 eggs and a slice of sourdough with butter"   # meal via inference
argus summary                                             # goals + nutrition + latest metrics
argus today                                               # whole day in one call, with What changed and New since
argus today --basis yesterday                             # What changed against yesterday (usual | yesterday | last-week)
argus brief                                               # today's Brief: the opening, up to four cards and an offer, or pending / quiet
argus mood pleasant --label calm --about self_care        # State of Mind
argus medications                                         # Health-shared list
```

## The Brief

```sh
argus brief                       # today's daily Brief — written once, when last night lands (else 10:00), then frozen
argus brief --date 2026-08-27     # a past day: its review, beside that day's own insight
argus brief --week latest         # the note on the newest settled ISO week (Mon–Sun)
argus brief --week 2026-W35       # one named week
argus brief --cached              # answer from storage; never wait on a review or week note
argus brief timeline              # past editions' first cards, newest first; --before / --limit / --paginate
```

The response is `{ date, kind, week?, status?, opening?, points?, insights?, offer?, feedback?, insight?, also_new?, chart?, dismissed_at?, headline, lead, notes[], paragraph, reason?, generated_at, model }`. `kind` is `morning` (today's daily Brief), `review` (a past day) or `week`; `evening` is retired and never produced. At a TTY `argus brief` prints the opening, then its key points (`points`, up to three, each `· text`), then one row per card (`Recovery · 3 measures  —  headline`), then the offer; `--json` passes the response through unchanged.

Today's `status` is `pending` (not written yet), `written` or `quiet` (nothing moved outside the person's usual). A written day carries `opening` — `{ sentence, lead }`, the sentence at most 16 words, the lead 30–60 — and `insights`: one to four cards in the server's rank, the one the opening leads with first, each `{ id, kind, metric, label, headline, sentence, figure, evidence?, source, opens }`. A card's `kind` is one of `recovery`, `changed`, `record`, `streak`, `pattern`, `question`, `experiment` (`goals-and-progress` says what fires each); `evidence` is drawn from the ledger, `source` names the window and the devices, `opens` where the figure came from. A quiet day carries the fixed opening "A quiet day: nothing moved outside your usual overnight." with a null lead. Lead with the opening's sentence, then the cards' headlines; quote a figure verbatim, never reformat or recompute one. `offer` is an experiment the person may start, built by the server from their own pattern — `experiment` is the `argus experiments start` body, and only the person starts it: show it, never start it on your own. `feedback` (`useful` or `not_for_me`) is the person's own answer; read it, never write it — feedback and *Not now* are the apps' alone. Every new key is absent while empty. `dismissed_at` is the person's own reading state on any device. `insight`, `also_new` and `chart` are the first card and the rest in the older shape (`kind` one of `outlier`, `streak`, `record`, `trend`, `question`, `experiment`), and `headline`, `lead` and `notes` mirror them again, for released readers; a review or a week note keeps `headline`, a 40–60 word `lead` and up to eight `notes` rows (`section`, `label`, `figure`, `text`, `ref`) as its own and carries no cards. `paragraph` is a deprecated mirror of `lead`. This CLI sends `x-argus-brief` so it receives every card kind it knows.

A null `lead` carries a `reason`: `nothing_to_say` (pending or quiet, or nothing stored), `unavailable`, `rate_limited` when the day's cap of four written editions is spent, or `generating` while a review or week note is being written; every one of them exits 0 and is reported as it stands, never replaced with commentary of your own.

`argus brief timeline` prints `{ items: [{ date, kind, metric, label, headline, sentence, figure, dismissed, count? }], next_before }` — each item is the edition's first card, `count` its cards when more than one; pass `next_before` back as `--before` for the next page, or `--paginate` to collect them all. Quiet days are absent by design.

## Chat

`argus chat "how did I sleep this week"` streams Argus's grounded answer. Use `--thread cht_…` to continue a thread and `--json` for raw event JSONL; `argus chat threads --json` lists recent threads. Rename with `argus chat threads rename cht_… --title "Sleep this week"` (`updateChatThread`; MCP `rename_chat_thread`); the chosen title is final and never overwritten by the assistant. Recent threads include up to 160 characters of their latest message as `preview`. Use `--question inv_… --revision 2` to start from an exact saved question revision; this is mutually exclusive with `--thread`. At a TTY, `argus chat` opens a REPL. Writes arrive as proposals and still require Apply or Dismiss in a product surface.

With `--json`, each line is one event: `user_message`, `assistant_started`, `status` (what the turn reads before its first tool runs, e.g. `Reading steps · today`), `tool_started`/`tool_completed` (labels carry the read's window, e.g. `Read sleep · 18–24 Sep`), `text_delta`, `part`, `thread_titled`, `assistant_completed` and `error`. Reads in one step run together. A `head` part leads an answer from the record: `{ text, figure?, unit?, metric? }` — one sentence and, when one figure carries the answer, that figure as Argus prints it (`6:12` `h`, `8,412` `steps`); the server refuses any head figure the turn did not read, so quote it verbatim and never recompute it. A `provenance` part closes an answer that read the record: `{ summary, records?, at, reads: [{ label, kind?, records?, subject, scope?, metric?, outcome }] }`, built by the server from the reads, never by the model — cite it for where an answer came from. A read's `kind` is the read's own name (`get_sleep`, `list_lab_tests`); `records: true` marks a read whose output carried Records content (a lab result or lab goal, a medication or supplement, a document, a Brief's private card); on the provenance itself it marks an answer whose context carried Records (its thread's Brief or Question, or an earlier answer in the thread), which may then read nothing and name "the Brief", "this question" or "this conversation". The apps offer no Share on such an answer, so treat it as the person's medical record and share it only when they ask. Pass through event and part types you do not know. A stopped answer is stored with `response_status` `interrupted` (or `limit`, with `reads`); send `Keep going from where you stopped.` on the same thread to continue it. `argus chat retry --thread cht_… [--message chm_…]` answers the thread's last question again in place of its answer (no second copy of the question; the stream opens at `assistant_started`; refused with 409 when that answer's proposal was applied or `--message` is not the latest answer). `argus chat feedback chm_… --rating up|down|none [--note "…"]` records the person's verdict on an answer — only when the person gives it; it is read back as the message's `feedback`.


Today's Brief is only ever read. A review or a week note that has settled but was never written is written on the read that asks for it: `GET /v1/brief` returns `lead: null` with `reason: "generating"` while that work runs. Pass `wait=1` (the CLI does unless `--cached`) to wait within a 90-second request window; it can still return `generating`, so read again later without inventing a replacement note. Preparation and transport failures remain errors; they are not proof that a note is being written. `mode=cached` only reads storage.


## Saved questions and analyses

`argus questions list --state active` finds saved questions. Read a question
before using `questions compare inv_… --file comparison.json` or
`questions curve inv_… --file curve.json`; each command saves an explicitly
reviewed analysis definition with expected_revision and stable client_ref.
`questions analyses` / `questions analysis` read comparisons;
`questions curves` / `questions curve-result` read response curves.
The lists accept the question ID and `--cursor`; individual reads take the
run ID. Reads preserve the saved snapshot and never recalculate. Follow the
keeping-questions skill for support gates, timing semantics, retry conflicts
and interpretation limits. A descriptive curve is not a causal conclusion or an
optimal dose/time; its omission range is not a confidence interval.

Skills version 1.15.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
