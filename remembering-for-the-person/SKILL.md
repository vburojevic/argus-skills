---
name: remembering-for-the-person
version: 1.0.0
description: Use when listing, adding, or deleting the visible facts Argus remembers across Ask Argus threads and agent surfaces.
license: MIT
---

# Remembering for the person

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Argus memory is explicit: nothing is remembered silently, and every memory is readable and deletable by the person. Before adding anything, list what is already there with `argus memories list` or MCP tool `list_memories`.

## When to remember

Add one short, durable, health-relevant fact only when relaying what the person stated about themselves. Keep it plain prose, first person about the person, and at most 280 characters. Examples: “Allergic to shellfish” or “Cutting until October.” Use `argus memories add "Allergic to shellfish"` or MCP tool `add_memory`.

Do not add:

- an inference, diagnosis, or conclusion drawn from ledger data;
- a sensitive category the person did not volunteer;
- a transient state such as today’s soreness or one poor night;
- a fact about another person;
- a duplicate of a visible memory.

Argus keeps at most 50 active memories and never silently evicts one. If the cap is reached, show the person the list and ask which exact memory to remove.

## Removing a memory

Inspect the list, identify the exact `mem_…` id, then run `argus memories delete mem_abc123` and re-run it with `--confirm` after the CLI exits 4. MCP tool `delete_memory` is destructive and takes the exact id. Editing is delete then recreate in v1.

## Worked transcript

```text
Person: I’m allergic to shellfish. Please remember that.
Agent: I can save exactly that stated fact.
$ argus memories add "Allergic to shellfish" --output json
{"id":"mem_abc123","text":"Allergic to shellfish","source":"api"}
$ argus memories list --output json
[{"id":"mem_abc123","text":"Allergic to shellfish","source":"api"}]

The fact is now visible on Account and available to new Ask Argus threads. It was not inferred from meals or symptoms.
```

Skills version 1.0.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
