# Feedback Contracts (`domain/contracts/feedback/`)

**AI Interview Simulator** — Schemas for evaluation feedback, quality/severity/confidence, and structured reporting.

---

## Package Structure

```
domain/contracts/feedback/
├── __init__.py                     # empty package marker
├── confidence.py                   # Confidence (base, final)
├── decision_explanation_schema.py  # DecisionExplanationSchema
├── error_type.py                   # ErrorType enum
├── evaluation_decision.py          # EvaluationDecision
├── evaluation_report.py            # EvaluationReport
├── feedback.py                     # FeedbackSignal, LearningSuggestion, FeedbackBlockResult, FeedbackBundle
├── quality.py                      # Quality enum
└── severity.py                     # Severity enum
```

---

## Enums

### `Quality` (`quality.py`)

| Member | Value | Rank |
|---|---|---|
| `INCORRECT` | `"incorrect"` | 0 |
| `PARTIAL` | `"partial"` | 1 |
| `INEFFICIENT` | `"inefficient"` | 2 |
| `CORRECT` | `"correct"` | 3 |
| `OPTIMAL` | `"optimal"` | 4 |

**Methods:**
- `rank() -> int` — numeric ordering (0–4)
- `is_at_least(other) -> bool` — `self.rank() >= other.rank()`
- `is_better_than(other) -> bool` — `self.rank() > other.rank()`

### `Severity` (`severity.py`)

| Member | Value | Rank | Weight |
|---|---|---|---|
| `ERROR` | `"error"` | 0 | 1.0 |
| `WARNING` | `"warning"` | 1 | 0.7 |
| `INFO` | `"info"` | 2 | 0.5 |

**Methods:**
- `rank() -> int` — numeric ordering (0=most severe)
- `weight() -> float` — severity weight multiplier
- `weighted_confidence(confidence: float) -> float` — `weight() * confidence`

### `ErrorType` (`error_type.py`)

| Member | Value |
|---|---|
| `SYNTAX` | `"syntax"` |
| `RUNTIME` | `"runtime"` |
| `LOGIC` | `"logic"` |
| `SIGNATURE` | `"signature"` |
| `TIMEOUT` | `"timeout"` |
| `UNKNOWN` | `"unknown"` |

Used by `ExecutionAnalyzer` to classify execution errors.

---

## Models

### `Confidence` (`confidence.py`)

| Field | Type | Bounds | Description |
|---|---|---|---|
| `base` | `float` | 0.0–1.0 | Per-question LLM certainty |
| `final` | `float` | 0.0–1.0 | Aggregated post-interview confidence |

**Constraints:** `frozen=True`, `extra="forbid"`.

### `DecisionExplanationSchema` (`decision_explanation_schema.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `drivers` | `list[str]` | `[]` | Factors that drove a positive decision |
| `blockers` | `list[str]` | `[]` | Factors that blocked a positive decision |

Used by `DecisionExplanationGenerator` to produce human-readable hire/no-hire reasoning.

### `EvaluationDecision` (`evaluation_decision.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `score` | `float` | — | Score 0.0–100.0 |
| `feedback` | `str` | — | Textual feedback (min 1 char) |
| `strengths` | `list[str]` | `[]` | Identified strengths |
| `weaknesses` | `list[str]` | `[]` | Identified weaknesses |
| `clarification_needed` | `bool` | — | Whether a follow-up is required |
| `follow_up_question` | `Optional[str]` | `None` | The follow-up question |

**Validation:**
- `clarification_needed=True` ⟹ `follow_up_question` is required
- `clarification_needed=False` ⟹ `follow_up_question` must be `None`

**Constraints:** `extra="forbid"` (not frozen — mutable).

### `EvaluationReport` (`evaluation_report.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `interview_id` | `str` | — | Interview identifier |
| `total_score` | `float` | — | Overall score 0.0–100.0 |
| `passed` | `bool` | — | Whether the candidate passed |
| `feedback` | `str` | — | Summary feedback (min 1 char) |
| `evaluations` | `list[QuestionEvaluation]` | `[]` | Per-question evaluations |
| `confidence` | `Confidence` | — | Overall confidence |

**Validation:** Must contain at least one `QuestionEvaluation`.

**Constraints:** `frozen=True`.

### `FeedbackSignal` (`feedback.py` — dataclass)

| Field | Type | Description |
|---|---|---|
| `severity` | `Severity` | Signal severity |
| `message` | `str` | Signal content |

### `LearningSuggestion` (`feedback.py` — dataclass)

| Field | Type | Description |
|---|---|---|
| `topic` | `str` | Topic to study |
| `action` | `str` | Recommended action |

### `FeedbackBlockResult` (`feedback.py` — dataclass)

| Field | Type | Default | Description |
|---|---|---|---|
| `title` | `str` | — | Block title |
| `content` | `str` | — | Block content |
| `severity` | `Severity` | — | Aggregate severity |
| `confidence` | `float` | — | Confidence score |
| `signals` | `list[FeedbackSignal]` | — | Individual signals |
| `learning` | `list[LearningSuggestion]` | — | Learning suggestions |
| `quality` | `Optional[Quality]` | `None` | Quality rating |
| `metadata` | `Optional[dict]` | `None` | Additional metadata |

### `FeedbackBundle` (`feedback.py` — dataclass)

| Field | Type | Description |
|---|---|---|
| `blocks` | `list[FeedbackBlockResult]` | All feedback blocks |
| `overall_severity` | `Severity` | Aggregate severity |
| `overall_confidence` | `float` | Aggregate confidence |
| `overall_quality` | `Quality` | Aggregate quality |
| `markdown` | `str` | Rendered markdown output |

---

## Dependencies

```
EvaluationReport
  ├── QuestionEvaluation  (domain/contracts/question/)
  └── Confidence          (here)

FeedbackBundle
  ├── FeedbackBlockResult  (here)
  │     ├── FeedbackSignal  (here)
  │     ├── LearningSuggestion (here)
  │     ├── Severity        (here)
  │     └── Quality         (here)
  ├── Severity              (here)
  └── Quality               (here)
```
