---
name: managing-recipes
version: 1.0.2
description: Use when listing, reading, creating, promoting, editing, deleting, or logging reusable Argus Recipes and their fractional Portions.
license: MIT
---

# Manage recipes with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

A Recipe is a per-user reusable composed meal. Its lines and nutrient totals describe one Portion. Logging it creates a Meal from the recipe's current snapshots; editing the Recipe later never rewrites meals already logged.

## List and inspect

```sh
argus recipes list --output json
argus recipes show rcp_abc123 --output json
```

`argus recipes list` returns name, per-portion energy and protein, and `last_logged_at`. Use `argus recipes show` before editing lines. MCP tools `list_recipes` and `get_recipe` return the same stored facts.

## Create from structured lines

```sh
argus recipes create "Batch plate" --lines '[{"text":"Prepared grains","grams":180,"basis":"per_100g","nutrients":{"dietary_energy_consumed":150,"dietary_protein":0.05}}]'
```

MCP tool `save_recipe` accepts either `{name, lines}` or promotion from a meal. Lines use the same two arms as meal ingestion: a resolved `{food_id, qty, unit?, grams?}` line or an inline `{text, qty, unit?, grams?, nutrients?, basis?, gtin?}` line. Nutrition is numeric and canonical; unknown nutrient keys are reported as ignored. A Recipe stores the scaled absolute snapshot for one Portion.

## Promote an existing meal

```sh
argus meals get meal_abc123 --output json
argus recipes save --from-meal meal_abc123 --name "Batch plate"
```

Promotion copies the meal's lines at snapshot fidelity. Omit `--name` to use the meal name. It does not modify or delete the source Meal.

## Edit and delete

```sh
argus recipes edit rcp_abc123 --name "Batch bowl"
argus recipes edit rcp_abc123 --lines '[{"text":"Prepared serving","nutrients":{"dietary_energy_consumed":520}}]'
argus recipes delete rcp_abc123 --yes
```

MCP tools `update_recipe` and `delete_recipe` mirror these operations. A line edit replaces the complete recipe-line snapshot for future logs only. Read the Recipe first, then echo unchanged stored lines with their description, food_id, qty, unit, grams, source, and nutrients fields; this preserves grounded lines while another line is edited. Deleting a Recipe leaves past Meals and their items intact; those meals simply lose the recipe link.

## Log Portions

```sh
argus recipes log rcp_abc123
argus recipes log rcp_abc123 --portions 0.5 --at 2026-08-31T12:30:00+02:00 --timezone Europe/Zagreb --client-ref batch-log-1
```

MCP tool `log_recipe` accepts `recipe_id`, `at`, `timezone`, fractional `portions`, and optional `client_ref`. Quantity, grams, and nutrients all scale by Portions. `client_ref` makes retries return the existing Meal instead of logging twice.

Recipe-backed Meals are deliberately excluded from `argus usuals` and MCP tool `list_meal_usuals`. The Recipe is already the named quick-log representation; deriving a usual from every recipe log would create duplicate choices. `repeat_of` remains the separate exact-replay primitive for ordinary Meals.

## Worked transcript

```text
$ argus recipes save --from-meal meal_abc123 --name "Batch plate" --output json
{"id":"rcp_abc123","name":"Batch plate","totals":{"dietary_energy_consumed":640,"dietary_protein":32}}
$ argus recipes log rcp_abc123 --portions 0.5 --client-ref batch-half-1 --output json
{"id":"meal_def456","name":"Batch plate","totals":{"dietary_energy_consumed":320,"dietary_protein":16}}

Half a Portion was logged from the current recipe snapshot. The resulting Meal is on the ledger; the Recipe remains reusable.
```

Skills version 1.0.2 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
