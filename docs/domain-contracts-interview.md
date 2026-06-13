# Interview Contracts (`domain/contracts/interview/`)

**AI Interview Simulator** — Pydantic schemas for interview configuration, state, evaluation, and tracking.

---

## Package Structure

```
domain/contracts/interview/
├── __init__.py                        # empty
├── answer.py                          # Answer
├── hire_decision.py                   # HireDecision enum
├── interview_area.py                  # InterviewArea enum
├── interview_cost_metrics.py          # InterviewCostMetrics, OperationCostMetrics
├── interview_evaluation.py            # InterviewEvaluation
├── interview_level.py                 # InterviewLevel enum
├── interview_memory_context.py        # InterviewMemoryContext
├── interview_metrics.py               # InterviewMetrics, OperationMetrics
├── interview_progress.py              # InterviewProgress enum
├── interview_setup.py                 # InterviewSetup
├── interview_type.py                  # InterviewType enum + get_areas()
└── retrieval_adaptation_signal.py     # RetrievalAdaptationSignal
```

---

## Enums

### `InterviewType` (`interview_type.py`)

| Member | Value |
|---|---|
| `HR` | `"hr"` |
| `TECHNICAL` | `"technical"` |

**Method:** `get_areas() -> list[InterviewArea]` — returns the 5 areas for the type.

### `InterviewArea` (`interview_area.py`)

| HR Areas | Technical Areas |
|---|---|
| `HR_BACKGROUND` | `TECH_BACKGROUND` |
| `HR_TECHNICAL_KNOWLEDGE` | `TECH_TECHNICAL_KNOWLEDGE` |
| `HR_SITUATIONAL` | `TECH_CASE_STUDY` |
| `HR_BRAIN_TEASER` | `TECH_DATABASE` |
| `HR_ANALYTICAL` | `TECH_CODING` |

### `InterviewProgress` (`interview_progress.py`)

| Member | Value |
|---|---|
| `SETUP` | `"setup"` |
| `QUESTIONS_GENERATED` | `"questions_generated"` |
| `IN_PROGRESS` | `"in_progress"` |
| `COMPLETED` | `"completed"` |
| `FAILED` | `"failed"` |

### `InterviewLevel` (`interview_level.py`)

| Member | Value |
|---|---|
| `POOR` | `"poor"` |
| `AVERAGE` | `"average"` |
| `STRONG` | `"strong"` |
| `EXCELLENT` | `"excellent"` |

FAANG-style overall performance tier, determined by `InterviewScoringEngine`.

### `HireDecision` (`hire_decision.py`)

| Member | Value |
|---|---|
| `NO_HIRE` | `"no_hire"` |
| `LEAN_NO_HIRE` | `"lean_no_hire"` |
| `LEAN_HIRE` | `"lean_hire"` |
| `HIRE` | `"hire"` |

Determined by `DecisionEngine` based on dimension scores.

---

## Models

### `Answer` (`answer.py`)

| Field | Type | Description |
|---|---|---|
| `question_id` | `str` | Associated question (min 1 char) |
| `content` | `str` | Answer text (min 1 char) |
| `attempt` | `int` | Attempt number (≥ 1) |

**Constraints:** `frozen=True`, `extra="forbid"`.

### `InterviewSetup` (`interview_setup.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `interview_type` | `InterviewType` | — | `hr` or `technical` |
| `role` | `Role` | — | Target role (from `domain/contracts/user/`) |
| `company` | `CompanyProfile` | — | Target company |
| `language` | `str` | `"en"` | Language code (2–5 chars) |

Pure configuration input — no questions, state, or evaluation. **Constraints:** `frozen=True`.

### `InterviewEvaluation` (`interview_evaluation.py`)

The comprehensive post-interview evaluation result.

| Field | Type | Description |
|---|---|---|
| `overall_score` | `float` | Final score 0–100 |
| `raw_score` | `float \| None` | Uncalibrated score |
| `adjusted_score` | `float \| None` | Bias-corrected score |
| `executive_summary` | `str` | LLM-generated summary |
| `performance_dimensions` | `list[PerformanceDimension]` | Per-dimension breakdown |
| `dimension_scores` | `dict[str, float]` | Deterministic dimension scores |
| `dimension_signals` | `dict[str, float]` | Execution-derived signals |
| `level` | `InterviewLevel` | Overall performance tier |
| `hire_decision` | `HireDecision` | Hire/no-hire decision |
| `decision_explanation` | `dict[str, list[str]]` | Drivers/blockers reasoning |
| `hiring_probability` | `float` | Probability 0–100 |
| `percentile_rank` | `float` | Percentile 0–100 |
| `percentile_explanation` | `str` | Human-readable percentile context |
| `gating_triggered` | `bool` | Whether gating was activated |
| `gating_reason` | `str \| None` | Why gating fired |
| `weighted_breakdown` | `dict[str, float]` | Weighted dimension scores |
| `per_question_assessment` | `list[QuestionEvaluation]` | Per-question results |
| `improvement_suggestions` | `list[str]` | Growth suggestions |
| `confidence` | `Confidence` | Overall confidence |

**Constraints:** `frozen=True`.

### `InterviewMemoryContext` (`interview_memory_context.py`)

Tracks state across questions for adaptive follow-ups.

| Field | Type | Default | Description |
|---|---|---|---|
| `covered_areas` | `list[InterviewArea]` | `[]` | Areas already asked about |
| `covered_concepts` | `list[str]` | `[]` | Concepts already covered |
| `weak_dimensions` | `list[PerformanceDimensionType]` | `[]` | Dimensions where candidate struggled |
| `previous_failures` | `list[str]` | `[]` | Failed question IDs |
| `retrieval_history` | `list[str]` | `[]` | Past retrieval queries |
| `follow_up_history` | `list[str]` | `[]` | Follow-up questions asked |
| `retrieval_adaptation` | `RetrievalAdaptationSignal` | default | Adaptation signal |

**Constraints:** `frozen=True`.

### `RetrievalAdaptationSignal` (`retrieval_adaptation_signal.py`)

Signals to adapt future question retrieval.

| Field | Type | Default | Description |
|---|---|---|---|
| `weak_areas` | `list[InterviewArea]` | `[]` | Areas needing reinforcement |
| `weak_dimensions` | `list[PerformanceDimensionType]` | `[]` | Weak dimension types |
| `low_confidence` | `bool` | `False` | LLM had low confidence |
| `repeated_failures` | `bool` | `False` | Candidate failing repeatedly |

**Constraints:** `frozen=True`.

### `InterviewMetrics` (`interview_metrics.py` — dataclass, slots=True)

| Field | Type | Default | Description |
|---|---|---|---|
| `total_calls` | `int` | — | LLM call count |
| `total_input_tokens` | `int` | — | Input tokens |
| `total_output_tokens` | `int` | — | Output tokens |
| `total_tokens` | `int` | — | Total tokens |
| `total_retries` | `int` | — | Retry count |
| `avg_latency_ms` | `float` | — | Average latency |
| `operations` | `list[OperationMetrics]` | `[]` | Per-operation breakdown |

**`OperationMetrics`:** `operation`, `calls`, `input_tokens`, `output_tokens`, `total_tokens`, `avg_latency_ms`.

### `InterviewCostMetrics` (`interview_cost_metrics.py` — dataclass, slots=True)

| Field | Type | Default | Description |
|---|---|---|---|
| `total_cost_usd` | `float` | — | Total LLM cost |
| `cost_per_question_usd` | `float` | — | Avg cost per question |
| `operations` | `list[OperationCostMetrics]` | `[]` | Per-operation cost breakdown |

**`OperationCostMetrics`:** `operation`, `input_tokens`, `output_tokens`, `cost_usd`.

---

## Dependencies

```
InterviewSetup
  ├── Role              (domain/contracts/user/)
  ├── CompanyProfile    (domain/contracts/user/)
  └── InterviewType     (here)

InterviewEvaluation
  ├── PerformanceDimension (domain/contracts/shared/)
  ├── QuestionEvaluation   (domain/contracts/question/)
  ├── Confidence           (domain/contracts/feedback/)
  ├── InterviewLevel       (here)
  └── HireDecision         (here)

InterviewMemoryContext
  ├── InterviewArea                  (here)
  ├── PerformanceDimensionType       (domain/contracts/shared/)
  └── RetrievalAdaptationSignal      (here)
```
