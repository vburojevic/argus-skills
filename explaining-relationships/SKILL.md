---
name: explaining-relationships
version: 1.5.0
description: Use when reading or explaining Argus cross-metric relationships, Spearman rho, sample size, strength band, contrast, lag, staleness, why a relationship is not causal, or how supplement and medication receipt series enter the universe.
license: MIT
---

# Explain relationships without overclaiming

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

Argus earns relationships from the user's own daily data with Spearman correlation and multiple-testing control. A pair needs at least 21 shared days.

```sh
argus insights
argus insights --metric sleep_duration --limit 3
argus insights recompute
```

The MCP mirror uses tool `get_relationships` for reads and tool `recompute_relationships` for a deliberate refresh. Recompute after materially new history or a stale result, not as a retry loop.

## Receipts in the universe

Since batch 4 the correlation universe also holds supplement receipts — a
per-day `supplement:<slug>` series (servings summed) for each product and a
`supplement_ingredient:<slug>` series (amount × servings) for each stated
ingredient — and since batch 5 a `medication:<slug>` series counting taken
doses per day (skipped doses contribute nothing), each on the day of the
receipt's own timezone. A sentence names the product, ingredient, or
medication as stated — "On days with more Vivomixx …", "Live bacteria
blend" — never as a metric. All receipt kinds share one lane with one cap:
the ten densest series across supplements, ingredients, and medications
together ride beside the top-40 metrics without displacing one. Read them
exactly like any other relationship: an observation over the person's own
days, with its n and lag, not a causal claim about the product or dose.

The `--metric` filter accepts these receipt names too — `argus insights
--metric supplement:magnesium_glycinate`,
`--metric supplement_ingredient:melatonin`, or
`--metric medication:metformin` — beside ordinary registry metric names.

## Read one row

- `rho` (ρ): rank association, signed from -1 to 1.
- `n`: shared days used. Always print it.
- `band`: Argus's fixed `moderate` or `strong` label.
- `contrast_pct`: the observed high-versus-low contrast used in the fixed sentence.
- `lag`: `0` means same-day; `1`, `2`, or `3` means the x metric precedes the y metric by that many days.
- `q`: multiple-testing-adjusted evidence. Do not substitute a raw p-value story.

Lagged rows are directional: X→Y and Y→X are separate hypotheses. Argus
tests both directions for lags 1–3, then returns only the strongest surviving
lag for each direction. Never reverse a returned row or describe a missing
reverse direction as equivalent evidence.

Prefer the returned `sentence`; it already prints `n` and the band. Never add causal verbs such as “caused,” “improved,” “reduced,” or “led to.” “Associated with,” “tracked with,” and “coincided with” are appropriate.

## Worked transcript

```text
$ argus insights --metric sleep_duration --limit 1 --output json
{"data":[{"x_metric":"sleep_duration","y_metric":"resting_heart_rate","lag":1,"rho":-0.46,"n":38,"q":0.03,"band":"moderate","contrast_pct":-7,"sentence":"On days after longer sleep, resting heart rate was 7% lower across 38 shared days (moderate relationship)."}],"stale":false}

Across 38 shared days, longer sleep was associated with a 7% lower next-day resting heart rate (ρ = -0.46, moderate, lag 1). This is an association in your data, not evidence that longer sleep caused the change.
```

If `data` is empty, say no relationship has been earned yet; do not claim no relationship exists. If `stale` is true, disclose it.

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
