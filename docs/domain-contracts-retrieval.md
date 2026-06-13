# Retrieval Contracts (`domain/contracts/retrieval/`)

**AI Interview Simulator** — Contract for the question retrieval planning intent.

---

## Package Structure

```
domain/contracts/retrieval/
├── __init__.py                       # empty
└── retrieval_planning_intent.py      # RetrievalPlanningIntent
```

---

## `RetrievalPlanningIntent` (`retrieval_planning_intent.py`)

Input contract that drives the question retrieval phase of interview orchestration.

| Field | Type | Default | Description |
|---|---|---|---|
| `focus_areas` | `list[str]` | — | Topic/concept areas to prioritize |
| `required_tags` | `list[str]` | — | Tags that matching questions must have |
| `target_level` | `str` | — | Target seniority/level string |
| `query_text` | `str` | — | Natural language query for retrieval |
| `max_candidates` | `int` | `15` | Maximum number of candidate questions to return |

**Constraints:** `frozen=True`, `extra="forbid"`.

---

## Data Flow

```
OrchestrationIntentBuilder
  │
  └── builds RetrievalPlanningIntent
        │
        ▼
QuestionRetrievalRuntime
  │
  ├── uses focus_areas, required_tags, query_text to search question bank
  ├── respects max_candidates limit
  └── returns candidate questions
        │
        ▼
CandidatePoolBuilder → filters by role/seniority
        │
        ▼
ConstraintBasedPlanner → selects final question set
```

`RetrievalPlanningIntent` is the bridge between `OrchestrationIntentBuilder` (which determines what the interview needs based on role/level) and `QuestionRetrievalRuntime` (which searches the question corpus). `focus_areas` and `required_tags` are derived from the role's required skills and the interview type, while `query_text` is used for semantic/embedding-based search.
