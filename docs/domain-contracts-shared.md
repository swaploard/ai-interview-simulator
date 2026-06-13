# Shared Contracts (`domain/contracts/shared/`)

**AI Interview Simulator** — Shared enums and models used across multiple other contract packages.

---

## Package Structure

```
domain/contracts/shared/
├── __init__.py                         # empty
├── action_type.py                      # ActionType enum
├── performance_dimension_type.py       # PerformanceDimensionType enum
├── performance_dimension_labels.py     # DIMENSION_LABELS dict
└── performance_dimension.py            # PerformanceDimension model
```

---

## Enums

### `ActionType` (`action_type.py`)

Available user/intent actions in the interview flow.

| Member | Value | Meaning |
|---|---|---|
| `RETRY` | `"retry"` | Retry the current question |
| `NEXT` | `"next"` | Proceed to next question |
| `GENERATE_REPORT` | `"generate_report"` | Generate final report |
| `SUBMIT` | `"submit"` | Submit an answer |
| `NONE` | `"none"` | No action / idle |

Used by `InterviewStateBase.allowed_actions` and `InterviewStateBase.intent` to control and track the current action.

### `PerformanceDimensionType` (`performance_dimension_type.py`)

The four evaluation dimensions scored by `InterviewScoringEngine`.

| Member | Value |
|---|---|
| `TECHNICAL_DEPTH` | `"technical_depth"` |
| `PROBLEM_SOLVING` | `"problem_solving"` |
| `COMMUNICATION` | `"communication"` |
| `SYSTEM_DESIGN` | `"system_design"` |

---

## Constants

### `DIMENSION_LABELS` (`performance_dimension_labels.py`)

Human-readable labels keyed by `PerformanceDimensionType`:

```python
{
    PerformanceDimensionType.TECHNICAL_DEPTH: "Technical Depth",
    PerformanceDimensionType.COMMUNICATION: "Communication",
    PerformanceDimensionType.PROBLEM_SOLVING: "Problem Solving",
    PerformanceDimensionType.SYSTEM_DESIGN: "System Design",
}
```

---

## Models

### `PerformanceDimension` (`performance_dimension.py`)

A scored evaluation dimension with justification.

| Field | Type | Bounds | Description |
|---|---|---|---|
| `name` | `str` | min 1 char | Dimension name (e.g. `"Technical Depth"`) |
| `score` | `float` | 0.0–100.0 | Score for this dimension |
| `justification` | `str` | min 1 char | LLM-generated explanation of the score |

**Constraints:** `frozen=True`.

---

## Usage Across Contracts

```
PerformanceDimension (here)
  └── used in InterviewEvaluation.performance_dimensions

PerformanceDimensionType (here)
  ├── used in InterviewMemoryContext.weak_dimensions
  ├── used in RetrievalAdaptationSignal.weak_dimensions
  ├── used in InterviewStateBase.dimension_signals
  └── used by DimensionScorer to map question areas → dimensions

ActionType (here)
  └── used in InterviewStateBase.allowed_actions, .intent

DIMENSION_LABELS (here)
  └── used by UI and report rendering for human-readable names
```
