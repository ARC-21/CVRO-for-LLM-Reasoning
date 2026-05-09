# Counterfactual Value of Reasoning Operations

Counterfactual Value of Reasoning Operations (CVRO) is a framework for studying and controlling test-time reasoning in language models. The project treats reasoning as a sequence of choices among distinct operations rather than as a single scalar decision about how many tokens to generate.

At an intermediate reasoning state, a model may need to continue its current trajectory, branch into an alternative trajectory, verify or check a candidate answer, or stop. CVRO measures the value of these operations by applying them counterfactually from the same saved reasoning state and observing their effect on verified correctness and compute cost.

## Core Idea

Most test-time reasoning methods control the amount of computation a model uses. CVRO instead focuses on the type of computation being performed.

The central question is:

> Given the current reasoning state, which operation has the highest expected value: stop, continue, branch, or verify/check?

CVRO estimates this by saving intermediate reasoning states, forcing each operation from the same state under matched compute, and scoring the resulting outputs with task-specific validators.

## Reasoning Operations

CVRO uses four core reasoning operations:

| Operation | Description |
|---|---|
| `stop` | Commit to the current answer and terminate reasoning. |
| `continue` | Extend the current reasoning trajectory. |
| `branch` | Start an independent alternative reasoning trajectory. |
| `verify/check` | Audit the current candidate answer or reasoning path using a verifier, executable test, symbolic checker, or deterministic validator. |

These operations are treated as distinct interventions on the reasoning process. Continuing is not treated as a weaker form of branching, and verification is not treated as simply more reasoning.

## Counterfactual Action Values

For a saved intermediate reasoning state `s` and operation `a`, CVRO estimates an action value:

    Q(s, a) = verified utility after forcing operation a from state s
              - compute cost of operation a

The key requirement is that all operations are evaluated from the same saved state. This allows CVRO to compare what would have happened if the model had stopped, continued, branched, or verified at that exact point.

## Controller

CVRO trains a lightweight controller on counterfactual action-value labels. The controller does not modify the base language model. Instead, it observes features of the current reasoning state and selects the next operation to perform.

Example state features may include:

- the original prompt
- the partial reasoning trace
- the current candidate answer
- token budget already used
- answer history
- disagreement between branches or samples
- confidence or entropy signals when available
- verifier or checker outputs when available
- repetition, contradiction, or self-correction patterns
- task metadata

The controller can be implemented as a small classifier, ranker, regression model, or shallow neural network.

## Supported Task Types

CVRO is intended for tasks where outputs can be checked reliably. Example domains include:

| Domain | Verification Method |
|---|---|
| Math | Exact answer checks, symbolic equivalence, numerical equivalence, or solution validators. |
| Code | Unit tests, execution results, static checks, or input-output validators. |
| Symbolic reasoning | Programmatic validators, proof checkers, satisfiability checks, or deterministic rule systems. |

The framework is designed around verifiable reasoning tasks because counterfactual action values require reliable scoring.

## What CVRO Measures

CVRO is designed to measure:

- whether intermediate reasoning states contain useful information for choosing the next operation
- when continuing the current trajectory helps or hurts
- when branching into an alternative trajectory is more useful than continuing
- when verification/checking is worth its cost
- when stopping prevents overthinking or answer degradation
- whether operation-level control improves accuracy-cost tradeoffs
- whether gains come from state-conditioned control rather than prompt-level routing or scalar budget allocation

## Main Outputs

A CVRO-style project can produce:

- a dataset of saved reasoning states
- counterfactual outcomes for each reasoning operation
- action-value labels for operation selection
- a lightweight operation controller
- evaluation scripts for matched-compute comparisons
- analyses of operation-value regimes and failure modes

## Failure Modes Studied

CVRO can be used to analyze reasoning failures such as:

| Failure Type | Description |
|---|---|
| Premature stop | The model stops before a useful operation would have improved the answer. |
| Futile continuation | The model keeps extending a flawed trajectory. |
| Missed branch | The model fails to explore an alternative path when the current path is unreliable. |
| Unnecessary branch | The model branches when the current trajectory is already sufficient. |
| Missed verification | The model fails to check an answer that could have been corrected. |
| Wasteful verification | The model spends compute checking when verification is unlikely to help. |
| Harmful verification | A verification step causes a correct answer to be changed or rejected. |
| Overthinking | Additional reasoning changes a correct answer into an incorrect one. |

## Repository Scope

This repository is intended to contain code and resources for building, training, and evaluating CVRO-style reasoning controllers.

Typical components include:

- `data/` — saved reasoning states, task metadata, and processed examples
- `src/` — core CVRO implementation
- `controllers/` — lightweight operation-selection models
- `verifiers/` — math, code, and symbolic validation utilities
- `prompts/` — operation prompts for stop, continue, branch, and verify/check
- `experiments/` — experiment entry points and configuration files
- `analysis/` — scripts for regret, action-value, and failure-mode analysis
- `results/` — evaluation outputs and summary tables

## Project Status

This project is under active development. The current focus is defining reliable counterfactual action-value collection, training lightweight operation controllers, and evaluating whether state-conditioned operation choice improves test-time reasoning under matched compute.
