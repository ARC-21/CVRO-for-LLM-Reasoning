# Prompt 05: Counterfactual Collection and Action-Value Construction

You are implementing the counterfactual outcome collection and action-value construction layer for the CVRO pilot.

This stage forces all four CVRO operations from saved `ReasoningState` objects, scores the resulting outputs, records costs, creates post-operation states, and computes cost-aware action values.

Do not implement dataset loading, parser internals, initial state extraction, operation internals, controller training, or final analysis in this prompt. Use the interfaces and artifacts produced by previous stages.

## Context

CVRO depends on same-state counterfactual comparisons.

For a saved reasoning state `s`, the pipeline must force:

1. `stop`
2. `continue`
3. `branch`
4. `verify_check`

from that same state, then score the results using the task-specific scoring layer.

The action values should answer:

> What happened when this operation was forced from this exact state under this cost budget?

This stage is the core empirical data-generation step for CVRO.

## Inputs

This stage should consume:

| Input | Description |
|---|---|
| `TaskExample` artifacts | Processed canonical tasks. |
| `ReasoningState` artifacts | Saved states from initial trajectories. |
| Operation implementations | `stop`, `continue`, `branch`, `verify_check`. |
| Parser/scoring utilities | Domain-specific scoring. |
| Verifier/checker utilities | Math/code/symbolic validation. |
| Config files | Rollout counts, seeds, costs, paths, operation settings. |

Expected natural state inputs:

- `states_train_natural.jsonl`
- `states_val_natural.jsonl`
- `states_test_natural.jsonl`

The implementation should allow configurable paths.

## Outputs

This stage should produce:

| Artifact | Description |
|---|---|
| `counterfactual_outcomes_train.jsonl` | Forced-operation outcomes for training states. |
| `counterfactual_outcomes_val.jsonl` | Forced-operation outcomes for validation states. |
| `counterfactual_outcomes_test_audit.jsonl` | Forced-operation outcomes for test-audit states. |
| `action_values_train.parquet` or `.jsonl` | Cost-aware action values for train states. |
| `action_values_val.parquet` or `.jsonl` | Cost-aware action values for validation states. |
| `action_values_test_audit.parquet` or `.jsonl` | Cost-aware action values for test-audit states. |
| `post_operation_states_train.jsonl` | Sampled post-operation training states. |
| `post_operation_states_val.jsonl` | Sampled post-operation validation states. |
| `post_operation_states_test_audit.jsonl` | Sampled post-operation test-audit states. |
| `counterfactual_collection_report.json` | Machine-readable integrity report. |
| `counterfactual_collection_report.md` | Human-readable summary. |

Use configurable artifact paths.

## Rollout Counts

Force all four operations from each selected state.

### Train and Validation

| Operation | Rollouts per state |
|---|---:|
| `stop` | `1` |
| `continue` | `2` |
| `branch` | `2` |
| `verify_check` | `1` |
| **Total** | **`6`** |

### Test Audit

| Operation | Rollouts per state |
|---|---:|
| `stop` | `1` |
| `continue` | `3` |
| `branch` | `3` |
| `verify_check` | `1` |
| **Total** | **`8`** |

The rollout count should be controlled by config. Operations themselves should not hardcode rollout count.

## Pilot Counts

The decisive Qwen3-14B pilot uses:

| Split | States | Outcomes per state | Outcomes |
|---|---:|---:|---:|
| Train | `2,700` | `6` | `16,200` |
| Validation | `450` | `6` | `2,700` |
| Test audit | `900` | `8` | `7,200` |
| **Total** | **`4,050`** | — | **`26,100`** |

The train/validation/test state counts include the final selected mix of natural and post-operation states. See the post-operation state section below.

## Same-State Requirement

This is a required invariant.

For each `state_id`, every public operation must be evaluated from the same saved state.

The collection code should be able to verify that each state has the expected outcome set:

- stop rollout(s)
- continue rollout(s)
- branch rollout(s)
- verify/check rollout(s)

Missing or duplicated operation outcomes should be reported in the integrity report.

Do not compare operation outcomes from different states.

## Counterfactual Outcome Schema

Every forced operation result should be scored and serialized as a `CounterfactualOutcome`.

Required fields include:

| Field | Description |
|---|---|
| `outcome_id` | Stable unique ID. |
| `state_id` | Source state. |
| `task_id` | Parent task. |
| `split` | Train, validation, or test audit. |
| `domain` | Math, code, or symbolic. |
| `backbone` | Model name. |
| `operation` | `stop`, `continue`, `branch`, or `verify_check`. |
| `operation_subtype` | e.g. `stop_commit`, `stop_elicit`, `self_check`, `tool_check`, `full_check`. |
| `rollout_index` | Index for repeated rollouts. |
| `rollout_seed` | Generation seed. |
| `source_state_hash` | Hash or reference proving same-state origin. |
| `generated_text` | Operation output or reference to stored output. |
| `parsed_answer` | Parsed answer/code/label if available. |
| `correct` | Boolean correctness. |
| `scoreable` | Whether output could be scored. |
| `failure_type` | Parse error, wrong answer, timeout, checker error, etc. |
| `tokens_generated` | New tokens generated by operation. |
| `tool_cost` | Checker/tool cost. |
| `runtime_seconds` | Operation runtime. |
| `checker_metadata` | Domain-specific checker metadata. |
| `operation_metadata` | Operation-specific metadata. |
| `config_hash` | Config used for this outcome. |
| `run_id` | Run identifier. |

## Scoring Outcomes

Every operation output must be scored using the scoring layer.

At minimum, store:

```text
correct ∈ {true, false}
scoreable ∈ {true, false}
failure_type
parsed_answer
tokens_generated
tool_cost
runtime_seconds
