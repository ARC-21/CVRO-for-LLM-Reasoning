# Prompt 07: Pilot Experiment Orchestration

You are implementing the end-to-end pilot experiment orchestration for CVRO.

This stage wires together the existing data pipeline, parsers/verifiers, state extraction, operations, counterfactual collection, controllers, dangerous baselines, robustness evaluation, and pilot gate checks.

The goal is to make the **Qwen3-14B decisive pilot** runnable from config with minimal manual intervention.

Do not reimplement dataset adapters, parsers, operations, counterfactual collection internals, or controller internals here. Use the modules produced by earlier prompts. This prompt should create the pilot runner, configs, baseline orchestration, robustness subset generation/evaluation, artifact manifests, and execution entry points.

## Pilot Objective

The pilot should answer whether CVRO is worth scaling to the full four-backbone run.

The pilot must test whether:

1. all four operations have nontrivial value regimes;
2. state-conditioned CVRO beats scalar adaptive compute;
3. state-conditioned CVRO beats prompt/input-only routing;
4. state-conditioned CVRO beats shuffled-state controls;
5. state-conditioned CVRO beats or matches matched Best-of-N with lower cost;
6. parsers, checkers, state extraction, and operation execution are reliable enough for the full run.

The pilot is not just a smoke test. It should be a small but decisive version of the main experiment.

## Pilot Scope

Use the following pilot setup:

| Item | Value |
|---|---:|
| Main backbone | `Qwen3-14B` |
| Canonical problems | `1,350` |
| Train / validation / test | `900 / 150 / 300` |
| Domains | `math`, `code`, `symbolic` |
| States per problem | `3` |
| Counterfactual outcomes | `26,100` |
| Robustness prompts | `1,800` |
| Controller seeds | `3` |

Domain split:

| Domain | Train | Validation | Test | Total |
|---|---:|---:|---:|---:|
| Math | `300` | `50` | `100` | `450` |
| Code | `300` | `50` | `100` | `450` |
| Symbolic | `300` | `50` | `100` | `450` |
| **Total** | **`900`** | **`150`** | **`300`** | **`1,350`** |

## End-to-End Pipeline

Implement a pilot runner that can execute the following stages in order:

1. prepare or validate processed task split;
2. validate parsers and verifiers;
3. generate initial trajectories;
4. extract online-observable natural states;
5. collect counterfactual outcomes from natural states;
6. generate and sample post-operation states;
7. collect full counterfactual outcomes on selected natural/post-operation states;
8. compute action values;
9. train CVRO controllers and dangerous controller baselines;
10. run end-to-end CVRO policy evaluation;
11. run dangerous baselines;
12. generate robustness variants;
13. run robustness policy evaluation;
14. run remove-action ablations;
15. aggregate metrics and pilot gates;
16. write artifacts and run manifest.

The runner should support running individual stages as well as the whole pipeline.

## Execution Modes

Support at least three modes:

| Mode | Purpose |
|---|---|
| `dry_run` | Tiny local run with mock/tiny data and mock/tiny model backend. |
| `pilot` | Full Qwen3-14B decisive pilot. |
| `resume` | Continue an interrupted pilot from completed artifacts. |

The dry run should be fast enough to run during development.

The pilot run should be Modal-compatible and sharded where needed.

## Required Entry Points

Implement command-line or script entry points equivalent to:

| Command intent | Description |
|---|---|
| prepare data | Build or validate processed task artifacts. |
| validate scoring | Run parser/verifier validation reports. |
| generate trajectories | Run initial trajectory generation. |
| extract states | Extract early/middle/late states. |
| collect counterfactuals | Force operations and build action-value labels. |
| train controllers | Train CVRO and controller baselines. |
| run baselines | Run dangerous baselines. |
| run robustness | Generate/evaluate robustness prompts. |
| run pilot all | Execute the full pilot pipeline. |
| summarize pilot | Aggregate results and produce summary artifacts. |

The exact CLI structure is up to you, but the workflow should be documented and config-driven.

## Config Requirements

Create pilot configs for:

| Config | Content |
|---|---|
| data config | Dataset paths, split IDs, dry-run settings. |
| model config | Backbone name, backend, decoding defaults, Modal resources. |
| state config | Checkpoint thresholds and extraction rules. |
| operation config | Operation prompts, token budgets, decoding settings. |
| counterfactual config | Rollout counts, post-operation mix, lambda values. |
| controller config | Feature sets, model types, seeds, hyperparameters. |
| baseline config | Baseline budgets, candidate counts, verifier access. |
| robustness config | Variant types, counts, validation rules. |
| pilot config | Stage order, artifact paths, run ID, resume behavior. |

Avoid hardcoding pilot parameters in source code.

## Artifact Manifest

Every pilot run should produce a run manifest.

The manifest should include:

- run ID
- timestamp
- git commit if available
- configs used
- config hashes
- dataset manifest hash
- model/backbone identifier
- prompt template versions
- parser/checker versions
- artifact paths
- stage completion status
- counts by stage
- failure counts
- cost summaries
- whether pilot gates passed

Suggested output:

- `artifacts/pilot/<run_id>/run_manifest.json`
- `artifacts/pilot/<run_id>/run_manifest.md`

## Dangerous Baselines

The pilot must include the baselines most likely to invalidate the CVRO claim.

### Required Baselines

| Baseline | Purpose |
|---|---|
| `direct_answer` | Lower bound without extended reasoning. |
| `short_cot` | Cheap fixed reasoning. |
| `long_cot` | Brute-force fixed reasoning. |
| `self_consistency` | Sampling baseline. |
| `best_of_n_vote` | Tests whether branching is just blind sampling. |
| `best_of_n_verify` | Tests verifier-assisted sampling. |
| `always_continue` | Tests whether CVRO is just more thinking. |
| `always_branch` | Tests whether CVRO is just repeated sampling. |
| `always_verify_check` | Tests whether CVRO is just verifier routing. |
| `early_exit` | Tests whether CVRO is just stopping. |
| `scalar_adaptive_compute` | Tests whether operation type matters beyond token budget. |
| `prompt_only_controller` | Tests whether intermediate state matters. |
| `input_only_controller` | Prompt plus metadata, but no intermediate state. |
| `shuffled_state_controller` | Tests whether state features are aligned signal. |
| `oracle_action_upper_bound` | Hindsight upper bound from counterfactual labels. |

If implementation time is tight, prioritize:

1. scalar adaptive compute;
2. prompt/input-only controller;
3. shuffled-state controller;
4. matched Best-of-N verify;
5. always-action policies;
6. oracle upper bound.

## Matched Best-of-N Requirements

Best-of-N must be implemented brutally enough that it cannot be dismissed as weak.

For `best_of_n_vote` and `best_of_n_verify`, enforce:

- same or lower total generated-token budget than CVRO at the chosen operating point;
- same answer parser;
- same verifier/checker access where applicable;
- same aggregation rule where possible;
- same scoring code;
- same task set;
- same robustness variants where evaluated.

For each domain/backbone, estimate CVRO validation cost and choose Best-of-N candidate counts and candidate budgets from validation only.

Do not tune Best-of-N on test performance.

Report:

- candidate count;
- per-candidate budget;
- total generated tokens;
- verifier/checker cost;
- valid-output rate;
- agreement rate;
- verified-pass rate;
- accuracy at matched cost.

## Scalar Adaptive Compute Baseline

Implement or orchestrate the scalar adaptive compute baseline from the controller layer.

It should receive comparable features to CVRO but must only choose scalar compute quantity, such as:

- stop/continue length;
- token budget;
- short/medium/long reasoning budget.

It cannot choose `branch` or `verify_check` as distinct operations.

This is one of the main baselines. It must be strong, not a strawman.

## Prompt/Input-Only and Shuffled Controls

Include:

| Control | Requirement |
|---|---|
| `prompt_only` | Uses original problem text only. |
| `input_only` | Uses original problem plus domain/source/difficulty metadata, no intermediate reasoning state. |
| `shuffled_state` | Preserves feature distributions but breaks state-example alignment. |

If `prompt_only` and `input_only` are equivalent in the implementation, document this clearly.

## End-to-End CVRO Policy Evaluation

The pilot should evaluate a trained CVRO controller in an actual runtime loop.

Runtime loop:

1. generate initial chunk;
2. extract online state features;
3. controller predicts Q-values;
4. select allowed operation;
5. execute operation;
6. update state;
7. repeat up to 3 decisions;
8. return final answer or candidate.

Maximum decisions:

| Setting | Value |
|---|---:|
| Max controller decisions | `3` |

The runtime policy must use the same feature extraction path as controller training.

Do not evaluate only static action-selection accuracy. The pilot needs end-to-end task accuracy and cost.

## Remove-Action Ablations

Run remove-action ablations for the pilot.

Required ablations:

| Ablation | Allowed actions |
|---|---|
| `no_stop` | continue, branch, verify/check |
| `no_continue` | stop, branch, verify/check |
| `no_branch` | stop, continue, verify/check |
| `no_verify_check` | stop, continue, branch |

These should be evaluated on the pilot test set.

They do not need to run on every robustness variant unless cheap, but they must run on the canonical test set.

## Robustness Subset

Generate robustness variants from the 300 canonical test problems.

Variants:

| Variant | Count |
|---|---:|
| Original | `300` |
| Paraphrase / formatting | `300` |
| Variable renaming / option-order | `300` |
| Irrelevant context | `300` |
| Spurious cue | `300` |
| Causal counterfactual edit | `300` |
| **Total** | **`1,800`** |

Run robustness policy evaluation on the 1,800 prompts for at least:

- CVRO;
- scalar adaptive compute;
- prompt/input-only controller;
- matched Best-of-N verify;
- long CoT.

Robustness is secondary in the pilot. Do not overcomplicate it. The goal is to catch obvious instability.

## Robustness Variant Requirements

Each variant should store:

- canonical task ID;
- variant ID;
- variant type;
- variant prompt;
- expected answer relation:
  - same answer;
  - changed answer;
  - changed structure;
- updated gold/checker target if answer changes;
- generation metadata;
- validation status.

For causal counterfactual edits, the correct answer should change predictably and remain checkable.

For semantics-preserving variants, the correct answer should remain the same.

If automatic generation is unreliable, allow deterministic or templated variants for the pilot.

## Pilot Gates

The pilot report must compute pass/fail or warning status for these gates.

### Infrastructure Gates

| Gate | Target |
|---|---:|
| State extraction failure rate | `<= 3%` |
| Parser failure rate overall | `<= 5%` |
| Parser failure rate per domain | `<= 8%` |
| Broken verifier/checker rate | `<= 3%` |
| Code sandbox timeout unrelated to bad solution | `<= 3%` |
| CVRO runtime loop failure | `<= 3%` |
| Budget matching error | `<= 5%` |

### Oracle-Action Gates

| Gate | Target |
|---|---:|
| Stop oracle-best share | `>= 10%` |
| Continue oracle-best share | `>= 10%` |
| Branch oracle-best share | `>= 8%` |
| Verify/check oracle-best share | `>= 8%` |
| Largest single oracle-best action share | `<= 55%` |
| Branch and verify/check share | absolutely not below `5%` each |

### Headroom Gates

| Comparison | Target |
|---|---:|
| Oracle action policy vs scalar adaptive compute | `>= 8` accuracy points better, or `>= 5` points with `>= 25%` lower token cost |
| Oracle action policy vs matched Best-of-N verify | `>= 5` points better, or equal accuracy with `>= 25%` lower token cost |

### Learned-CVRO Gates

| Comparison | Target |
|---|---:|
| CVRO vs scalar adaptive compute | `>= 2` accuracy points better, or equal accuracy with `>= 20%` fewer tokens |
| CVRO vs prompt/input-only | `>= 2` points better |
| CVRO vs shuffled-state | `>= 3` points better |
| CVRO vs matched Best-of-N verify | `>= 1-1.5` points better, or equal accuracy with `>= 20%` fewer tokens |
| CVRO vs long CoT | `>= 20%` fewer tokens at similar or better accuracy |

### Remove-Action Gates

| Ablation | Expected signal |
|---|---|
| `no_stop` | More tokens and/or more overthinking flips. |
| `no_continue` | Lower accuracy on coherent partial trajectories. |
| `no_branch` | Drop in high-uncertainty or disagreement states. |
| `no_verify_check` | Drop in code or candidate-present states. |

The pilot does not need every gate to pass perfectly, but the final report should clearly identify scale/no-scale status.

## Metrics to Aggregate

For every method/baseline, collect:

| Metric | Description |
|---|---|
| Accuracy | Final task correctness. |
| Generated tokens | Total generated-token cost. |
| Tool/checker cost | Checker calls/runtime/normalized cost. |
| Total normalized cost | Combined cost if configured. |
| Invalid-output rate | Parse/check failure rate. |
| Runtime loop failure rate | Runtime errors. |
| Overthinking flip rate | Correct-to-wrong due to additional reasoning where measurable. |
| Underthinking error rate | Stop when useful operation existed, where measurable. |
| Operation distribution | Chosen stop/continue/branch/verify_check rates. |
| Regret | Oracle value minus selected-action value on audit states. |
| Per-domain metrics | Math/code/symbolic breakdown. |
| Per-state-position metrics | Early/middle/late/post-operation breakdown. |

## Statistical Reporting

For the pilot, support paired comparisons over task instances.

Use:

| Item | Setting |
|---|---|
| Controller seeds | `3` |
| Decoding repeats | Use configured pilot values. |
| Bootstrap unit | Task instance, not rollout. |
| Confidence intervals | 95% if implemented. |

The final analysis prompt will handle detailed reporting, but the pilot runner should preserve all data needed for paired bootstrap analysis.

## Modal Execution

The pilot should be deployable on Modal.

Support:

- GPU batch jobs for model generation;
- CPU jobs for scoring/checking;
- sharded execution by task/state/method;
- artifact volumes or configured storage;
- retries;
- run IDs;
- resume behavior;
- aggregation after shards finish.

The implementation may include placeholders where credentials or environment-specific details are needed, but the code should not require notebooks.

## Local Dry Run

Implement a local dry run that exercises the whole orchestration on a tiny subset.

The dry run should:

- use a small mock or tiny model backend if necessary;
- process a handful of tasks;
- generate trajectories or mock them;
- extract states;
- run all four operations;
- compute action values;
- train a tiny controller;
- run at least a minimal CVRO policy evaluation;
- produce a small manifest and summary.

The dry run should catch schema/interface errors before GPU jobs are launched.

## Failure Handling

The pilot runner should not fail silently.

It should log and summarize:

- missing artifacts;
- failed model calls;
- failed parser/checker calls;
- incomplete state-action rollout groups;
- budget overruns;
- duplicated IDs;
- empty outputs;
- invalid outputs;
- robustness variant failures;
- baseline execution failures.

Individual failed examples should not crash the entire run unless they exceed configured thresholds.

## Artifact Outputs

Suggested output structure:

| Artifact | Purpose |
|---|---|
| `artifacts/pilot/<run_id>/manifest.json` | Full run manifest. |
| `artifacts/pilot/<run_id>/stage_status.json` | Stage completion status. |
| `artifacts/pilot/<run_id>/metrics_raw.jsonl` | Per-example/method metrics. |
| `artifacts/pilot/<run_id>/metrics_summary.json` | Aggregated metrics. |
| `artifacts/pilot/<run_id>/baseline_results/` | Baseline outputs and scores. |
| `artifacts/pilot/<run_id>/cvro_policy_results/` | CVRO runtime policy outputs. |
| `artifacts/pilot/<run_id>/robustness_results/` | Robustness outputs and scores. |
| `artifacts/pilot/<run_id>/ablation_results/` | Remove-action ablation results. |
| `artifacts/pilot/<run_id>/pilot_gates.json` | Pass/warn/fail gate statuses. |

The next prompt will create analysis and reporting artifacts from these outputs.

## Do Not Do

Do not tune prompts or operation definitions after seeing test results.

Do not use test-audit labels for controller training or validation.

Do not use robustness test performance to tune main hyperparameters.

Do not let Best-of-N have weaker parser/checker access than CVRO.

Do not treat verifier/checker costs as free.

Do not silently skip failed baselines.

Do not run an unrestricted multi-turn agent.

Do not expand beyond the four public operations in the pilot.

Do not require manual notebook intervention for the pilot.

## Definition of Done

This prompt is complete when the repository can:

1. run a tiny local dry-run pilot end to end,
2. launch or prepare a Modal-compatible Qwen3-14B pilot,
3. orchestrate trajectory generation, state extraction, counterfactual collection, controller training, baseline evaluation, CVRO runtime evaluation, robustness evaluation, and ablations,
4. produce raw metrics and artifact manifests,
5. compute pilot gate values or provide all inputs needed to compute them,
6. resume interrupted runs,
7. keep train/val/test-audit isolation intact,
8. enforce matched-cost baseline comparison as much as possible.

The output of this stage should be a complete pilot run directory that the analysis/reporting stage can turn into the final pilot decision report.
