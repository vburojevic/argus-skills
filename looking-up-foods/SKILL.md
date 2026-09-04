---
name: looking-up-foods
version: 1.1.1
description: Use when searching the Argus food corpus, looking up a barcode/GTIN, reading nutrition facts for a food, or checking data provenance and licensing of food records.
license: MIT
---

# Looking up foods with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

The corpus blends USDA FoodData Central (public domain), Open Food Facts (ODbL, firewalled store), and open-gov tables. Food ids encode the store: `food_u…` (USDA), `food_o…` (OFF), `food_c/n/a…` (open-gov).

```sh
argus foods search "chicken breast"             # fused full-text + fuzzy search
argus foods search "milka oreo" --limit 5
argus foods get food_u171077                    # full nutrient vector (per 100 g) + portions
argus foods barcode 3017620422003               # local corpus, then live OFF
```

## Reading results

- `nutrients` keys are `dietary_*` metric names in canonical units **per 100 g**; scale by portion grams yourself, or just log a meal and let the server do it.
- `portions` lists household measures with gram weights — use them for "1 slice"-style math.
- Every record carries `source`, `license`, and `attribution`. **When showing OFF data (`source: "off"`) to users, include the attribution string** — it's an ODbL condition, not decoration.
- OFF rows carry `origin: "dump" | "live"`; `live` means Argus fetched and stored the product during this lookup.

## Choosing between search, barcode, and infer

- Have a package? `foods barcode` is exact. It accepts 8/12/13/14 digits plus printed spaces or dashes, checks the digit, searches USDA then the local OFF store, and finally makes one rate-limited live OFF lookup.
- Know the food name? `foods search`, then `foods get` for facts.
- Have a meal description? `argus infer "…"` parses AND grounds in one call — usually the better tool for natural language.

Barcode problems are instructions, not near-matches:

- `barcode_invalid`: re-read or type the printed number; do not repair it.
- `barcode_in_store`: the code is meaningful only inside that shop; ask for a description.
- `barcode_not_food`: ISBN/ISSN publication code; stop.
- `barcode_unknown`: not in local data or live OFF; fall back to `argus infer "<the user's description>"`, never invent product facts.

## Worked transcript

```text
$ argus foods barcode 3017620422003 --output json
{"id":"food_o03017620422003","description":"Nutella","source":"off","origin":"live","license":"ODbL-1.0","attribution":"Open Food Facts (openfoodfacts.org), ODbL 1.0","nutrients":{"dietary_energy_consumed":539},"portions":[]}

This exact barcode resolved through Open Food Facts. The values are per 100 g; preserve the attribution, and confirm grams before logging it.
```

Food search currently returns one bounded result array rather than a cursor page. Raise `--limit` within the command's maximum; do not pass `--paginate` or claim that every corpus match was returned.

Skills version 1.1.1 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
