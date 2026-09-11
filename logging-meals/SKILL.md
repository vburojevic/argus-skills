---
name: logging-meals
version: 1.4.0
description: Use when logging food, meals, or drinks to Argus from natural language, resolved foods, or structured inline nutrition, correcting or deleting logged meals, or previewing nutrition without writing.
license: MIT
---

# Logging meals with argus

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

## The hero path — free text

```sh
argus log "2 eggs and a slice of sourdough with butter"
argus log "a burrito" --confirm-items      # preview the inferred items; nothing is written until --confirm (exit 4 otherwise)
argus log "protein shake with 300ml milk" --at 2026-08-25T08:30:00+02:00
```

The server parses the text (LLM), grounds each item against the food corpus, and creates a meal whose nutrients **automatically project to `dietary_*` metrics** — after logging, `argus summary` already reflects it. The response's `items[].source` says `grounded` (matched to a real food), `estimated` (model estimate — nutrition is approximate), or `manual` (nutrition you supplied on the line and Argus stored verbatim; surfaces label it "as stated"). Those three are the whole set.

## Repeat an exact stored meal

```sh
argus usuals --output json
argus log --repeat meal_abc123
```

`argus usuals` ranks replayable meals for the current local clock. `argus log --repeat` copies the chosen meal's stored items and nutrient vectors exactly, including estimated items, without another inference call. MCP tools: `list_meal_usuals`, `log_meal`. **Meals logged from a Recipe never become usuals** — the Recipe is already that meal's named quick-log, so listing it twice would duplicate the choice; reach for `argus recipes list` alongside `argus usuals` when offering someone their repeats (skill `managing-recipes`).

Pass repeat_of using the returned meal_id; do not reconstruct text or infer favorites from the count.

## Preview without writing

```sh
argus infer "cevapi in lepinja with ajvar"     # parse + ground, writes nothing
```

Use this when the user wants to know "how much protein is X" without logging it.

## Precise logging (resolved items)

```sh
argus foods search "greek yogurt"               # find food ids (food_…)
argus meals log-food food_u12345 --grams 200
argus meals log-food food_o03017620422003 --grams 30 --at 2026-08-25T08:30:00+02:00
```

Use `log-food` after `argus foods barcode` or `foods get` returns an exact record and the person confirms its weight. MCP tool `log_meal` is the equivalent, with `items: [{"food_id":"food_…","grams":30}]`; quantity defaults to one when exact grams are supplied. Do not turn package weight into consumed weight without confirmation.

## Inline nutrition from an agent

Use MCP tool `log_meal` when the submitting agent already has ingredient-level nutrition. One meal may mix corpus-backed and inline lines:

```json
{
  "at": "2026-08-31T12:30:00+02:00",
  "timezone": "Europe/Zagreb",
  "name": "Batch plate",
  "client_ref": "meal-import-2026-08-31-lunch",
  "origin": "meal-import",
  "items": [
    { "food_id": "food_u12345", "qty": 1, "grams": 120 },
    {
      "text": "Prepared grains",
      "qty": 1,
      "grams": 180,
      "basis": "per_100g",
      "nutrients": {
        "dietary_energy_consumed": 150,
        "dietary_protein": 0.05
      }
    }
  ]
}
```

Nutrition values are numeric and canonical: kcal for `dietary_energy_consumed`, grams for every mass nutrient including sodium and micronutrients. Do not send strings with unit suffixes. Unknown nutrient keys are accepted but returned in `ignored_nutrients`; malformed, negative, or non-finite values are rejected.

| `basis` | Meaning | Required arithmetic |
|---|---|---|
| `total` | Absolute nutrients for the consumed line | Store as supplied |
| `per_100g` | Nutrients per 100 g | Multiply by `grams / 100`; `grams` is required |
| `per_item` | Nutrients for one item | Multiply by `qty` |

Conversion cribsheet before submission:

- sodium grams = salt grams ÷ 2.5
- kcal = kJ ÷ 4.184
- absolute nutrient = per-serving nutrient × servings consumed
- convert every supported mass to grams; do not guess an IU conversion without the compound form

`client_ref` is a per-user idempotency key: retries with the same value return the existing Meal instead of logging twice. `origin` is a free-form source label, not an integration enum; it appears only in meal detail/history. Both belong to the meal request, not to individual lines.

## Worked transcript

```text
$ argus foods barcode 3017620422003 --output json
{"id":"food_o03017620422003","description":"Nutella","source":"off","origin":"live","portions":[]}
$ argus meals log-food food_o03017620422003 --grams 30 --output json
{"id":"meal_abc123","items":[{"description":"Nutella","grams":30,"food_id":"food_o03017620422003","source":"grounded"}]}

Thirty confirmed grams were logged from the resolved food record. Its nutrient projection now appears in Today and summary.
```

## Correcting and deleting

```sh
argus meals list --date 2026-08-25 --paginate   # find the meal id across every page
argus meals get meal_abc123                     # inspect items before changing them
argus meals edit meal_abc123 --text "3 eggs and toast"   # re-infer, projections follow
argus meals delete meal_abc123                  # exits 4; re-run with --confirm
```

Edits and deletes **propagate to the projected metrics** — totals, summary, and goal progress all update; never try to fix projections directly.

## Gotchas

- A meal logged with the wrong `--at` lands on the wrong day in `summary`; edit the meal rather than re-logging.
- `estimated` items are honest guesses; if accuracy matters, search the corpus and log resolved items instead.
- Inline nutrition is a per-user snapshot. It never writes to or trains the shared food corpus.

Skills version 1.4.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
