---
name: logging-supplements
version: 1.4.0
description: Use when creating, editing, scheduling, taking, listing, or undoing supplements in Argus, including nutrient contributions, non-nutrient ingredients, starter prefills, regimens, checklist status, and nutrition coverage.
license: MIT
---

# Log supplements with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Supplements are user-owned product definitions. A serving is described in two ways, and every supplement needs at least one:

- **Contributions** say what a `dietary_*` metric names. Taking the supplement projects these into nutrition, so they reach coverage, goals, and correlations.
- **Ingredients** are `{name, amount, unit}` entries for what no metric names, such as creatine, melatonin, CFU, and herbal extracts. Argus records them as stated in the product's own unit. They are NEVER projected as a sample and never appear in coverage.

Form describes the physical object: a probiotic is a capsule and creatine is a powder. Never file a product as `other` merely because it carries no nutrient. Packaging is the serving, not the form: a sachet product is `powder` with serving_unit `sachet`, and taking one sachet is one serving.

## Define from a label

```sh
argus supplement starter
argus supplement add "Vitamin D3" --form softgel --serving-size 1 --serving-unit softgel --contributions '[{"metric":"dietary_vitamin_d","value":50,"unit":"µg"}]'
argus supplement add "Probiotic" --form capsule --serving-size 1 --serving-unit capsule --ingredients '[{"name":"L. rhamnosus","amount":10,"unit":"billion CFU"}]'
argus supplement add "Probiotic · 450 billion CFU" --form powder --serving-size 1 --serving-unit sachet --ingredients '[{"name":"Live bacteria blend","amount":450,"unit":"billion CFU"}]'
argus supplement list
argus supplement get supplement_abc
argus supplement edit supplement_abc --data '{"serving_size":2}'
```

Use starter rows only as editable prefills. Confirm the product label; never invent a dosage, recommend a product, or silently choose units. State an ingredient's unit exactly as the label prints it — Argus does not convert it.

## Schedule, take, and undo

```sh
argus regimens create --data '{"supplement_id":"supplement_abc","servings":1,"schedule":{"kind":"daily"},"slot":"morning","active":true}'
argus supplement checklist --date 2026-08-27
argus take "Vitamin D3"                       # fuzzy name hero
argus supplement take supplement_abc --regimen-id regimen_abc
argus supplement intakes --limit 50
argus supplement undo supplementIntake_abc    # exits 4; re-run with --confirm
```

As-needed regimens are never pending. A pending checklist item is a schedule state, not a medical obligation.

`argus today` (tool `get_today`) carries the day's checklist as `supplements`, so a day view needs no second call; `argus supplement checklist --date` is for a date on its own. Chat can propose an intake (`take_supplement`); the person applies it in a product surface.

After an intake, `argus coverage` includes its nutrient contributions, and `argus coverage dietary_vitamin_d` names the intake as a contributor beside the day's meals. Undoing the intake removes those supplement projections. Coverage states the reference intakes it read from — quote that basis with any target; the `reading-coverage` skill has the sentence, the zero law, and how a limit inverts.

The MCP mirror is complete: tools `list_supplements`, `list_starter_supplements`, `get_supplement`, `create_supplement`, `update_supplement`, `archive_supplement`, `list_regimens`, `create_regimen`, `update_regimen`, `delete_regimen`, `take_supplement`, `get_supplement_checklist`, `list_supplement_intakes`, and `delete_supplement_intake` follow the same confirmation and honesty rules.

## Worked transcript

```text
$ argus take "Vitamin D3" --output json
{"id":"supplementIntake_abc","supplement_id":"supplement_abc","regimen_id":"regimen_abc","servings":1,"source":"manual","logged_via":"cli"}
$ argus coverage --output json
{"date":"2026-08-27","evaluated":true,"basis":{"sex":"male","age_band":"31-50","version":1},"summary":{"targets":17,"below_70":5,"limits_over":0},"nutrients":[{"metric":"dietary_vitamin_d","label":"Vitamin D","intake":50,"target":15,"unit":"µg","fraction":3.333,"has_data":true,"kind":"rda","group":"vitamin"}]}

One confirmed serving was logged. Argus now counts 50 µg of vitamin D toward coverage — 15 µg is the reference intake for a male aged 31–50 — and the 3.33 fraction is uncapped and is not a diagnosis or dosage recommendation.
```

Skills version 1.4.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
