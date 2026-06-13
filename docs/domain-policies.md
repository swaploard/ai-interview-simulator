# Domain Policies (`domain/policies/`)

**AI Interview Simulator** — Plain injectable business rule classes (not Pydantic). Each policy encapsulates a single decision logic that can be unit-tested and replaced independently.

---

## Package Structure

```
domain/policies/
├── hint_policy.py        # HintPolicy — resolves hint level
├── decision_policy.py    # DecisionPolicy — retry vs next decision
└── navigation_policy.py  # NavigationPolicy — question navigation
```

---

## 1. `HintPolicy` (`hint_policy.py`)

Determines what hint level to offer based on candidate performance.

**Method:** `resolve(*, quality: Quality, attempts: int, has_error: bool) -> HintLevel`

| Quality | Attempts | Result |
|---|---|---|
| `CORRECT` | any | `NONE` (no hint needed) |
| `INCORRECT` | ≤ 1 | `TARGETED` |
| `INCORRECT` | > 1 | `SOLUTION` |
| `PARTIAL` | 1 | `BASIC` |
| `PARTIAL` | 2 | `TARGETED` |
| `PARTIAL` | ≥ 3 | `SOLUTION` |

Escalation pattern: `NONE → BASIC → TARGETED → SOLUTION` based on worsening quality and increasing attempts.

---

## 2. `DecisionPolicy` (`decision_policy.py`)

Determines whether the candidate should retry the current question or move to the next.

**Method:** `decide(*, quality: Quality, attempts: int, max_attempts: int) -> str`

| Quality | Attempts | Result |
|---|---|---|
| `CORRECT` or better | any | `"next"` |
| `PARTIAL` | any | `"next"` |
| `INCORRECT` | < `max_attempts` | `"retry"` |
| `INCORRECT` | ≥ `max_attempts` | `"next"` |

Only `INCORRECT` quality with remaining attempts triggers a retry; anything else progresses the interview.

---

## 3. `NavigationPolicy` (`navigation_policy.py`)

Static helper methods for question navigation within an interview.

| Method | Description |
|---|---|
| `select_next_question_index(questions, current_index) -> int` | Sequential navigation: returns `current_index + 1` (clamped to last index) |
| `find_question_by_difficulty(questions, target_difficulty, fallback_index) -> int` | Finds first question matching target difficulty, or returns `fallback_index` |

---

## Usage

```
Orchestration Flow
  │
  ├── HintPolicy.resolve()
  │     └── Called by AIHintService to decide hint level before LLM prompt
  │
  ├── DecisionPolicy.decide()
  │     └── Called after evaluation/execution to determine next action
  │
  └── NavigationPolicy
        └── Called by InterviewOrchestrator for question progression
```

All three policies are injectable — they can be swapped in tests or replaced with different business logic without changing the services that depend on them.
