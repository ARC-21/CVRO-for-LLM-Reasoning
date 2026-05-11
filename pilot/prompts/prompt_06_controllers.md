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

    Q(stop), Q(continue), Q(branch), Q(verify_check)

Also compute the oracle-best action:

    oracle_action = argmax_a Q(s, a)

Do not train only a four-way classifier. If two actions are close, the argmax label can be noisy. Preserve value margins.

Recommended training objective for the main MLP:

    Loss =
        Huber(Q_pred, Q_observed)
        + beta * CE(argmax(Q_observed), softmax(Q_pred))

`beta` should be selected using validation data only.

## Controllers to Implement

Implement at least the following.

### 1. Linear / Logistic Baseline

Purpose:

- interpretable baseline
- checks whether signal is simple
- sanity check against overpowered controller

It may be implemented as:

- linear regression over Q-values
- multinomial logistic regression over oracle actions
- or both

### 2. Small MLP Main Controller

Purpose:

- main CVRO operation-selection controller

Requirements:

- lightweight
- fixed small architecture from config
- outputs predicted Q-values for the four operations
- supports 3 pilot controller seeds
- saves model checkpoints and prediction artifacts

Do not make this a large opaque model.

### 3. Scalar Adaptive Compute Baseline

This is one of the most important baselines.

It should receive comparable state features but must not choose among operation types.

It should predict only a scalar compute decision such as:

- stop/continue length
- token budget
- short/medium/long reasoning budget
- or another scalar budget schedule

It cannot choose branch or verify/check as separate operation types.

Purpose:

> Test whether CVRO is more than adaptive token-budget allocation.

The scalar baseline should be as strong as reasonably possible within its restricted action space.

### 4. Prompt-Only / Input-Only Controller

Purpose:

> Test whether intermediate reasoning state matters.

Prompt-only should use only the original prompt, with no intermediate trace.

Input-only may include prompt plus metadata such as domain/source/difficulty, but no intermediate reasoning state. If prompt-only and input-only become identical in implementation, document the equivalence.

### 5. Shuffled-State Controller

Purpose:

> Test whether state features are aligned with the correct task/state rather than acting as generic priors.

Implementation requirement:

- preserve feature distributions
- break state-example alignment
- keep split integrity
- do not accidentally leak target labels

The shuffled controller should be trained/evaluated under the same controller class and seed setup where feasible.

### 6. Remove-Action Controllers

Support ablations where the controller cannot choose one operation.

Required remove-action settings:

| Ablation | Allowed actions |
|---|---|
| `no_stop` | continue, branch, verify/check |
| `no_continue` | stop, branch, verify/check |
| `no_branch` | stop, continue, verify/check |
| `no_verify_check` | stop, continue, branch |

At inference/prediction time, unavailable actions should be masked out before selecting the predicted-best action.

These ablations test whether each operation matters.

## Action Selection

Given predicted Q-values, choose the action with the highest predicted value among allowed actions.

The prediction artifact should include:

- predicted Q-values for all operations
- selected action
- action ranking
- top-2 margin
- masked actions if any
- controller name
- controller seed
- feature configuration
- lambda/cost setting

## Training Seeds

For the pilot, use **3 controller seeds**.

Every trained controller should record:

- seed
- feature config
- model config
- lambda/cost setting
- train/val artifact hashes
- run ID
- training metrics
- validation metrics

The full run may later use 5 seeds, but the pilot target is 3.

## Data Splits and Leakage Rules

Use:

- train action values for fitting
- validation action values for hyperparameter selection
- test-audit action values only for final audit/regret analysis

Do not use test-audit labels for:

- model training
- early stopping
- hyperparameter tuning
- feature selection
- lambda selection
- threshold selection

This should be enforced structurally if possible.

## Metrics to Compute

For validation and test-audit predictions, compute:

| Metric | Description |
|---|---|
| Operation-selection accuracy | Selected action equals oracle action. |
| Top-2 accuracy | Selected action is among top two observed Q-values. |
| Regret | Oracle Q-value minus selected-action Q-value. |
| Mean predicted-selected Q | Average observed Q of selected actions. |
| Action distribution | Frequency of selected stop/continue/branch/verify_check. |
| Per-domain metrics | Metrics by math/code/symbolic. |
| Per-state-position metrics | Metrics by early/middle/late/post-operation source. |
| Margin calibration | Relationship between predicted margin and regret if feasible. |
| Invalid-action rate | Should be zero after masking. |

The final pilot runner will evaluate end-to-end accuracy/cost; this stage should focus on action prediction and regret over existing action values.

## Cost Settings and Lambda Handling

The action-value artifacts may include multiple lambda/cost settings.

The controller should support:

- training for a default lambda
- evaluating across a lambda sweep if available
- selecting main lambda using validation only

Do not tune lambda on test-audit data.

Store the lambda/cost setting used in every model and prediction artifact.

## Integration With Runtime Policy

The controller should expose an interface that the later pilot runner can call at inference time.

It should be able to take a current `ReasoningState` and return:

- predicted Q-values
- selected operation
- action ranking
- confidence/margin
- feature vector metadata

This runtime interface should use the same feature extraction path as training.

Avoid duplicating feature logic between training and inference.

## Local Dry Run

Support a local dry run using tiny mock or sample action-value artifacts.

The dry run should:

- load small state/action-value files
- extract features
- train a tiny controller
- save predictions
- compute basic metrics
- run at least one remove-action ablation
- run prompt-only or shuffled-state control if possible

## Validation Checks

Add checks for:

- every feature row maps to a valid state
- every target row maps to a valid state
- no duplicate state IDs within a split
- train/val/test separation
- test-audit labels not used for fitting
- feature ablation configs actually remove intended feature groups
- shuffled-state baseline actually shuffles alignment
- remove-action masks actually prevent selecting removed actions
- prediction artifacts include all four Q-values
- selected action is always in allowed action set
- metrics are computed over task/state units correctly

## Artifact Integrity

Saved model and prediction artifacts should include:

- config hash
- feature schema version
- controller type
- controller seed
- lambda setting
- train/val artifact hashes
- timestamp/run ID

If feature schemas change, stale models should not silently load as valid.

## Do Not Do

Do not fine-tune the base language model.

Do not train a large transformer controller.

Do not use test-audit action values for tuning.

Do not include gold correctness or oracle labels as input features.

Do not leak future operation outcomes into runtime features.

Do not let scalar adaptive compute choose branch or verify/check as distinct operations.

Do not let prompt-only/input-only see intermediate trace features.

Do not silently ignore missing features without recording missingness.

Do not collapse all training to oracle-action classification only.

## Definition of Done

This prompt is complete when the repository can:

1. load state and action-value artifacts,
2. extract runtime-valid controller features,
3. train a linear/simple baseline,
4. train the main small MLP controller,
5. train or run scalar adaptive compute baseline,
6. run prompt-only/input-only controls,
7. run shuffled-state control,
8. run remove-action ablations,
9. produce validation and test-audit predictions,
10. compute action-selection and regret metrics,
11. save model/prediction artifacts with config metadata,
12. support local dry runs.

The controller layer must be ready for the pilot orchestration stage, where controllers will be evaluated end-to-end against dangerous baselines.
