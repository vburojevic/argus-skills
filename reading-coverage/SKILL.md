---
name: reading-coverage
version: 1.0.0
description: Use when reading or explaining Argus micronutrient coverage — the ledger sentence, the reference-intake basis, targets versus limits, a real zero on a logged day, one nutrient's contributors, or its 90-day history.
license: MIT
---

# Read micronutrient coverage with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Coverage compares what the person logged — meals, supplement intakes, entries made by hand — with adult Dietary Reference Intakes for their sex and age band. Eighteen nutrients: eleven vitamins, six minerals, and fiber. It is a ledger, not a diagnosis.

Two reads:

```sh
argus coverage                       # today, all eighteen
argus coverage --date 2026-09-03
argus coverage --days 30             # 7, 30, or 90
argus coverage dietary_zinc          # one nutrient, whole
argus coverage dietary_zinc --date 2026-09-03
```

MCP mirrors both: tools `get_nutrition_coverage` and `get_nutrition_coverage_detail`.

A metric outside the eighteen is 404 `unknown_nutrient`. A profile without a biological sex and birth date is 409 `profile_incomplete` on the detail, and `evaluated: false` carrying `reason` on the card — say which, rather than guessing the missing facts.

## The sentence and the basis

`summary` carries `targets`, `below_70`, and `limits_over`. Read it; never recount the rows — every surface says the same "5 of 17 below 70 %" because they all read this one object.

- `below_70 > 0` → "5 of 17 below 70 %"; otherwise "All 17 at 70 % or above".
- Then one clause per limit: "sodium under the limit" or "sodium over the limit". Over a period, a limit reads as over only when `chronically_over` is true — two heavy days in thirty are not the month's verdict.

`basis` is `{sex, age_band, version}`. **State it whenever you quote a target**: "Reference intakes · male 31–50". A 420 mg magnesium target is that band's figure, not a universal one, and the band changes on a birthday.

## What the figures mean

- `kind` is `rda`, `ai`, or `limit`. RDA and adequate intake are floors — higher is closer. **`limit` inverts**: sodium's 2 300 mg is a budget, and `fraction > 1` is over it, not met. Never describe a limit as "met" or read a full bar there as good.
- `fraction` is uncapped: 3.33 means three and a third times the reference intake. Report it as logged. It is not a dosage recommendation and not a diagnosis.
- `group` is the heading a nutrient renders under — `vitamin`, `mineral`, or `other`. Sodium's group is `mineral`; it appears under Limits because of its `kind`.
- Period rows carry `mean_fraction`, `gap` (target minus mean intake), and `days_below_70` or `days_over` against `logged_days`.

## Zero is a fact, absence is not

On a day with food on the books every one of the eighteen is returned, and an intake of 0 is a **real zero** — nothing logged that day carried vitamin K. Say so plainly. On a day with nothing logged the whole card is `evaluated: false`, `reason: "nothing_logged"`, and the answer is "unknown", never zero. The same line runs through the detail's `history`: a point with `logged: false` is an unknown day; do not average it, plot it, or call it a miss.

## One nutrient, whole

`argus coverage dietary_zinc` returns the day's row, its `basis`, its `contributors`, and 90 profile days of `history` ending on that date — slice 7, 30, or 90 from it rather than asking again.

Each contributor is a meal, a supplement intake, or a hand entry, with its title, its time, and its `amount` in the nutrient's own unit. Their amounts **sum to the row's `intake`**: meal and supplement projections are never superseded, so the lines are the whole of the figure. Attribute honestly — "3.4 of the 8.4 mg came from Zinc picolinate at 21:00" — and never guess which food carried what when the ledger does not say.

## Worked transcript

```text
$ argus coverage dietary_zinc --output json
{"date":"2026-09-03","basis":{"sex":"male","age_band":"31-50","version":1},"nutrient":{"metric":"dietary_zinc","label":"Zinc","unit":"mg","intake":8.4,"target":11,"kind":"rda","group":"mineral","fraction":0.764,"has_data":true},"contributors":[{"kind":"meal","id":"meal_abc","title":"Oats & blueberries","at":"2026-09-03T05:40:00.000Z","amount":1.9,"unit":"mg"},{"kind":"supplement","id":"supplementIntake_abc","title":"Zinc picolinate","at":"2026-09-03T19:00:00.000Z","amount":3.4,"unit":"mg"}],"history":[]}

Zinc is 8.4 of 11 mg today — 76 % of the RDA for a male aged 31–50. Breakfast carried 1.9 mg and the 21:00 Zinc picolinate 3.4 mg. That is a shortfall on one day against one reference intake, not a deficiency.
```

Skills version 1.0.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
