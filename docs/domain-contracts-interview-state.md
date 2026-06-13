# Interview State (`domain/contracts/interview_state/`)

**AI Interview Simulator** — The central runtime state, composed via mixin-based multiple inheritance for separation of concerns.

---

## Package Structure

```
domain/contracts/interview_state/
├── __init__.py      # composes InterviewState from 7 mixins
├── base.py          # InterviewStateBase — all Pydantic fields
├── results.py       # InterviewStateResultsMixin — register_evaluation/execution
├── progress.py      # InterviewStateProgressMixin — clear_result_for_question
├── events.py        # InterviewStateEventsMixin — apply_event (event sourcing)
├── computed.py      # InterviewStateComputedMixin — current_question, add_answer
├── validation.py    # InterviewStateValidationMixin — model_validator
└── factory.py       # InterviewStateFactoryMixin — create_initial(), create_empty()
```

---

## Composition (`__init__.py`)

`InterviewState` is built by combining 7 mixins into a single class:

```
InterviewState(
    InterviewStateBase,              # all fields
    InterviewStateResultsMixin,      # register_evaluation(), register_execution()
    InterviewStateProgressMixin,     # clear_result_for_question()
    InterviewStateEventsMixin,       # apply_event()
    InterviewStateComputedMixin,     # current_question, add_answer()
    InterviewStateValidationMixin,   # model_validator
    InterviewStateFactoryMixin,      # create_initial(), create_empty()
)
```

Re-exported via `__all__ = ["InterviewState"]`.

---

## 1. `InterviewStateBase` (`base.py`)

The single source of truth for all Pydantic fields.

| Field | Type | Default | Description |
|---|---|---|---|
| `interview_id` | `str` | — | UUID |
| `role` | `Role` | — | Target role |
| `company` | `str` | — | Company name |
| `language` | `str` | `"en"` | Interface language |
| `interview_type` | `InterviewType` | `TECHNICAL` | HR or technical |
| `progress` | `InterviewProgress` | `SETUP` | Lifecycle stage |
| `questions` | `list[Question]` | `[]` | Interview questions |
| `asked_question_ids` | `list[str]` | `[]` | IDs of questions shown |
| `answers` | `list[Answer]` | `[]` | All submitted answers |
| `report_output` | `dict \| None` | `None` | Generated report |
| `interview_evaluation` | `Optional[InterviewEvaluation]` | `None` | Final evaluation |
| `interview_metrics` | `InterviewMetrics \| None` | `None` | LLM metrics |
| `interview_cost_metrics` | `InterviewCostMetrics \| None` | `None` | Cost metrics |
| `chat_history` | `list[str]` | `[]` | Conversation log |
| `results_by_question` | `dict[str, QuestionResult]` | `{}` | Per-question results |
| `dimension_signals` | `dict[PerformanceDimensionType, float]` | `{}` | Signal aggregates |
| `current_question_index` | `int` | `0` | Index into questions |
| `current_question` | `Optional[object]` | `None` | (deprecated, use mixin) |
| `allowed_actions` | `list[ActionType]` | `[]` | Available actions |
| `awaiting_user_input` | `bool` | `False` | Waiting for candidate |
| `memory_context` | `InterviewMemoryContext` | default | Adaptive memory |
| `retrieval_memory` | `InterviewRetrievalMemory` | default | Retrieval history |
| `planned_areas` | `list[str]` | `[]` | Planned coverage areas |
| `adaptive_interview_enabled` | `bool` | `False` | Adaptive mode flag |
| `enable_humanizer` | `bool` | `True` | Humanizer toggle |
| `follow_up_count` | `int` | `0` | Follow-ups used (0–2) |
| `last_humanizer_follow_up` | `bool` | `False` | Humanizer state |
| `events` | `list` | `[]` | Event log |
| `last_feedback_bundle` | `Optional[FeedbackBundle]` | `None` | Last feedback |
| `is_completed` | `bool` | `False` | Completion flag |
| `is_processing` | `bool` | `False` | Processing flag |
| `current_step` | `Optional[LoaderStep]` | `None` | UI step tracking |
| `current_progress` | `int` | `0` | Progress percentage |
| `intent` | `ActionType \| None` | `None` | Current user intent |

**Method:** `with_current_question(question, index)` — returns a new state with `current_question_index` updated and `question.id` appended to `asked_question_ids`.

**Constraints:** `extra="forbid"`.

---

## 2. `InterviewStateResultsMixin` (`results.py`)

Mutators for per-question results (immutable pattern — returns new state copies).

| Method | Description |
|---|---|
| `register_evaluation(evaluation: QuestionEvaluation)` | Upserts `QuestionResult` with an LLM evaluation |
| `register_execution(execution: ExecutionResult)` | Upserts `QuestionResult` with an execution result |
| `get_result_for_question(question_id) -> Optional[QuestionResult]` | Retrieves result by question ID |
| `is_question_processed(question) -> bool` | Checks if a question has been evaluated (written) or executed (coding/db) |

Both `register_*` methods look up existing results or create new ones, then `model_copy(update=...)` to produce a new state.

---

## 3. `InterviewStateProgressMixin` (`progress.py`)

| Method | Description |
|---|---|
| `clear_result_for_question(qid: str)` | Removes a question's result from `results_by_question` (returns new state) |

---

## 4. `InterviewStateEventsMixin` (`events.py`)

Event-sourcing style mutation.

| Method | Description |
|---|---|
| `apply_event(event)` | Deep-copies state, appends event to log, and processes known event types (currently `AnswerSubmittedEvent` → appends an `Answer` to the answers list) |

Extensible for future event types.

---

## 5. `InterviewStateComputedMixin` (`computed.py`)

Derived properties and helpers.

| Member | Description |
|---|---|
| `current_question` (property) | Returns `questions[current_question_index]` or `None` if out of bounds |
| `get_attempt_for_question(question_id) -> int` | Count of answers for a given question |
| `add_answer(answer: Answer)` | Returns new state with answer appended |
| `get_answers_for_question(question_id) -> list[Answer]` | All answers for a question |
| `get_latest_answer_for_question(question_id) -> Optional[Answer]` | Most recent answer |

---

## 6. `InterviewStateValidationMixin` (`validation.py`)

Pydantic `model_validator(mode="after")` ensuring consistency:

| Condition | Rule |
|---|---|
| `progress == IN_PROGRESS` | Must have at least one question |
| `progress == COMPLETED` | Must have at least one result in `results_by_question` |

---

## 7. `InterviewStateFactoryMixin` (`factory.py`)

| Classmethod | Description |
|---|---|
| `create_initial(role_type, interview_type, company, language, questions, interview_id)` | Builds a state with provided config and `SETUP` progress |
| `create_empty()` | Builds a fully default state with a generated `interview_id`, defaults for everything |

---

## Data Flow

```
InterviewStateFactoryMixin.create_initial()
  │
  ▼
  [interview loop]
  │
  ├── InterviewStateComputedMixin.current_question → get next question
  ├── Answer submitted → add_answer()
  ├── Execution/Evaluation → register_execution() / register_evaluation()
  ├── Event logged → apply_event()
  ├── Retry needed → clear_result_for_question()
  └── Validation → InterviewStateValidationMixin runs on every mutation
```
