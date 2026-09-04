---
name: argus-cli-usage
version: 1.5.1
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
- **Timezones**: samples record your local timezone; daily summaries use the profile timezone (`argus profile set --timezone Europe/Zagreb`).
- Discovery: `argus --help`, `argus <group> --help`; raw escape hatch `argus api GET /v1/... [--data '{...}']`; `argus docs` prints spec pointers.
- Diagnostics: `argus doctor` prints a read-only human report; `argus doctor --json` reports API/auth/profile readiness and why each higher-level capability is evaluated or unevaluated.
- Skills: `argus skills check` reports stale guidance with exit 1; use `argus skills update` or `npx skills add vburojevic/argus-skills` to refresh it.
- Environment safety: a project containing an `.argus-environment` marker selects that environment — its own host and its own credential profile, so keys never cross environments. Outside a marked project the CLI is production. `ARGUS_ENVIRONMENT=prod` is the explicit production override; `ARGUS_API_URL` may select only the serving environment's host. `ARGUS_DEBUG=api` logs request metadata to stderr.

## The two commands that matter daily

```sh
argus log "2 eggs and a slice of sourdough with butter"   # meal via inference
argus summary                                             # goals + nutrition + latest metrics
argus today                                               # whole day in one call
argus brief                                               # the day's Brief: headline, lead, rows — or an honest null reason
argus mood pleasant --label calm --about self_care        # State of Mind
argus medications                                         # Health-shared list
```

## The Brief

```sh
argus brief                       # the live day: morning until 17:00 local, evening after it
argus brief --date 2026-08-27     # a past day, written as a review
argus brief --week latest         # the note on the newest closed ISO week (Mon–Sun)
argus brief --week 2026-W35       # one named week
argus brief --cached              # answer from storage; never wait on generation
```

The response is `{ date, kind, week?, headline, lead, notes[], paragraph, reason?, generated_at, model }`. `kind` is `morning`, `evening`, `review` or `week`.

`headline` is one sentence of at most 14 words — the one thing to know, and the only line Argus's own teaser, widgets and watch show. Lead with it. `lead` is the 40–60 word paragraph behind it, never a second summary of the same sentence. Each of the up to eight `notes` is a ledger row: `section` (one of `night`, `readiness`, `yesterday`, `today`, `week`, `experiments`, `checklist`, `record`), `label` (≤ 3 words, e.g. `Resting HR`), `figure` (the figure exactly as Argus prints it — `7:46`, `115 / 140 g`, `Day 14 of 21` — or `null`), `text` (the sentence) and the `ref` (`{ kind, id }`) of the record it states or `null`. Quote a figure verbatim; never reformat or recompute one. `paragraph` is a deprecated mirror of `lead` — read `lead`.

A null `lead` carries a `reason`: `nothing_to_say`, `unavailable`, or `rate_limited` when the day's generation ceiling is reached; every one of them exits 0 and is reported as it stands, never replaced with commentary of your own.

## Chat

`argus chat "how did I sleep this week"` streams Argus's grounded answer. Use `--thread cht_…` to continue a thread and `--json` for raw event JSONL; `argus chat threads --json` lists recent threads. At a TTY, `argus chat` opens a REPL. Writes arrive as proposals and still require Apply or Dismiss in a product surface.

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.

A cold note takes about a minute to write. `GET /v1/brief` answers at once with what is on the record — the previous note, or `lead: null` with `reason: "generating"` — and finishes out of band; pass `wait=1` (the CLI does unless `--cached`) to block for the fresh note instead of reading again.

Skills version 1.5.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
