# Prompt 01: Data Pipeline and Canonical Pilot Split

You are implementing the data pipeline for the CVRO pilot.

The goal of this prompt is to build the dataset ingestion, normalization, filtering, and split-generation layer for the **Qwen3-14B pilot**. This stage should produce a unified set of canonical task examples that later stages can use for trajectory generation, state extraction, counterfactual operation collection, controller training, and evaluation.

Do not implement model inference, operation execution, controller training, or reporting in this prompt. Focus on producing clean, reproducible, verifiable task artifacts.

## Context

CVRO needs tasks where model outputs can be scored reliably. The pilot uses three domains:

1. `math`
2. `code`
3. `symbolic`

The pilot target is:

| Split | Problems |
|---|---:|
| Train | `900` |
| Validation | `150` |
| Test | `300` |
| **Total** | **`1,350`** |

Balanced across domains:

| Domain | Train | Validation | Test | Total |
|---|---:|---:|---:|---:|
| Math | `300` | `50` | `100` | `450` |
| Code | `300` | `50` | `100` | `450` |
| Symbolic | `300` | `50` | `100` | `450` |
| **Total** | **`900`** | **`150`** | **`300`** | **`1,350`** |

The full run will later use larger splits, but the immediate implementation target is the pilot split above.

## Datasets to Support

Implement dataset adapters for these sources where feasible. The code should be modular so unsupported sources can be skipped gracefully during local dry runs, but the intended pilot should support enough sources to hit the target counts.

### Math Sources

Use:

- `MATH`
- `GSM8K`, filtered toward harder or multi-step examples where possible

Pilot math allocation target:

| Source | Train | Validation | Test | Total |
|---|---:|---:|---:|---:|
| MATH | `250` | `40` | `80` | `370` |
| GSM8K hard/filtered | `50` | `10` | `20` | `80` |
| **Total** | **`300`** | **`50`** | **`100`** | **`450`** |

The exact source counts may vary slightly if data availability or filtering requires it, but the final pilot split must remain balanced at 450 math examples.

### Code Sources

Use:

- `APPS`
- `LiveCodeBench`
- `MBPP`
- `HumanEval`

Pilot code allocation target:

| Source | Train | Validation | Test | Total |
|---|---:|---:|---:|---:|
| APPS | `230` | `35` | `55` | `320` |
| LiveCodeBench | `30` | `10` | `25` | `65` |
| MBPP | `30` | `5` | `10` | `45` |
| HumanEval | `10` | `0` | `10` | `20` |
| **Total** | **`300`** | **`50`** | **`100`** | **`450`** |

The code examples must have runnable or checkable tests. Do not admit code tasks that cannot be scored by later verifier code.

### Symbolic Sources

Use:

- `PrOntoQA`
- `CLUTRR`
- `ProofWriter`
- selected BBH-style symbolic or logical tasks

Pilot symbolic allocation target:

| Source | Train | Validation | Test | Total |
|---|---:|---:|---:|---:|
| PrOntoQA | `120` | `20` | `40` | `180` |
| CLUTRR | `75` | `12` | `25` | `112` |
| ProofWriter | `75` | `13` | `25` | `113` |
| Selected BBH-style tasks | `30` | `5` | `10` | `45` |
| **Total** | **`300`** | **`50`** | **`100`** | **`450`** |

Use only symbolic tasks with deterministic labels or validators. Avoid tasks where correctness depends on subjective commonsense judgments.

## Unified Task Schema

All processed examples should conform to a single `TaskExample` schema defined by the project-level schemas.

Each task must include at least:

| Field | Description |
|---|---|
| `task_id` | Stable unique identifier. |
| `domain` | `math`, `code`, or `symbolic`. |
| `source` | Dataset source name. |
| `source_id` | Original dataset ID/index if available. |
| `split` | `train`, `val`, or `test`. |
| `prompt` | Canonical prompt to give the model. |
| `gold` | Gold answer, tests, label, or validator target. |
| `verifier_type` | Type of checker needed downstream. |
| `answer_format` | Expected final-answer/code/label format. |
| `difficulty_metadata` | Difficulty, topic, proof depth, task type, or source metadata where available. |
| `allowed_tools` | Checkers/tools allowed for this task. |
| `source_metadata` | Dataset version, URL/name, config, original split, license info if available. |
| `filter_metadata` | Any filtering decisions or transformations applied. |

Use stable deterministic IDs. A recommended pattern is:

`{domain}:{source}:{normalized_source_id}`

If source IDs are not stable, derive deterministic hashes from canonical task content and source metadata.

## Canonical Prompt Format

The data pipeline should produce prompts that are clear but not operation-specific. Operation prompts are handled later.

For this stage, the canonical task prompt should include the problem statement and any necessary input/output specification.

Do not include chain-of-thought instructions here unless they are part of the source task. The trajectory-generation stage will wrap tasks with solve prompts.

## Filtering Rules

Filtering should be deterministic and logged.

### General Filtering

Exclude examples if:

- the prompt is empty or malformed
- the gold answer/test/label is missing
- the example cannot be scored by any planned verifier/checker
- the prompt requires images, diagrams, web access, files, or non-text information unavailable to the model
- the answer format is too ambiguous to parse later
- the task is duplicated after normalization
- the task is too large for pilot inference budgets

### Math Filtering

Exclude math tasks if:

- the final answer is ambiguous or missing
- the solution requires a diagram not present in text
- the expected answer has multiple equally valid forms that cannot be normalized
- the answer depends on long explanatory proof rather than a checkable final result
- the problem is trivial enough that it is unlikely to require reasoning control, if this can be detected without using model outputs

Keep tasks with:

- numeric answers
- symbolic expressions
- fractions
- equations
- finite sets/tuples where normalization is feasible
- multi-step word problems
- competition-style algebra, number theory, counting/probability, and text-only geometry where scoreable

### Code Filtering

Exclude code tasks if:

- no tests or checker are available
- the problem requires nonstandard packages
- the problem requires network access, file-system side effects, GUI interaction, or external services
- the expected runtime is too high for sandboxed checking
- input/output specification is unclear
- tests are known to be flaky or insufficient to distinguish correctness

Keep tasks with:

- function-level Python solutions
- clear input/output format
- public tests, hidden tests, or generated tests
- deterministic execution behavior
- manageable runtime

### Symbolic Filtering

Exclude symbolic tasks if:

- label space is unclear
- correctness is subjective
- the example requires external knowledge beyond supplied facts/rules
- no deterministic label/checker can be attached
- entity/relation labels cannot be normalized

Keep tasks with:

- deterministic True/False/Unknown labels
- relation labels
- proof-depth metadata
- rule/fact structures
- generated ground truth
- programmatically checkable answers

## Split Generation

Create one deterministic pilot split.

Requirements:

- use a fixed split seed
- maintain domain balance
- maintain approximate source allocations
- avoid duplicate or near-duplicate examples across splits where detectable
- keep source IDs and task IDs stable
- write a manifest with counts by domain/source/split
- log excluded examples and exclusion reasons

Do not create multiple random splits for the pilot. Use one stable split.

## Output Artifacts

Write processed artifacts to a configurable output directory, defaulting to `data/processed/`.

Required outputs:

| Artifact | Content |
|---|---|
| `tasks_train.jsonl` | 900 training tasks. |
| `tasks_val.jsonl` | 150 validation tasks. |
| `tasks_test.jsonl` | 300 test tasks. |
| `tasks_all.jsonl` | All 1,350 pilot tasks. |
| `task_manifest.json` | Counts, config, seed, source versions, hashes. |
| `excluded_examples.jsonl` | Examples removed during filtering, with reasons. |
| `source_download_manifest.json` | Dataset source paths, versions, dates, configs, or hashes. |

If Parquet is also useful, add it, but JSONL should exist for easy inspection.

## Robustness Variant Placeholder

Do not fully implement robustness generation in this prompt unless it is natural to do so. However, the task schema should leave room for later linking robustness variants to canonical tasks.

The later robustness stage needs to know:

- canonical task ID
- variant type
- whether expected answer should stay the same or change
- validator target for the variant

Make sure the task schema can support this later without refactoring.

## Local Dry Run

Implement a dry-run mode that creates a tiny processed split, such as:

| Split | Count |
|---|---:|
| Train | 6 |
| Validation | 3 |
| Test | 3 |

The dry run may use mock examples if real dataset download is unavailable, but the mock examples must conform to the same schema and verifier-type expectations.

The dry run should be enough for downstream parser and state-extraction code to execute basic tests.

## Reproducibility Requirements

The data pipeline must record:

- dataset source
- version/config/date where available
- raw file path or cache path
- processing config
- split seed
- filtering decisions
- task counts before and after filtering
- final artifact hashes if convenient

Do not silently download mutable data without recording enough metadata to reproduce the split later.

## Integration Requirements

The output of this stage must be usable by later stages without additional manual editing.

Later stages should be able to load a task and know:

- how to prompt the model
- how to parse the answer
- how to score the answer
- what verifier/checker type to use
- what domain-specific metadata is available

## Tests and Validation Checks

Add tests or validation scripts for:

- schema compliance for every processed task
- split counts by domain/source/split
- uniqueness of task IDs
- no duplicate task IDs across splits
- required fields present
- verifier/checker type assigned for every task
- no empty prompts
- no missing gold targets
- dry-run artifact generation

Also include a human-readable summary printed after data preparation.

## Do Not Do

Do not implement model inference here.

Do not implement CVRO operations here.

Do not train controllers here.

Do not generate counterfactual outcomes here.

Do not manually patch individual examples without recording the patch in metadata.

Do not silently drop examples without writing an exclusion reason.

Do not allow test examples to leak into train or validation.

Do not depend on notebook-only execution.

## Definition of Done

This prompt is complete when the repository can run a data-preparation command that produces:

- `tasks_train.jsonl`
- `tasks_val.jsonl`
- `tasks_test.jsonl`
- `tasks_all.jsonl`
- `task_manifest.json`
- `excluded_examples.jsonl`
- `source_download_manifest.json`

for both:

1. a tiny local dry run, and
2. the full 1,350-example pilot split, assuming datasets are available.

The resulting task artifacts must be schema-valid, domain-balanced, reproducible, and ready for the parser/verifier and state-extraction stages.
