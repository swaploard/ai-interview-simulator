# Corpus Contracts (`domain/contracts/corpus/`)

**AI Interview Simulator** — Pydantic schemas for the curated question bank.

---

## Package Structure

```
domain/contracts/corpus/
├── __init__.py            # re-exports CuratedQuestion, QuestionCorpus
├── curated_question.py    # CuratedQuestion model
└── question_corpus.py     # QuestionCorpus model
```

---

## 1. `CuratedQuestion` (`curated_question.py`)

A frozen, strict Pydantic model representing a single hand-curated interview question.

| Field | Type | Description |
|---|---|---|
| `id` | `str` | Unique identifier |
| `question` | `str` | The question text |
| `role` | `RoleType` | Target role (e.g. backend, frontend, fullstack) |
| `seniority` | `SeniorityLevel` | Target seniority (junior, mid, senior, lead) |
| `area` | `InterviewArea` | Topic area (e.g. algorithms, system_design, sql) |
| `domains` | `list[str]` | Knowledge domains covered |
| `difficulty` | `int` | Difficulty rating (numeric scale) |
| `source` | `str` | Origin of the question (e.g. "manual_curation", "ai_generated") |
| `quality_score` | `float` | Quality rating (typically 0.0–1.0) |
| `tags` | `list[str]` | Tag metadata for filtering/searching |
| `expected_topics` | `list[str]` | Topics the candidate's answer should cover |
| `follow_up_hints` | `list[str]` | Escalating hints to offer the candidate (optional, defaults to `[]`) |

**Constraints:**
- `frozen=True` — instances are immutable after creation
- `extra="forbid"` — no additional fields allowed beyond those defined

**Dependencies on other contracts:**

```
CuratedQuestion
  ├── RoleType         (domain/contracts/user/role.py)
  ├── SeniorityLevel   (domain/contracts/user/seniority_level.py)
  └── InterviewArea    (domain/contracts/interview/interview_area.py)
```

---

## 2. `QuestionCorpus` (`question_corpus.py`)

A simple frozen collection of curated questions.

| Field | Type | Description |
|---|---|---|
| `questions` | `list[CuratedQuestion]` | The full set of curated questions |

**Constraints:** Same as `CuratedQuestion` — `frozen=True`, `extra="forbid"`.

---

## 3. `__init__.py`

Re-exports both models for convenient imports:

```python
from .question_corpus import QuestionCorpus
from .curated_question import CuratedQuestion
```

Consumers can write `from domain.contracts.corpus import CuratedQuestion` instead of reaching into the submodules.

---

## Data Flow

```
QuestionCorpus
  └── questions: list[CuratedQuestion]
         │
         ▼
CandidatePoolBuilder (services/candidate_pool/)
  ├── filters by role match
  ├── filters by seniority compatibility
  └── produces candidate pool for planner
         │
         ▼
InterviewOrchestrator → ConstraintBasedPlanner → AdaptiveInterviewAssembler
```

1. `QuestionCorpus` holds the complete question bank loaded at startup.
2. `CandidatePoolBuilder` filters `CuratedQuestion` entries by matching the interview's required `RoleType` and `SeniorityLevel`, producing a candidate pool.
3. The pool feeds into `ConstraintBasedPlanner` which selects questions based on area coverage, difficulty progression, and semantic diversity.
4. Finally `AdaptiveInterviewAssembler` arranges selected questions into staged interview sections (WARMUP, CORE, ADVANCED).
