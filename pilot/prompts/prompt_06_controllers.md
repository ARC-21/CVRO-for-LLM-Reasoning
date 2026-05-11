# Prompt 06: Controllers, Feature Extraction, and Dangerous Control Baselines

You are implementing the controller-training layer for the CVRO pilot.

This stage trains lightweight operation-selection controllers from counterfactual action-value artifacts. It also implements the dangerous controller baselines needed to test whether CVRO is genuinely using intermediate reasoning states rather than merely doing prompt-level routing or scalar adaptive compute.

Do not implement dataset loading, parsers, initial trajectory generation, CVRO operation execution, counterfactual collection, or final reporting in this prompt. Use the artifacts produced by previous stages.

## Context

CVRO trains a small controller to select among four operations from an intermediate reasoning state:

1. `stop`
2. `continue`
3. `branch`
4. `verify_check`

The controller should learn from cost-aware counterfactual action values:

    Q_lambda(s, a)

where `s` is a saved reasoning state and `a` is one of the four operations.

The main claim tested by the controller is:

> Intermediate reasoning states contain actionable information about which operation has highest expected value.

Therefore, the controller system must include strong negative and competing baselines:

- scalar adaptive compute
- prompt-only controller
- input-only controller
- shuffled-state controller
- linear/simple controller baselines
- remove-action ablations

## Inputs

This stage should consume:

| Input | Description |
|---|---|
| `ReasoningState` artifacts | Natural and post-operation states. |
| `ActionValue` artifacts | Q-values and oracle actions from counterfactual collection. |
| `TaskExample` artifacts | Task metadata and prompts. |
| Counterfactual outcome summaries | Optional additional rollout/cost/failure features. |
| Config files | Feature configuration, controller hyperparameters, seeds, lambda settings. |

Expected artifacts include:

- `action_values_train.*`
- `action_values_val.*`
- `action_values_test_audit.*`
- `states_train*`
- `states_val*`
- `states_test*`
- processed task files

Use configurable paths.

## Outputs

This stage should produce:

| Artifact | Description |
|---|---|
| `features_train.*` | Extracted controller features for train states. |
| `features_val.*` | Extracted controller features for validation states. |
| `features_test_audit.*` | Extracted controller features for test-audit states. |
| `controller_models/` | Saved trained controller models and metadata. |
| `controller_predictions_val.*` | Predicted Q-values/actions on validation states. |
| `controller_predictions_test_audit.*` | Predicted Q-values/actions on test-audit states. |
| `controller_training_report.json` | Machine-readable training/evaluation summary. |
| `controller_training_report.md` | Human-readable training/evaluation summary. |

The final pilot runner will later use these controllers for runtime policy evaluation.

## Controller Feature Extraction

Implement a feature extraction layer that turns each `ReasoningState` into model-consumable features.

The exact feature implementation is up to you, but it must support feature groups that can be enabled/disabled by config for ablations.

### Required Feature Groups

| Feature group | Examples |
|---|---|
| Position features | State position, source type, tokens used, decision index if available. |
| Domain/source metadata | Domain, dataset source, task type, difficulty metadata. |
| Prompt features | Prompt length, answer format, task metadata. |
| Trace structure features | Trace length, sentence count, paragraph count, code block count, equation count. |
| Candidate features | Candidate present, candidate type, parse status, candidate length. |
| Answer-history features | Number of candidates, answer changed flag, repeated candidate, candidate disagreement. |
| Uncertainty features | Logprob, entropy, margin, proxy confidence if available. |
| Stability features | Self-correction phrases, uncertainty phrases, contradiction markers, repetition indicators, answer flips. |
| Verifier/checker-state features | Checker available, previous checker result, failure type, test pass/fail info if available. |
| Branch-disagreement features | Original candidate vs branch candidate agreement, branch score metadata if available. |
| Cost-progress features | Tokens used so far, fraction of planned budget, prior operation costs if available. |

Not all features will be available for every state. Missingness should be explicit and handled robustly.

### Feature Constraints

The main controller must use only features available at runtime.

Do not include:

- gold labels
- final correctness of the natural trajectory
- test-audit action values
- oracle action labels as input features
- future operation outcomes
- full counterfactual outcome summaries that would not exist at runtime

It is acceptable to include prior checker results only if the state is a post-verify state or if such checker results would actually be available at runtime.

## Feature Ablation Support

Feature extraction must support ablations.

Required ablation configurations:

| Ablation | Description |
|---|---|
| `full_state` | All allowed runtime state features. |
| `prompt_only` | Original prompt text/metadata only; no intermediate trace/candidate/state features. |
| `input_only` | Prompt plus domain/source/difficulty metadata, but no intermediate reasoning state. |
| `no_trace_features` | Remove trace structure/text-derived features. |
| `no_uncertainty_features` | Remove confidence/logprob/entropy/proxy uncertainty features. |
| `no_verifier_features` | Remove checker/verifier state features. |
| `no_answer_history_features` | Remove candidate history and answer-change features. |
| `shuffled_state` | Break state-example alignment while preserving marginal feature distributions. |

The distinction between `prompt_only` and `input_only` must be explicit. If they collapse to the same implementation, document that and expose one as an alias.

## Text Features

The main pilot should not depend on a large trainable text encoder.

Allowed:

- scalar/handcrafted features
- categorical features
- optional frozen embeddings if implemented cleanly
- bag-of-indicators for trace phrases
- lightweight text statistics

Do not fine-tune a transformer encoder as the main controller. That would weaken the methodological claim that the useful signal is visible in lightweight state features.

## Controller Targets

The primary target is Q-value regression.

For each state, the target contains four values:

```text
Q(stop), Q(continue), Q(branch), Q(verify_check)
