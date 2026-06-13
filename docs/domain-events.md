# Domain Events (`domain/events/`)

**AI Interview Simulator** — Event-sourcing style events that record interview lifecycle transitions. Each event is immutable and timestamped, consumed by `InterviewStateEventsMixin.apply_event()`.

---

## Package Structure

```
domain/events/
├── interview_event.py               # InterviewEvent (base)
├── answer_submitted_event.py        # AnswerSubmittedEvent
├── evaluation_completed_event.py    # EvaluationCompletedEvent
└── execution_completed_event.py     # ExecutionCompletedEvent
```

---

## Base Event

### `InterviewEvent` (`interview_event.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `str` | — | Discriminator string identifying the event kind |
| `timestamp` | `datetime` | `utcnow()` | When the event was created |

All events inherit from this base, ensuring every event has a type tag and a timestamp for audit trail ordering.

---

## Concrete Events

### `AnswerSubmittedEvent` (`answer_submitted_event.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `str` | `"answer_submitted"` | Event discriminator |
| `question_id` | `str` | — | Which question was answered |
| `content` | `str` | — | The answer text |

**Handler** (`InterviewStateEventsMixin`): Appends a new `Answer` to the state's `answers` list.

### `EvaluationCompletedEvent` (`evaluation_completed_event.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `str` | `"evaluation_completed"` | Event discriminator |
| `question_id` | `str` | — | Which question was evaluated |
| `score` | `float` | — | The evaluation score |

### `ExecutionCompletedEvent` (`execution_completed_event.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `str` | `"execution_completed"` | Event discriminator |
| `question_id` | `str` | — | Which question was executed |
| `success` | `bool` | — | Whether execution succeeded |

---

## Usage

```
InterviewEvent                    (base — type + timestamp)
  ├── AnswerSubmittedEvent       ("answer_submitted", question_id, content)
  ├── EvaluationCompletedEvent   ("evaluation_completed", question_id, score)
  └── ExecutionCompletedEvent    ("execution_completed", question_id, success)
```

Events are appended to `InterviewState.events` via `InterviewStateEventsMixin.apply_event()`. This provides a full **audit trail** of all state transitions throughout an interview session, enabling replay, debugging, and temporal queries.
