---
name: managing-access
version: 1.2.0
description: Use when authenticating the argus CLI, creating/revoking Argus API keys, configuring non-interactive/CI access, wiring the Argus MCP server into an agent, or diagnosing 401s.
license: MIT
---

# Managing Argus access

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## Authenticating the CLI

```sh
argus auth status                               # what identity (if any) is active
argus auth login                                # interactive: device flow in the browser
argus auth login --with-key                     # non-interactive: key via stdin, verified then stored
argus auth logout
```

Agents/CI: prefer an environment key — no stored state, wins over stored credentials. A project directory may carry an `.argus-environment` marker naming a non-production environment; the CLI then selects that environment's host and keeps its credentials in a separate profile, so a key minted for one environment is never sent to another. Outside a marked project the CLI is production. From inside one, production requires the explicit `ARGUS_ENVIRONMENT=prod` override.

Read `argus account get` for the authenticated account, plan, and limits. Read `argus profile get` for timezone, characteristics, and display units; neither command returns health samples.

## API keys

```sh
argus keys create --name "claude-laptop"        # secret shown ONCE — store it immediately
argus keys list
argus keys revoke key_abc123 --confirm
```

Keys are `argus_`-prefixed, hashed at rest, rate-limited per key. `keys list` and MCP tool `list_api_keys` include `device_id` and `device_name` when a key belongs to an iOS device; agent/CLI keys have both fields null. One key per agent/machine so revocation is surgical.

## MCP access

The Argus MCP server lives at `<ARGUS_API_URL>/mcp`. It publishes `argus://brief/today`, the manifest at `argus://skills`, and each current skill at `argus://skills/<name>`. A null Brief has empty text plus a machine-readable reason. Two auth modes:
- **X-Api-Key header** — simplest for agent configs.
- **OAuth** — MCP-native clients discover everything via `/.well-known/oauth-protected-resource` and log in interactively.

`argus agent setup` installs the Argus skills + a hosted-dev MCP config into detected agents (Claude Code, Codex, Cursor, Gemini CLI); `--json` prints the plan without writing. It selects production only with `ARGUS_ENVIRONMENT=prod` and the exact production origin.

Use `argus skills list`, `argus skills install`, `argus skills update`, and `argus skills check` to compare or install the served manifest. `check` exits 0 when current and 1 when stale; `argus doctor` prints the same update command when installed skills trail the server.

Run `argus doctor --json` after setup. It is read-only and distinguishes invalid auth (exit 2), unreachable API (exit 1), and authenticated capabilities that remain unevaluated (exit 0 with reasons).

## Diagnosing 401s (exit code 2)

1. `argus auth status` — is anything configured?
2. Key revoked/expired? `argus keys list` from a session-authed context, or create a fresh key.
3. Wrong server? Check `ARGUS_API_URL`.

Skills version 1.2.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
