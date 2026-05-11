# Prompt 08: Pilot Analysis, Reporting, and Scale Decision

You are implementing the analysis and reporting layer for the CVRO pilot.

This stage consumes the pilot run artifacts and produces the final pilot decision report. It should summarize whether CVRO is ready to scale to the full four-backbone experiment.

Do not implement data loading, parsers, model inference, operations, counterfactual collection, controller training, or pilot orchestration in this prompt. Use the artifacts produced by earlier stages.

## Goal

The pilot report should answer one practical question:

> Should we scale CVRO to the full four-backbone run?

The report should make that recommendation using predeclared evidence, not after-the-fact interpretation.

The pilot should be considered promising only if CVRO survives the main reviewer attacks:

1. it is not just scalar adaptive compute;
2. it is not just prompt-level routing;
3. it is not just learned Best-of-N;
4. it is not just verifier routing;
5. all four operations have nontrivial value regimes;
6. the infrastructure is reliable enough to trust the measurements.

## Inputs

This stage should consume artifacts from the completed pilot run.

Expected inputs include:

| Artifact | Description |
|---|---|
| `run_manifest.json` | Run configuration, artifact paths, stage status. |
| `task_manifest.json` | Dataset counts and source metadata. |
| `state_extraction_report.json` | State extraction diagnostics. |
| `parser_validation_report.json` | Parser/checker diagnostics. |
| `counterfactual_collection_report.json` | Outcome collection integrity report. |
| `action_values_train.*` | Train action values. |
| `action_values_val.*` | Validation action values. |
| `action_values_test_audit.*` | Test-audit action values. |
| `controller_predictions_val.*` | Controller validation predictions. |
| `controller_predictions_test_audit.*` | Controller test-audit predictions. |
| `metrics_raw.jsonl` | Per-example/method runtime evaluation metrics. |
| `metrics_summary.json` | Aggregated pilot metrics if already produced. |
| `pilot_gates.json` | Gate statuses if already computed. |
| `baseline_results/` | Outputs and scores for baselines. |
| `cvro_policy_results/` | CVRO runtime policy outputs and scores. |
| `robustness_results/` | Robustness evaluation outputs. |
| `ablation_results/` | Remove-action ablation outputs. |

The exact file paths should be read from the run manifest or config.

## Outputs

Produce both machine-readable and human-readable outputs.

Required outputs:

| Artifact | Description |
|---|---|
| `results/pilot_summary.md` | Main human-readable pilot report. |
| `results/pilot_summary.json` | Machine-readable summary. |
| `results/pilot_gates.json` | Pass/warn/fail status for each gate. |
| `results/tables/*.csv` | Main result tables. |
| `results/figures/*` | Plots for action distributions, value curves, and cost frontiers. |
| `results/scale_decision.md` | Short scale/no-scale recommendation. |
| `results/failure_examples.jsonl` | Representative failure cases and wrong-operation examples. |
| `results/artifact_audit.json` | Completeness and integrity audit. |

If plots are not available in a dry run, still produce tables and summaries.

## Report Structure

The main `pilot_summary.md` should include these sections:

1. Executive summary
2. Pilot configuration
3. Infrastructure reliability
4. Oracle action distribution
5. Headroom analysis
6. CVRO vs dangerous baselines
7. Remove-action ablations
8. Action-regime analysis
9. Marginal value curves
10. Robustness evaluation
11. Failure taxonomy
12. Cost and runtime accounting
13. Gate results
14. Scale recommendation
15. Known limitations before full run

The report should be concise but complete. It should not bury the main decision.

## Scale Decision Labels

Use three possible final decisions:

| Decision | Meaning |
|---|---|
| `scale` | Results are strong enough to proceed to full four-backbone run. |
| `revise_and_rerun_pilot` | Mechanism is promising but one or more critical gates failed. |
| `do_not_scale` | Pilot evidence suggests CVRO is not currently viable. |

Do not produce a vague recommendation. The report should select one of these labels and explain why.

## Main Pilot Gates

Compute pass/warn/fail status for every gate below.

### Infrastructure Gates

| Gate | Pass target | Warning region | Fail condition |
|---|---:|---:|---:|
| State extraction failure rate | `<= 3%` | `3-6%` | `> 6%` |
| Parser failure rate overall | `<= 5%` | `5-8%` | `> 8%` |
| Parser failure rate per domain | `<= 8%` | `8-12%` | `> 12%` |
| Broken verifier/checker rate | `<= 3%` | `3-6%` | `> 6%` |
| Code sandbox timeout unrelated to bad solution | `<= 3%` | `3-6%` | `> 6%` |
| CVRO runtime loop failure | `<= 3%` | `3-6%` | `> 6%` |
| Budget matching error | `<= 5%` | `5-8%` | `> 8%` |

### Oracle-Action Gates

| Gate | Pass target | Warning region | Fail condition |
|---|---:|---:|---:|
| Stop oracle-best share | `>= 10%` | `7-10%` | `< 7%` |
| Continue oracle-best share | `>= 10%` | `7-10%` | `< 7%` |
| Branch oracle-best share | `>= 8%` | `5-8%` | `< 5%` |
| Verify/check oracle-best share | `>= 8%` | `5-8%` | `< 5%` |
| Largest single oracle-best action share | `<= 55%` | `55-65%` | `> 65%` |
| Action entropy | report value | report value | flag if very low |

### Headroom Gates

| Comparison | Pass target | Warning region | Fail condition |
|---|---:|---:|---:|
| Oracle action policy vs scalar adaptive compute | `>= 8` accuracy points better, or `>= 5` points with `>= 25%` lower token cost | positive but smaller | no meaningful advantage |
| Oracle action policy vs matched Best-of-N verify | `>= 5` points better, or equal accuracy with `>= 25%` lower token cost | positive but smaller | no meaningful advantage |

### Learned-CVRO Gates

| Comparison | Pass target | Warning region | Fail condition |
|---|---:|---:|---:|
| CVRO vs scalar adaptive compute | `>= 2` accuracy points better, or equal accuracy with `>= 20%` fewer tokens | positive but smaller | scalar baseline matches or beats CVRO |
| CVRO vs prompt/input-only | `>= 2` points better | `0-2` points better | prompt/input-only matches or beats CVRO |
| CVRO vs shuffled-state | `>= 3` points better | `1-3` points better | shuffled-state matches or nearly matches CVRO |
| CVRO vs matched Best-of-N verify | `>= 1-1.5` points better, or equal accuracy with `>= 20%` fewer tokens | near tie | Best-of-N clearly better |
| CVRO vs long CoT | `>= 20%` fewer tokens at similar or better accuracy | smaller efficiency gain | long CoT clearly better |

### Remove-Action Gates

Report these as diagnostic rather than hard pass/fail unless the pattern is completely degenerate.

| Ablation | Expected signal |
|---|---|
| `no_stop` | More tokens and/or more overthinking flips. |
| `no_continue` | Lower accuracy on coherent partial trajectories. |
| `no_branch` | Drop in high-uncertainty or disagreement states. |
| `no_verify_check` | Drop in code or candidate-present states. |

## Metrics to Compute

For each method and baseline, compute:

| Metric | Description |
|---|---|
| Accuracy | Final correctness. |
| Generated tokens | Total generated-token cost. |
| Tool/checker cost | Checker calls/runtime/normalized cost. |
| Total normalized cost | Combined cost if configured. |
| Invalid-output rate | Parse/check failure rate. |
| Runtime loop failure rate | Runtime errors. |
| Operation distribution | Chosen stop/continue/branch/verify_check rates. |
| Regret | Oracle value minus selected-action value on audit states. |
| Operation-selection accuracy | Agreement with oracle-best action. |
| Top-2 operation accuracy | Selected action is among top two oracle actions. |
| Overthinking flip rate | Correct-to-wrong after extra reasoning where measurable. |
| Underthinking error rate | Stopped when useful operation existed where measurable. |
| Per-domain metrics | Math/code/symbolic breakdown. |
| Per-state-position metrics | Early/middle/late/post-operation breakdown. |

Use consistent metric definitions across CVRO and baselines.

## Dangerous Baseline Comparisons

The report must explicitly compare CVRO against the dangerous baselines.

Required comparison tables:

1. CVRO vs scalar adaptive compute
2. CVRO vs prompt-only/input-only controller
3. CVRO vs shuffled-state controller
4. CVRO vs matched Best-of-N vote
5. CVRO vs matched Best-of-N verify
6. CVRO vs always-continue
7. CVRO vs always-branch
8. CVRO vs always-verify/check
9. CVRO vs early exit
10. CVRO vs long CoT

For each comparison, report:

- accuracy difference;
- token-cost difference;
- tool-cost difference;
- total-cost difference if available;
- invalid-output difference;
- confidence interval if available;
- whether the gate passed.

## Matched Best-of-N Analysis

Best-of-N is one of the main threats to CVRO.

The report must include a specific Best-of-N section with:

| Item | Description |
|---|---|
| candidate count | Number of candidates generated. |
| per-candidate budget | Token budget per candidate. |
| total budget | Generated tokens plus verifier/checker cost. |
| aggregation rule | Vote, verifier, confidence, etc. |
| verifier access | Whether same verifier/checker access as CVRO. |
| valid-output rate | Fraction of candidates that parsed or executed. |
| agreement rate | Candidate agreement rate. |
| verified-pass rate | Fraction passing verifier/checker. |
| CVRO-only wins | Tasks CVRO solves and Best-of-N misses. |
| Best-of-N-only wins | Tasks Best-of-N solves and CVRO misses. |

Classify CVRO-only wins by selected operation where possible:

| CVRO winning operation | Interpretation |
|---|---|
| stop | CVRO avoided overgeneration or answer drift. |
| continue | Current trajectory was useful and cheaper than sampling. |
| branch | State-conditioned branch beat blind sampling. |
| verify_check | Targeted audit beat candidate generation. |

## Oracle Action Distribution

Report oracle-best action distribution by:

- overall;
- domain;
- source dataset;
- state position;
- state source;
- candidate-present flag;
- difficulty bucket;
- confidence bucket if available;
- backbone, if future runs add more backbones.

For the pilot, Qwen3-14B is sufficient.

The report should flag degeneracy:

- one action dominates over 55%;
- branch below 8%;
- verify/check below 8%;
- branch or verify/check below 5%;
- action entropy very low.

## Action-Regime Maps

Produce tables or plots showing which operation has highest value in different regimes.

Suggested regimes:

| Regime | Features |
|---|---|
| Early high uncertainty | early state, low confidence, no candidate |
| Middle coherent trajectory | middle state, candidate absent or stable, low contradiction |
| Late candidate present | candidate exists, near final answer |
| Repetition/drift | repeated phrases, answer flips, contradictions |
| Branch disagreement | post-branch states with candidate mismatch |
| Verify failure | post-verify-failure states |
| Code candidate present | code task with parseable candidate code |
| Symbolic rule conflict | symbolic task with inconsistent or uncertain label |

Do not hardcode expected outcomes. The report should show empirical distributions.

## Marginal Value Curves

Generate plots or tables showing action value as a function of:

- tokens used;
- state position;
- candidate presence;
- confidence/proxy confidence;
- task difficulty;
- domain;
- state source;
- answer history / answer changes.

At minimum, produce a table of mean Q-values by state position and domain.

If plotting is implemented, produce one plot per key relation. Avoid unreadable multi-axis figures.

## Remove-Action Ablation Analysis

Report performance for:

- full CVRO;
- no-stop;
- no-continue;
- no-branch;
- no-verify/check.

For each, report:

- accuracy;
- generated tokens;
- tool/checker cost;
- operation distribution;
- regret;
- domain breakdown.

Also report where each ablation hurts most, e.g.:

- no-branch on high-uncertainty states;
- no-verify/check on code candidate-present states;
- no-stop on late states or overthinking cases;
- no-continue on coherent middle states.

## Robustness Reporting

For robustness prompts, report:

| Metric | Description |
|---|---|
| accuracy by variant type | Original, paraphrase, renaming, irrelevant context, spurious cue, causal edit. |
| answer stability | Semantics-preserving variants should keep answer. |
| operation stability | Operation choice should be stable or explainably changed. |
| causal sensitivity | Causal edits should change answer when expected. |
| cost by variant type | Whether distractors/spurious cues cause excess compute. |

Robustness is secondary in the pilot. Do not overclaim. Use it to identify obvious instability before the full run.

## Failure Taxonomy

Create a failure taxonomy using available outcomes and oracle actions.

Classify failures where possible into:

| Failure type | Meaning |
|---|---|
| `premature_stop` | Controller stopped when another action had higher observed value. |
| `futile_continuation` | Controller continued a low-value path. |
| `missed_branch` | Branch was oracle-best but not selected. |
| `unnecessary_branch` | Branch was selected when another cheaper action was better. |
| `missed_verification` | Verify/check was oracle-best but not selected. |
| `wasteful_verification` | Verify/check selected but added cost without value. |
| `harmful_verification` | Verify/check degraded a correct candidate. |
| `wrong_stopping_point` | Stopped too early or too late. |
| `parser_or_checker_failure` | Evaluation infrastructure issue. |

Save representative examples for each failure type.

Output:

- `failure_examples.jsonl`
- a summary table in `pilot_summary.md`

## Statistical Reporting

Support paired bootstrap over task instances.

Use:

| Item | Requirement |
|---|---|
| Bootstrap unit | Task instance, not rollout. |
| Resamples | Configurable; default can be lower in dry run and higher in pilot. |
| Confidence interval | 95% where available. |
| Paired comparisons | CVRO vs each dangerous baseline. |

Do not bootstrap over rollouts as if they were independent. Rollouts from the same task/state are correlated.

If full bootstrap is not implemented yet, provide deterministic paired difference tables and mark bootstrap as pending. But the structure should support it.

## Cost and Runtime Reporting

Include a cost section.

Report:

- total generated tokens by method;
- average generated tokens per task;
- total checker calls by method;
- average checker runtime;
- invalid-output rate;
- normalized total cost if configured;
- wall-clock runtime for pilot stages if available;
- Modal/GPU cost estimate if available from logs.

Do not report token savings without also reporting accuracy.

Do not report verify/check performance without checker cost.

## Artifact Completeness Audit

Before writing the final decision, run an artifact audit.

Check:

- all required input artifacts exist;
- task counts match expected pilot counts;
- state counts match expected counts;
- outcome counts match expected counts or missingness is reported;
- action values exist for expected states;
- controller predictions exist for required controllers;
- baseline outputs exist;
- robustness outputs exist;
- ablation outputs exist;
- train/val/test split isolation is intact;
- no duplicate IDs;
- no missing critical config hashes.

Write:

- `artifact_audit.json`
- artifact audit section in report.

## Scale Recommendation Logic

The final recommendation should be based on the gates.

Suggested logic:

### Recommend `scale` if:

- infrastructure gates pass or have only minor warnings;
- all four operations have nontrivial oracle-best shares;
- CVRO beats scalar adaptive compute;
- CVRO beats prompt/input-only or shuffled-state controls;
- CVRO is competitive with matched Best-of-N verify;
- oracle upper bound shows meaningful headroom;
- no catastrophic parser/checker issue appears.

### Recommend `revise_and_rerun_pilot` if:

- infrastructure is mostly sound;
- oracle action distribution is promising;
- but one dangerous baseline matches or beats CVRO;
- or one operation is underused because of likely prompt/checker/task-mix issues;
- or robustness exposes fixable instability.

### Recommend `do_not_scale` if:

- scalar adaptive compute matches or beats CVRO;
- prompt/input-only or shuffled-state controls match CVRO;
- matched Best-of-N verify clearly dominates;
- one operation dominates and branch/verify are degenerate;
- oracle upper-bound headroom is small;
- parser/checker/state extraction failures make results unreliable.

The report should not hide a failed pilot behind positive secondary metrics.

## Human-Readable Summary

The executive summary should include:

- final decision;
- top 3 reasons;
- top 3 risks;
- main result table;
- gate pass/warn/fail summary;
- whether to proceed to the full run.

Keep it direct.

## Local Dry Run

The analysis code should work on dry-run artifacts.

In dry-run mode:

- do not require all pilot counts;
- do not fail only because counts are tiny;
- still check schema and artifact integrity;
- produce a small report;
- clearly label it as dry run.

## Do Not Do

Do not tune thresholds after seeing pilot results.

Do not use robustness outcomes to change the main gate definitions.

Do not ignore failed baselines.

Do not report only aggregate accuracy.

Do not treat rollout samples as independent tasks.

Do not hide cost or invalid-output rates.

Do not overstate robustness results from the pilot.

Do not recommend scaling if the dangerous baselines invalidate the core claim.

## Definition of Done

This prompt is complete when the repository can:

1. load a completed pilot run directory,
2. audit artifact completeness,
3. compute infrastructure, oracle-action, headroom, learned-CVRO, and ablation gates,
4. compare CVRO against all dangerous baselines,
5. summarize action-regime and marginal-value behavior,
6. report robustness results,
7. classify representative failures,
8. produce machine-readable result files,
9. produce a human-readable `pilot_summary.md`,
10. produce a clear `scale`, `revise_and_rerun_pilot`, or `do_not_scale` recommendation.

The final pilot report should make it obvious whether the project is ready for the full four-backbone experiment.
