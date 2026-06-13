# AI Contracts (`domain/contracts/ai/`)

**AI Interview Simulator** — Pydantic schemas for the AI hint generation feature.

---

## Package Structure

```
domain/contracts/ai/
├── __init__.py       # package marker (empty)
├── hint_level.py     # HintLevel enum
└── ai_hint.py        # AIHintInput, AIHint models
```

---

## 1. `HintLevel` (`hint_level.py`)

Enum defining four progressive hint tiers:

| Member | Value | Meaning |
|---|---|---|
| `NONE` | `"none"` | No hint provided |
| `BASIC` | `"basic"` | Gentle nudge |
| `TARGETED` | `"targeted"` | Specific, targeted guidance |
| `SOLUTION` | `"solution"` | Full solution disclosure |

Inherits `str` so each member is directly serializable as its value string (e.g., `HintLevel.BASIC == "basic"`).

Used by `HintPolicy` in `domain/policies/` to escalate hint levels based on candidate quality, attempt count, and error state, and by `AIHintService` in `services/` to control LLM prompt strictness.

---

## 2. `AIHintInput` (`ai_hint.py`)

Input payload for requesting an AI-generated hint:

| Field | Type | Default | Description |
|---|---|---|---|
| `error` | `Optional[str]` | `None` | Error message the candidate encountered |
| `user_code` | `str` | — | The candidate's current code |
| `failed_tests` | `str` | — | Failing test output |
| `question` | `str` | — | The interview question prompt |
| `hint_level` | `HintLevel` | `HintLevel.BASIC` | Desired hint strength |

---

## 3. `AIHint` (`ai_hint.py`)

Output model returned by the AI hint engine:

| Field | Type | Description |
|---|---|---|
| `explanation` | `str` | Explanation of the issue or concept |
| `suggestion` | `str` | Suggested fix or next step |

**Frozen** (`model_config = {"frozen": True}`) — once created, the instance is immutable.

---

## Data Flow

```
AIHintInput                        AIHintService (LLM)                AIHint
  ├─ question                         │                               ├─ explanation
  ├─ user_code                        │                               └─ suggestion
  ├─ failed_tests                     │
  ├─ error (optional)                 │
  └─ hint_level (controls LLM)       │
```

1. `AIHintInput` is built by the caller with the candidate's context and desired hint level.
2. `AIHintService` reads the input and sends a tailored prompt to an LLM.
3. The LLM response is parsed and returned as a frozen `AIHint`.

---

## Relation to Hint Policy

The `HintLevel` is determined by `HintPolicy` (`domain/policies/hint_policy.py`) based on:

- **Quality score** of the candidate's last answer
- **Attempt count** for the current question
- **Presence of errors**

| Condition | Resulting HintLevel |
|---|---|
| Quality > 0.7 | `NONE` (candidate doesn't need help) |
| Attempts = 0, no error | `BASIC` |
| Attempts >= 1 or error present | `TARGETED` |
| Attempts >= max attempts | `SOLUTION` |

---

## Dependencies

- `ai_hint.py` imports `HintLevel` from `hint_level.py`
- `__init__.py` has no code
- `HintLevel` is re-exported/consumed by `domain/policies/hint_policy.py` and `services/ai_hint_engine/ai_hint_service.py`
