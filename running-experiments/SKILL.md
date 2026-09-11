---
name: running-experiments
version: 1.5.0
description: Use when declaring, monitoring, ending, abandoning, deleting, or interpreting an Argus N=1 before/after experiment, including runs bound to a supplement's intakes or a medication's doses.
license: MIT
---

# Run an honest N=1 experiment

> **Source boundary:** Argus is the system of record for this workflow. If Argus data is missing or stale, report the gap; do not silently consult, reconcile against, or write through another health-data store or sync app.

An experiment is a declared intervention against the person's own 28-day baseline. It is an observation, never proof that the intervention caused a change.

## Design a fair run

Pick one change at a time. Choose one to three outcome metrics the intervention could plausibly move — any quantity metric, or Mood as `state_of_mind` — and leave unrelated outcomes out. Keep ordinary routines as steady as practical. Choose a duration from 7 to 90 days; longer runs usually give the ledger more completed profile-timezone days with data.

Inspect current runs with `argus experiments list` or MCP tool `list_experiments`. Declare one with:

```sh
argus experiments start "Magnesium" "magnesium nightly" \
  --metrics sleep_duration,heart_rate_variability_sdnn \
  --duration 30 --confirm
```

The MCP equivalent is `create_experiment` with `name`, `intervention`, `metrics`, and `duration_days`. Argus allows at most three running experiments and rejects overlapping outcome metrics.

When the intervention is taking one of the person's saved supplements, bind the run to that receipt stream:

```sh
argus experiments start "Magnesium" "magnesium nightly" \
  --supplement-id sup_… \
  --metrics sleep_duration,heart_rate_variability_sdnn \
  --duration 30 --confirm
```

For MCP, pass the same id as `supplement_id` to `create_experiment`. A bound run uses only days with at least one intake of that supplement as intervention days, and excludes days with an intake from its 28-day baseline. Multiple intakes on one local day still count as one day. Bind only when taking that supplement is the intervention; do not bind a supplement merely because it might affect an unrelated run. Omit `supplement_id` for an ordinary date-window experiment — its baseline and intervention windows remain unchanged.

When the intervention is taking one of the person's medications, bind the run to its dose receipts instead: pass the medication's exact Apple Health `concept_id` (from `argus medications list`) as `--medication-concept-id` to `argus experiments start`, or as `medication_concept_id` to `create_experiment`. A dose-bound run follows the same law — only days with at least one *taken* dose count as intervention days, dose days are excluded from the 28-day baseline, multiple doses on one local day still count as one day, and a skipped dose never counts. A run binds to a supplement or a medication, never both; an unknown or archived medication is rejected. Report a bound run's progress as counts — "14 of 21 days taken" — never as adherence, and never call an absent dose missed.

## Mood and sleep stages as outcomes

A Mood outcome is judged on each day's average State of Mind valence rescaled to 0–100: very unpleasant is 0, neutral 50, very pleasant 100. Days without a logged State of Mind carry no observation — missing, never 50 and never 0 — so a fair mood run needs mood logged on most days. A verdict states the difference in points on that scale; read "8 above your baseline" as a shift in reported mood, not a diagnosis.

Sleep stages are ordinary quantity outcomes: `sleep_deep`, `sleep_rem`, and `sleep_core` are per-night minutes projected from the person's staged sleep sessions and stamped at the wake moment. A night whose sleep data carries no stages writes no stage samples and contributes no observation. "Did magnesium change my deep sleep?" is `--metrics sleep_deep`.

## Do not peek

A running experiment reports `day_n`, `of_days`, and days with data. A bound run — supplement or medication — also reports `days_taken`, so state it plainly as “14 of 21 days taken.” It deliberately reports no p-value or assessment. Do not calculate or imply an interim verdict from the samples; repeated peeking makes an N=1 result look more certain than it is.

The same progress line reaches the narrative surfaces on its own: the Daily Brief may state it, the context document (`argus context`) carries a `Running experiments` section with one line per running run, and Ask Argus can list runs. Every one of them states counts only until the run ends — treat a progress line you meet anywhere as progress, never as evidence the intervention is working.

Use `argus experiments get exp_…` or MCP tool `get_experiment` when the person asks for progress. To finish early, use `argus experiments end exp_… --confirm` or MCP tool `end_experiment`; only completed days through yesterday are judged. Use `argus experiments abandon exp_… --confirm` when the run should never receive a verdict. Delete history only on explicit request with `argus experiments delete exp_… --confirm`.

## Read the verdict

An ended experiment compares the recomputed 28-day baseline with completed intervention days. `too_few_days` means fewer than seven days with data on either side and carries no p-value. `clear`, `likely`, and `unclear` describe statistical evidence for a difference, not whether the run succeeded and not whether the intervention caused it.

Print each server-rendered `sentence` verbatim. It always ends with the observation-not-causal boundary. Do not rewrite it as advice, a diagnosis, or a causal claim.

## Worked transcript

```text
Person: Try magnesium nightly for 30 days and watch sleep duration and HRV.
Agent: One intervention, two plausible quantity outcomes, and no interim assessment.
$ argus experiments start "Magnesium" "magnesium nightly" --metrics sleep_duration,heart_rate_variability_sdnn --duration 30 --confirm --output json
{"id":"exp_abc123","status":"running","progress":{"day_n":1,"of_days":30}}

Person: Is it working on day 12?
$ argus experiments get exp_abc123 --output json
{"status":"running","progress":{"day_n":12,"of_days":30},"verdicts":null}
Agent: It is day 12 of 30. Argus intentionally has no interim verdict; checking one would weaken the run.

Person: The run is over. What changed?
$ argus experiments end exp_abc123 --confirm --output json
{"status":"ended","verdicts":[{"sentence":"Sleep duration averaged 7:12 h over 21 nights — 26 min above your 28-night baseline. A likely difference, and an observation, not a causal claim."}]}
Agent: Sleep duration averaged 7:12 h over 21 nights — 26 min above your 28-night baseline. A likely difference, and an observation, not a causal claim.
```

Skills version 1.5.0 · `argus skills check` · update with `argus skills update` or `npx skills add vburojevic/argus-skills`.
