# Prompt 02: Verifiers, Parsers, and Scoring Interfaces

You are implementing the parser, verifier, and scoring layer for the CVRO pilot.

This stage converts model outputs into structured answers and evaluates whether those answers are correct. It must support math, code, and symbolic reasoning tasks from the processed pilot dataset.

Do not implement model inference, state extraction, CVRO operations, counterfactual collection, controller training, or pilot orchestration in this prompt. Focus on answer extraction, validation, scoring, and failure reporting.

## Context

The CVRO pilot relies on counterfactual action values. Those values are only meaningful if outputs can be scored reliably.

Every model output should be passed through a common scoring interface:

    score_output(output, task) -> ScoreResult

The scoring layer must handle:

- valid correct answers
- valid wrong answers
- unparsable outputs
- malformed code
- execution errors
- timeouts
- verifier/checker failures
- ambiguous or unsupported formats

Silent scoring failures are not acceptable. If an output cannot be scored, the system should return a structured failure result rather than crashing or silently dropping the example.

## Required Scope

Support scoring for three task domains:

| Domain | Required scoring capability |
|---|---|
| `math` | final-answer extraction, numeric normalization, symbolic equivalence where feasible |
| `code` | code extraction, sandboxed execution, test-based correctness |
| `symbolic` | label/relation extraction, deterministic label checking, rule-based or dataset-provided validation where available |

The scoring layer should use the `TaskExample` schema produced by the data pipeline.

## Core Interfaces

Implement a small set of stable interfaces that downstream stages can call.

The exact internal implementation is up to you, but the repository should expose functionality equivalent to the following:

| Function | Purpose |
|---|---|
| `parse_answer(output, task)` | Extract a structured answer/code/label from raw model output. |
| `score_output(output, task)` | Parse and score a raw output against the task. |
| `score_parsed_answer(parsed, task)` | Score an already parsed answer. |
| `normalize_answer(answer, task)` | Normalize answer representation for comparison. |
| `run_checker(parsed, task)` | Run domain-specific deterministic checker if needed. |

The return objects should be serializable and should contain enough metadata for later analysis.

## Suggested Result Schemas

Use or adapt these conceptual schemas.

### `ParsedAnswer`

Should include:

| Field | Description |
|---|---|
| `success` | Whether parsing succeeded. |
| `domain` | `math`, `code`, or `symbolic`. |
| `raw_output` | Original model output or reference to it. |
| `parsed_text` | Extracted answer string/code/label. |
| `normalized` | Normalized representation if available. |
| `answer_type` | Numeric, symbolic, code, label, multiple-choice, relation, etc. |
| `parse_method` | Which parser rule/method succeeded. |
| `parse_error` | Error code/message if parsing failed. |
| `metadata` | Additional parser-specific metadata. |

### `ScoreResult`

Should include:

| Field | Description |
|---|---|
| `correct` | Boolean correctness if scoreable, otherwise false. |
| `scoreable` | Whether the output could be evaluated. |
| `parsed_answer` | Parsed answer object or serialized summary. |
| `normalized_answer` | Normalized answer if available. |
| `gold_normalized` | Normalized gold answer if available. |
| `verifier_type` | Exact, symbolic, numeric, unit test, relation checker, etc. |
| `failure_type` | None, parse error, wrong answer, runtime error, timeout, checker error, unsupported. |
| `checker_metadata` | Domain-specific checker information. |
| `tool_cost` | Runtime/checker cost estimate if applicable. |
| `runtime_seconds` | Execution/checking wall time if applicable. |

Use explicit failure types. Avoid returning ambiguous `None` values where a structured failure code is better.

## Math Parsing and Scoring

Implement robust math answer extraction and scoring.

### Must Support

The parser should handle common answer formats such as:

- `Final answer: ...`
- `Answer: ...`
- `\boxed{...}`
- final line containing a numeric or symbolic answer
- fractions
- decimals
- percentages where unambiguous
- simple algebraic expressions
- tuples/lists/sets when expected by the task
- multiple-choice letters where relevant

### Normalization

Math normalization should support, where feasible:

- stripping whitespace and punctuation
- removing common wrappers such as `\boxed{}`
- normalizing fractions
- numeric comparison with tolerance
- symbolic equivalence using SymPy or an equivalent library
- simple set/tuple normalization
- sign/format normalization where safe

### Scoring

Math scoring should attempt checks in a reasonable order:

1. exact normalized match
2. numeric equivalence if both sides are numeric
3. symbolic equivalence if both sides can be parsed symbolically
4. structured comparison for tuples/sets/lists if applicable
5. fallback wrong/unsupported result

Do not over-normalize in ways that create false positives. If equivalence is uncertain, return a structured unsupported or wrong result rather than guessing.

### Math Failure Types

Use specific failure types such as:

| Failure type | Meaning |
|---|---|
| `parse_error` | No usable answer extracted. |
| `unsupported_format` | Answer format not supported by parser. |
| `symbolic_parse_error` | SymPy or symbolic parser failed. |
| `ambiguous_answer` | Multiple incompatible answers found. |
| `wrong_answer` | Parsed answer is scoreable but incorrect. |

## Code Parsing and Scoring

Implement code extraction and test-based scoring.

### Must Support

The parser should extract Python code from:

- fenced code blocks
- raw Python snippets
- function definitions
- class/function submissions if task requires them
- contest-style full programs where supported by task metadata

Prefer explicit code blocks when present. If multiple code blocks appear, use a deterministic rule and report the choice in metadata.

### Execution Requirements

Code execution must be sandboxed and timeout-limited.

The implementation should support:

- per-test timeout
- total timeout
- captured stdout/stderr
- syntax-error handling
- runtime-error handling
- memory/resource limits where feasible
- allowed/disallowed import policy
- deterministic test execution
- structured failure summaries

Do not let arbitrary generated code access network resources or uncontrolled filesystem state.

### Test Sources

The scoring layer should support tests from:

- dataset-provided unit tests
- public examples converted into tests where appropriate
- generated tests if available from the data pipeline or later modules
- hidden tests if present in local dataset artifacts

This prompt does not need to design the generated-test strategy in full, but the checker should be extensible enough to accept additional tests later.

### Code Failure Types

Use specific failure types such as:

| Failure type | Meaning |
|---|---|
| `parse_error` | No code extracted. |
| `syntax_error` | Code could not be parsed/compiled. |
| `runtime_error` | Code raised an exception. |
| `timeout` | Code exceeded timeout. |
| `wrong_answer` | Code ran but failed tests. |
| `checker_error` | Test harness/checker failed unexpectedly. |
| `unsafe_code` | Code violates sandbox/import policy. |

### Checker Metadata

For code tasks, store metadata such as:

- number of tests run
- number of tests passed
- first failing test summary
- exception type
- timeout information
- whether the failure is suitable for a compact feedback summary
- execution time

This metadata will be used later by the `verify_check` operation.

## Symbolic Parsing and Scoring

Implement scoring for symbolic, logical, and relation-style tasks.

### Must Support

Depending on task metadata, symbolic scoring should handle:

- `True` / `False` / `Unknown`
- yes/no labels
- multiple-choice labels
- relation labels
- entity labels
- simple proof outcome labels
- dataset-specific answer strings

### Normalization

Normalize labels carefully:

- lowercase where safe
- strip punctuation
- map synonyms only when explicitly safe
- map `unknown`, `cannot be determined`, `not enough information`, etc. only if the task label space supports it
- normalize multiple-choice letters separately from answer text

### Deterministic Checking

Where task metadata includes facts/rules or a validator target, implement deterministic checking when feasible. If the task only has a gold label, exact normalized label matching is acceptable.

Do not introduce subjective semantic grading.

### Symbolic Failure Types

Use specific failure types such as:

| Failure type | Meaning |
|---|---|
| `parse_error` | Could not extract label/relation. |
| `invalid_label` | Extracted label not in allowed label set. |
| `wrong_answer` | Valid label but incorrect. |
| `checker_error` | Deterministic checker failed. |
| `unsupported_task` | Task metadata insufficient for scoring. |

## Common Final-Answer Extraction

Many outputs will include reasoning followed by a final answer marker.

Implement final-answer extraction robustly enough to support all domains.

Preferred extraction priority:

1. explicit `Final answer:` marker
2. explicit `Answer:` marker
3. domain-specific final code block or label line
4. last plausible answer-like expression
5. parse failure

If multiple explicit final answers appear, use a deterministic rule, preferably the last explicit final answer, and log that multiple candidates were present.

Do not silently choose among incompatible answers without recording metadata.

## Cost and Runtime Accounting

Scoring should report checker/tool cost.

At minimum:

| Cost field | Meaning |
|---|---|
| `runtime_seconds` | Wall-clock time spent in parser/checker. |
| `checker_calls` | Number of deterministic checks or test batches. |
| `tests_run` | For code tasks. |
| `tool_cost_normalized` | Optional normalized cost used later by action-value computation. |

Do not decide final CVRO cost weights in this prompt. Just expose raw cost information.

## Integration With Later Verify/Check Operation

The later `verify_check` operation will need compact failure summaries.

Implement helpers or metadata to produce concise feedback such as:

- math: mismatch type or symbolic/numeric parse issue
- code: first failing test, exception type, expected vs actual when safe
- symbolic: invalid label, contradiction, or failed rule

Do not expose huge raw logs by default. Store raw logs if useful, but provide compact summaries for prompts.

## Validation and Tests

Add tests or validation scripts covering at least:

### Math

- exact final-answer extraction
- boxed answer extraction
- fraction normalization
- decimal tolerance
- symbolic equivalence
- parse failure on no answer
- ambiguous multiple-answer handling

### Code

- code block extraction
- raw function extraction
- syntax error detection
- runtime error detection
- timeout behavior
- passing unit test
- failing unit test
- unsafe import or sandbox policy if implemented

### Symbolic

- True/False/Unknown parsing
- multiple-choice parsing
- invalid label handling
- relation-label parsing where supported
- exact gold matching

### Cross-Domain

- every processed task has an assigned verifier type
- every processed task can be passed to `score_output`
- scoring returns structured results rather than crashing
- dry-run examples can be scored end-to-end

## Data-Quality Report

Implement a small validation command or script that reads processed tasks and reports:

- number of tasks by domain/source/split
- number with supported verifier types
- parser/checker smoke-test success rate on simple gold outputs
- unsupported tasks
- missing gold/checker fields
- examples likely to fail later scoring

This report should run before model inference.

## Artifact Outputs

This stage may write:

| Artifact | Purpose |
|---|---|
| `parser_validation_report.json` | Machine-readable parser/checker validation results. |
| `parser_validation_report.md` | Human-readable summary. |
| `gold_scoring_smoke_test.jsonl` | Results from scoring gold/reference outputs. |
| `unsupported_tasks.jsonl` | Tasks that cannot be scored with current parser/checker support. |

Use configurable output paths.

## Do Not Do

Do not implement model inference here.

Do not implement CVRO operations here.

Do not train controllers here.

Do not use LLM-as-judge grading for the core pilot.

Do not silently mark unparseable answers as correct.

Do not manually repair generated code.

Do not give CVRO access to verifier information that baselines cannot also use later.

Do not treat code execution as free; report its runtime and checker metadata.

Do not use subjective grading for symbolic tasks.

## Definition of Done

This prompt is complete when the repository can:

1. load processed pilot tasks,
2. parse representative outputs for math, code, and symbolic domains,
3. score gold/reference outputs correctly where available,
4. return structured failure results for malformed outputs,
5. execute code tests safely for supported code tasks,
6. produce parser/checker validation reports,
7. expose stable scoring interfaces for later CVRO stages.

The scoring layer must be reliable enough that counterfactual action values can be computed without manual intervention.
