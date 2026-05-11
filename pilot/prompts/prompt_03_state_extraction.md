# Prompt 03: Initial Trajectories and Online-Observable State Extraction

You are implementing the initial trajectory generation and intermediate-state extraction layer for the CVRO pilot.

This stage takes processed `TaskExample` artifacts, generates initial reasoning trajectories with the pilot backbone, and extracts online-observable intermediate `ReasoningState` objects. These states will later be used for same-state counterfactual operation collection.

Do not implement the four CVRO operations, counterfactual collection, controller training, baseline evaluation, or reporting in this prompt. Focus on initial trajectories and state extraction.

## Context

CVRO estimates the value of operations from intermediate reasoning states. Those states must be extractable during live inference. Therefore, state extraction must be **online-observable** and must not depend on hindsight knowledge of the full final trajectory length unless an equivalent signal is available at runtime.

The pilot backbone is:

| Item | Value |
|---|---|
| Backbone | `Qwen3-14B` |
| Pilot tasks | `1,350` |
| Split | `900 / 150 / 300` |
| Domains | `math`, `code`, `symbolic` |
| States per task | `3` |

The goal of this stage is to produce initial trajectories and three states per task:

1. `early`
2. `middle`
3. `late`

## Inputs

This stage should consume processed task artifacts from the data pipeline, such as:

- `data/processed/tasks_train.jsonl`
- `data/processed/tasks_val.jsonl`
- `data/processed/tasks_test.jsonl`
- `data/processed/tasks_all.jsonl`
- `data/processed/task_manifest.json`

It should also use parser/scoring utilities from the verifier/parser stage where useful, especially for detecting answer candidates.

## Outputs

This stage should produce trajectory and state artifacts.

Required outputs:

| Artifact | Content |
|---|---|
| `trajectories_train.jsonl` | Initial model trajectories for train tasks. |
| `trajectories_val.jsonl` | Initial model trajectories for validation tasks. |
| `trajectories_test.jsonl` | Initial model trajectories for test tasks. |
| `states_train_natural.jsonl` | Extracted natural early/middle/late train states. |
| `states_val_natural.jsonl` | Extracted natural early/middle/late validation states. |
| `states_test_natural.jsonl` | Extracted natural early/middle/late test-audit states. |
| `state_extraction_report.json` | Counts, failures, and extraction diagnostics. |
| `state_extraction_report.md` | Human-readable summary. |

Use configurable artifact paths. The filenames above are suggested defaults.

## Initial Trajectory Generation

Generate one initial natural reasoning trajectory per task for the pilot backbone.

The trajectory should be a normal solution attempt, not one of the CVRO operations.

Use domain-specific solve prompts, but keep them simple and consistent.

### Math Trajectory Prompt

The model should solve step by step and end with a final answer marker.

Required behavior:

- allow reasoning
- require final answer format
- avoid operation-specific wording

Example intent:

    Solve the problem. Show your reasoning. End with:
    Final answer: <answer>

### Code Trajectory Prompt

The model should produce a solution explanation and code.

Required behavior:

- request final Python code
- enforce a code block if possible
- avoid asking for verification yet

Example intent:

    Solve the programming problem. Provide a short explanation and then the final Python code in one code block.

### Symbolic Trajectory Prompt

The model should reason from the given facts/rules and end with a final answer.

Required behavior:

- use the provided facts/rules
- end with final answer marker
- avoid subjective interpretation

Example intent:

    Solve the reasoning problem using the provided facts and rules. End with:
    Final answer: <answer>

The exact prompt templates should be stored as versioned prompt files or config entries. Do not hardcode them deep in the code.

## Trajectory Metadata

Each `ReasoningTrajectory` should store enough information for downstream reproducibility.

Required fields include:

| Field | Description |
|---|---|
| `trajectory_id` | Stable unique identifier. |
| `task_id` | Parent task ID. |
| `split` | Train, val, or test. |
| `domain` | Math, code, or symbolic. |
| `backbone` | Model name. |
| `prompt_template_id` | Solve prompt version. |
| `prompt_text` | Actual prompt sent to the model or reference to it. |
| `raw_output` | Full model output. |
| `tokens_generated` | Generated token count. |
| `generation_seed` | Decoding seed. |
| `decoding_config` | Temperature, top-p, max tokens, stop sequences, etc. |
| `parsed_final_answer` | Parsed final answer/code/label if available. |
| `initial_correct` | Correctness if immediately scoreable. |
| `candidate_answers` | Candidate answers found in the trace, if any. |
| `generation_metadata` | Runtime, model backend, errors, retry info. |

## Recommended Decoding Settings

Use config-driven decoding settings. Suggested pilot defaults:

| Domain | Max tokens | Temperature | top_p |
|---|---:|---:|---:|
| Math | `1024` | `0.7` | `0.95` |
| Code | `1536` | `0.7` | `0.95` |
| Symbolic | `768` | `0.7` | `0.95` |

These should be configurable. The pilot may finalize them before the full run. Log the exact values used.

## Online-Observable State Extraction

Extract exactly three states per successful trajectory where possible:

| State | Meaning |
|---|---|
| `early` | Early partial reasoning after the model has begun a nontrivial solution. |
| `middle` | Intermediate reasoning after additional progress, before final commitment when possible. |
| `late` | First candidate-answer trigger or final planned checkpoint. |

The extraction must be usable during runtime. Do not select states using information that would only be known after observing the full final answer trajectory, except for offline diagnostics.

## State Extraction Rules

Use deterministic, online-observable rules.

### Early State

Extract after the first completed reasoning unit at or after an initial token threshold.

Suggested thresholds:

| Domain | Initial threshold |
|---|---:|
| Math | `128-192` tokens |
| Code | `192-256` tokens |
| Symbolic | `128-160` tokens |

If a completed reasoning unit appears before the threshold, continue until the threshold region and then cut at the next boundary.

### Middle State

Extract after the second reasoning chunk or around the planned budget midpoint.

Important: do not define middle as “50% of the final generated trajectory length” unless you also implement the same planned-budget checkpoint at runtime.

The middle checkpoint should be based on an online schedule, such as:

- after a second fixed chunk, or
- after approximately 40-60% of the planned maximum reasoning budget, or
- after a second completed reasoning unit following the early state.

### Late State

Extract at the first explicit candidate-answer trigger if one appears.

Candidate-answer triggers include:

- `Final answer:`
- `Answer:`
- `Therefore, the answer is`
- code block completion for code tasks
- explicit label statement for symbolic tasks
- domain-specific answer markers recognized by the parser

If no answer trigger appears, use the final planned checkpoint before budget exhaustion or the last complete reasoning unit within the trajectory.

## Completed Reasoning Units

Define domain-aware boundaries.

| Domain | Reasoning-unit boundaries |
|---|---|
| Math | sentence boundary, equation block boundary, paragraph boundary, final-answer marker |
| Code | explanation paragraph boundary, code block boundary, function boundary, final code block |
| Symbolic | sentence boundary, rule-application boundary, proof-step boundary, paragraph boundary |

Avoid cutting inside:

- LaTeX expressions
- code blocks
- markdown fences
- incomplete list items
- partial equations
- incomplete JSON-like structures
- incomplete final-answer markers

If an ideal boundary cannot be found, use a deterministic fallback and record the fallback type.

## State Object Requirements

Each `ReasoningState` should include:

| Field | Description |
|---|---|
| `state_id` | Stable unique state identifier. |
| `task_id` | Parent task ID. |
| `trajectory_id` | Parent trajectory ID. |
| `split` | Train, val, or test. |
| `domain` | Math, code, symbolic. |
| `backbone` | Model name. |
| `state_source` | `natural` for this stage. |
| `state_position` | `early`, `middle`, or `late`. |
| `prompt` | Original task prompt or reference. |
| `trace_prefix` | Reasoning text up to checkpoint. |
| `tokens_used` | Tokens used up to checkpoint. |
| `candidate_answer` | Current candidate answer/code/label if present. |
| `candidate_present` | Boolean. |
| `answer_history` | List of candidate answers found so far, if any. |
| `parse_status` | Current parse status. |
| `trace_features` | Basic features extracted from trace. |
| `confidence_features` | Logprob/entropy/margin if available; otherwise null. |
| `checker_state` | Any existing checker result if available; usually null at this stage. |
| `difficulty_metadata` | Copied from task. |
| `extraction_metadata` | Boundary type, fallback used, trigger used, errors. |

## Basic Trace Features

Extract lightweight features if feasible. Do not build a large feature system here, but include useful metadata for later controllers.

Examples:

- token count
- character count
- number of sentences/paragraphs
- number of equations
- number of code blocks
- answer marker present
- candidate count so far
- candidate changed flag if multiple candidates
- self-correction phrases
- uncertainty phrases
- repetition indicators
- contradiction markers
- parseability flag

The controller prompt will later expand feature extraction if needed. This stage should store enough raw information to compute features later, even if not all features are computed immediately.

## Candidate Answer Detection

Use parser utilities where available.

The state extractor should detect whether the current trace prefix already contains a candidate answer. It does not need perfect final scoring, but it should record:

- candidate text
- candidate type
- candidate source marker
- whether multiple candidates exist
- last candidate before checkpoint

If no candidate exists, store `candidate_present = false`.

## Failure Handling

Do not silently drop tasks.

For every trajectory, try to extract three states. If extraction fails, write a structured failure record.

Failure types may include:

| Failure type | Meaning |
|---|---|
| `empty_output` | Model generated no usable output. |
| `too_short` | Output too short to extract all checkpoints. |
| `no_boundary` | No safe boundary found. |
| `parser_error` | Candidate/parser utilities failed unexpectedly. |
| `tokenization_error` | Token offsets could not be computed. |
| `model_error` | Inference failed. |
| `unknown_error` | Unexpected failure. |

The state extraction report should include failure rates by domain/source/split.

## Local Dry Run

Implement a dry run that can process a tiny subset, such as 3-10 tasks, without requiring the full pilot.

The dry run may use:

- mock model outputs, or
- a tiny local model, or
- a limited API/model call if configured

But it must exercise the same state extraction code path and produce schema-valid trajectory/state artifacts.

## Modal / Batch Execution

Design this stage so it can run as a batch job on Modal or a similar environment.

It should support:

- task sharding
- resumable generation
- retry on transient model errors
- output deduplication by trajectory/state ID
- configurable model backend
- local dry run
- artifact aggregation

Do not assume all tasks fit into one process.

## Validation Checks

Add validation checks for:

- exactly one trajectory per task for the configured run
- up to three states per trajectory
- state IDs unique
- state positions present where possible
- no state has empty prompt
- no state has empty trace prefix unless explicitly allowed
- tokens used are nonnegative and monotonic by state position
- extracted states do not exceed trajectory length
- branch-specific constraints are not relevant here but the state object should not include future operation outputs
- extraction failure rate reported

The pilot gate later requires state extraction failure rate to be at most 3%.

## Artifact Integrity

Each output artifact should include or reference:

- config hash
- run ID
- model/backend identifier
- prompt template version
- dataset manifest hash
- timestamp
- shard ID where applicable

This makes it possible to reproduce or audit later counterfactual outcomes.

## Do Not Do

Do not implement CVRO operations in this prompt.

Do not force stop/continue/branch/verify/check here.

Do not compute counterfactual action values here.

Do not train controllers here.

Do not use future test counterfactual labels.

Do not choose states based on knowing which action later succeeds.

Do not manually edit trajectories or states.

Do not let state extraction depend on final correctness.

Do not silently discard failed examples without logging.

## Definition of Done

This prompt is complete when the repository can:

1. load processed pilot tasks,
2. generate or mock initial reasoning trajectories,
3. extract online-observable early/middle/late states,
4. serialize trajectory and state artifacts,
5. produce extraction diagnostics,
6. run a local dry run,
7. support sharded/resumable execution for the pilot backbone,
8. validate that state artifacts conform to the shared schema.

The resulting `ReasoningState` artifacts must be ready for the next stage, where stop, continue, branch, and verify/check will be forced from the same saved states.
