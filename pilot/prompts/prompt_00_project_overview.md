# Prompt 00: CVRO Project Overview and Repository Contract

You are implementing the pilot codebase for **Counterfactual Value of Reasoning Operations (CVRO)**.

This repository should support an end-to-end pilot experiment for studying test-time reasoning control in language models. The immediate target is the **Qwen3-14B pilot**, not the full four-backbone run. The codebase should be designed so that the full run can be added later without rewriting the core pipeline.

## Project Summary

CVRO treats test-time reasoning as a sequence of state-dependent choices among four reasoning operations:

| Operation | Meaning |
|---|---|
| `stop` | Commit to the current answer and terminate reasoning. |
| `continue` | Extend the current reasoning trajectory. |
| `branch` | Start an independent alternative reasoning trajectory. |
| `verify_check` | Audit the current candidate answer or reasoning path using self-checking, deterministic validators, executable tests, symbolic checkers, or similar mechanisms. |

The central object is the **counterfactual value of a reasoning operation** from the same saved intermediate reasoning state.

For a saved state `s`, the pipeline should force each operation from that same state, score the resulting output, measure its cost, and compute operation-specific action values.

The main empirical claim the pilot must test is:

> Intermediate reasoning states contain actionable information about which operation has highest expected value: stop, continue, branch, or verify/check.

The codebase should be built to test this claim against strong alternatives: scalar adaptive compute, prompt/input-only routing, shuffled-state controls, always-action policies, and matched Best-of-N.

## Immediate Pilot Scope

The pilot is Qwen3-14B only.

Pilot target:

| Item | Value |
|---|---:|
| Backbone | `Qwen3-14B` |
| Canonical problems | `1,350` |
| Train / validation / test | `900 / 150 / 300` |
| Domains | `math`, `code`, `symbolic` |
| States per problem | `3` |
| Train/val outcomes per state | `6` |
| Test-audit outcomes per state | `8` |
| Total counterfactual outcomes | `26,100` |
| Robustness prompts | `1,800` |
| Controller seeds | `3` |

The pilot should be large enough to decide whether to scale, not just to smoke-test the pipeline.

## Repository-Level Requirements

Build the repository as a reproducible batch pipeline, not as a collection of notebooks.

The codebase must support:

- deterministic data preparation
- stable task IDs
- stable state IDs
- stable outcome IDs
- explicit config files
- resumable batch jobs
- local dry runs
- Modal-compatible batch execution
- machine-readable artifacts
- clear logging
- failure manifests
- final pilot reporting

Every major stage should read from explicit input artifacts and write explicit output artifacts. Avoid hidden dependencies through in-memory state.

The pipeline should be restartable. If a run is interrupted, rerunning the stage should not silently duplicate rows, overwrite unrelated outputs, or corrupt partially completed artifacts.

## Suggested Repository Layout

Use a layout close to this unless there is a strong reason to change it:

| Path | Purpose |
|---|---|
| `pilot/` | Pilot experiment code and configs. |
| `pilot/prompts/` | Prompt files for implementation guidance. |
| `configs/` | YAML/TOML/JSON configs for data, model, operations, controllers, and pilot runs. |
| `src/` | Core reusable implementation. |
| `src/cvro/` | CVRO package. |
| `data/raw/` | Downloaded or cached raw datasets. |
| `data/processed/` | Unified task splits and processed task artifacts. |
| `artifacts/` | Generated trajectories, states, outcomes, action values, models, and reports. |
| `results/` | Final tables, figures, and pilot summaries. |
| `tests/` | Unit and integration tests. |
| `scripts/` | CLI entry points for common jobs. |
| `modal/` or `deployment/` | Modal-specific job definitions if needed. |

Do not hardcode absolute paths. Make paths configurable.

## Core Schemas

The repo should define shared schemas early. They may be implemented with dataclasses, Pydantic models, TypedDicts, or another robust approach. The exact implementation is up to you, but downstream stages must agree on the same fields and serialization.

At minimum, support these object types:

### `TaskExample`

Represents a canonical task.

Required content:

- stable task ID
- domain: `math`, `code`, or `symbolic`
- source dataset
- original prompt
- gold answer, tests, or validator target
- verifier/checker type
- split: `train`, `val`, or `test`
- difficulty/source metadata where available
- answer format metadata
- allowed tools/checkers
- dataset version/source metadata

### `ReasoningTrajectory`

Represents an initial model attempt.

Required content:

- trajectory ID
- task ID
- backbone name
- prompt used
- raw model output
- token count
- generation settings
- generation seed
- parsed final answer if any
- correctness if immediately scoreable
- candidate answer history if extractable
- logging metadata

### `ReasoningState`

Represents an intermediate state from which operations are forced.

Required content:

- state ID
- task ID
- trajectory ID or parent state ID
- backbone name
- split
- domain
- state source:
  - `natural`
  - `post_continue`
  - `post_branch`
  - `post_verify_failure`
- state position:
  - `early`
  - `middle`
  - `late`
  - or post-operation equivalent
- original prompt
- trace prefix or compact state representation
- tokens used so far
- candidate answer if present
- candidate-present flag
- answer history if available
- parse status
- trace features if available
- confidence/logprob features if available
- checker/verifier state if available
- metadata needed for downstream operations

### `CounterfactualOutcome`

Represents the result of forcing one operation from one state.

Required content:

- outcome ID
- state ID
- task ID
- backbone name
- operation:
  - `stop`
  - `continue`
  - `branch`
  - `verify_check`
- operation subtype where applicable:
  - `stop_commit`
  - `stop_elicit`
  - `self_check`
  - `tool_check`
  - `full_check`
- rollout seed
- prompt/template used
- generated text or result object
- parsed answer/code/label
- correctness
- generated token cost
- checker/tool cost
- invalid-output flag
- parse/check failure type
- timing metadata
- config hash or run ID

### `ActionValue`

Represents computed operation values for a state.

Required content:

- state ID
- operation
- mean correctness over rollouts
- mean generated-token cost
- mean tool/checker cost
- invalid-output rate
- cost-aware Q-value
- oracle-best operation for the chosen cost setting
- lambda/cost configuration used

## Methodological Invariants

The codebase must enforce the following constraints.

### Same-State Counterfactuals

For a given saved `ReasoningState`, all four operations must be forced from that same state. The action-value comparison is invalid if different operations are evaluated from different states.

### Online-Observable State Extraction

State extraction must not depend on information that would be unavailable at runtime. Do not define checkpoints using hindsight knowledge of the final natural trajectory length unless the runtime system can observe the same information.

### Branch Isolation

The `branch` operation must not receive the full prior reasoning trace. It may receive the original prompt and optionally the current candidate answer, but not the complete trace prefix. Otherwise branch becomes contaminated continuation.

### Continue Preservation

The `continue` operation should extend the current trace. It should not restart from scratch.

### Verify/Check Transparency

`verify_check` must record whether it used model-only self-checking, deterministic/tool checking, or a combined full version. Tool/checker costs must be counted.

### Stop Subtypes

`stop` must distinguish committing to an existing candidate from eliciting a final answer from a partial trace.

### Test-Label Isolation

Test counterfactual labels are for audit and regret analysis only. They must not be used for controller training, hyperparameter tuning, prompt tuning, lambda selection, feature selection, or baseline calibration.

### Matched-Cost Evaluation

Baselines must report actual generated tokens, checker/tool cost, invalid-output rate, and normalized total cost. Best-of-N and verifier baselines must be budget-matched as strictly as possible.

### Reproducibility

Every artifact should be traceable to:

- config file
- model/backbone
- dataset split
- random seed
- operation template version
- parser/checker version
- run ID

## Pilot Gates

The final pilot report should decide whether the project is ready to scale.

The pilot should report at least these gates:

| Gate | Target |
|---|---:|
| State extraction failure rate | `<= 3%` |
| Parser failure rate overall | `<= 5%` |
| Broken verifier/checker rate | `<= 3%` |
| CVRO runtime loop failure | `<= 3%` |
| Budget matching error | `<= 5%` |
| Stop oracle-best share | `>= 10%` |
| Continue oracle-best share | `>= 10%` |
| Branch oracle-best share | `>= 8%` |
| Verify/check oracle-best share | `>= 8%` |
| Largest oracle-best action share | `<= 55%` |
| CVRO vs scalar adaptive compute | positive at matched cost |
| CVRO vs prompt/input-only | positive |
| CVRO vs shuffled-state | positive |
| CVRO vs matched Best-of-N verify | positive or substantially cheaper at similar accuracy |

The exact report generation is handled later, but the codebase should be built so these quantities are measurable.

## Modal and Execution Requirements

The repository should be deployable for pilot execution on Modal or a similar batch environment.

Support:

- GPU batch inference jobs
- CPU verification/checking jobs
- sharded processing
- resume/retry behavior
- artifact aggregation
- local dry-run mode
- config-driven execution

Do not assume a notebook environment. Do not require manual copying of intermediate outputs.

A local dry run should be able to process a tiny subset, such as 3–10 examples, without requiring the full Modal setup.

## Testing Expectations

Add tests for the parts where silent errors would invalidate the experiment.

At minimum, the repo should eventually include tests for:

- task schema serialization/deserialization
- parser behavior on known examples
- math equivalence checks
- code extraction and sandbox handling
- symbolic label parsing
- state extraction boundary behavior
- operation prompt construction
- branch not receiving full prior trace
- same-state outcome grouping
- Q-value computation
- train/validation/test isolation
- shuffled-state baseline actually shuffling state-example alignment

Tests do not need to be exhaustive at the first commit, but the repo structure should make them natural to add.

## What Not To Do

Do not implement this as notebooks only.

Do not use test counterfactual outcomes for tuning.

Do not let `branch` see the full trace.

Do not let `continue` restart from scratch.

Do not treat verifier/checker calls as free.

Do not manually repair model outputs after generation.

Do not silently drop failed examples without logging why.

Do not hardcode API keys, absolute paths, local usernames, or secrets.

Do not build a large transformer controller. The controller should be lightweight.

Do not expand beyond the four public operations in the pilot.

Do not optimize for leaderboard performance at the cost of methodological clarity.

## Definition of Done for This Prompt

After completing this prompt, the repository should have:

- a clear project skeleton
- basic config structure
- shared schema definitions or placeholders
- artifact directory conventions
- logging/run-ID conventions
- local dry-run conventions
- Modal/deployment placeholder or plan
- README or developer notes explaining how later stages fit together
- no implementation choices that violate the methodological invariants above

This prompt is the global contract for the rest of the pilot implementation. Later prompts will fill in data loading, parsers/verifiers, state extraction, operations, counterfactual collection, controllers, pilot orchestration, and reporting.
