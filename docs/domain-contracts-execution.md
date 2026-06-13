# Execution Contracts (`domain/contracts/execution/`)

**AI Interview Simulator** — Pydantic schemas for code execution and test results.

---

## Package Structure

```
domain/contracts/execution/
├── __init__.py                  # empty package marker
├── coding_spec.py              # CodingSpec
├── coding_test_case.py         # CodingTestCase
├── test_case.py                # TestCase (simple input/expected)
├── execution_result.py         # ExecutionResult, ExecutionType, ExecutionStatus
└── test_execution_result.py    # TestExecutionResult, TestStatus, TestType
```

---

## 1. `CodingSpec` (`coding_spec.py`)

Defines the callable signature for a coding question — what function or method the candidate must implement.

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `Literal["function", "class_method"]` | `"function"` | Whether the entry point is a standalone function or a method on a class |
| `entrypoint` | `str` | — | Function name or class name (required, min 1 char) |
| `method_name` | `Optional[str]` | `None` | Method name when `type="class_method"` |
| `parameters` | `list[str]` | `[]` | Parameter names for the callable signature |

**Validation:** If `type="class_method"`, `method_name` is required.

**Constraints:** `frozen=True`, `extra="forbid"`.

Used by `HarnessBuilder` to dynamically construct the test harness that wraps candidate code.

---

## 2. `CodingTestCase` (`coding_test_case.py`)

A single test case for a coding question with positional/keyword arguments and expected output.

| Field | Type | Default | Description |
|---|---|---|---|
| `args` | `list[Any]` | `[]` | Positional arguments passed to the callable |
| `kwargs` | `dict[str, Any]` | `{}` | Keyword arguments passed to the callable |
| `expected` | `Any` | — | The expected return value |

**Constraints:** `frozen=True`.

Used by `TestcaseRunner` — the harness calls the candidate's function with `args`/`kwargs` and compares the result to `expected`.

---

## 3. `TestCase` (`test_case.py`)

A simpler, string-based test case model (used for SQL or simpler evaluation).

| Field | Type | Default | Description |
|---|---|---|---|
| `input` | `str` | — | Input for the function |
| `expected_output` | `str` | — | Expected output |

**Note:** Has `__test__: ClassVar[bool] = False` to prevent pytest from collecting it as a test class.

**Constraints:** `frozen=True`, `extra="forbid"`.

---

## 4. `TestExecutionResult` (`test_execution_result.py`)

The detailed result of running a single test case.

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | `int` | — | Test case identifier |
| `type` | `TestType` | — | `visible` (shown to candidate) or `hidden` |
| `status` | `TestStatus` | — | `passed`, `failed`, or `error` |
| `expected` | `Optional[Any]` | `None` | Expected value |
| `actual` | `Optional[Any]` | `None` | Actual value produced |
| `error` | `Optional[str]` | `None` | Error message if execution failed |
| `args` | `list[Any]` | `[]` | Positional args used |
| `kwargs` | `dict[str, Any]` | `{}` | Keyword args used |

**Enums:**
- `TestStatus`: `PASSED`, `FAILED`, `ERROR`
- `TestType`: `VISIBLE`, `HIDDEN`

**Constraints:** `frozen=True`.

---

## 5. `ExecutionResult` (`execution_result.py`)

Top-level result of executing a candidate's code.

| Field | Type | Default | Description |
|---|---|---|---|
| `question_id` | `str` | — | Associated question ID |
| `execution_type` | `ExecutionType` | — | `coding` or `database` |
| `status` | `ExecutionStatus` | — | High-level status (see below) |
| `success` | `bool` | — | Overall pass/fail |
| `output` | `str` | `""` | Raw stdout output |
| `error` | `Optional[str]` | `None` | Error message (required when `success=False`) |
| `passed_tests` | `int` | `0` | Number of passed tests |
| `total_tests` | `int` | `0` | Total tests run |
| `execution_time_ms` | `int` | `0` | Wall-clock execution time |
| `test_results` | `list[TestExecutionResult]` | `[]` | Per-test detailed results |

**Enums:**
- `ExecutionType`: `CODING`, `DATABASE`
- `ExecutionStatus`:
  | Value | Meaning |
  |---|---|
  | `SUCCESS` | All tests passed |
  | `FAILED_TESTS` | Some tests failed |
  | `SYNTAX_ERROR` | Compile/parse error |
  | `RUNTIME_ERROR` | Exception at runtime |
  | `TIMEOUT` | Execution exceeded time limit |
  | `INTERNAL_ERROR` | Infrastructure error |

**Validations:**
- `success=True` ⟹ `status=SUCCESS` and `error=None`
- `success=False` ⟹ `status≠SUCCESS` and `error` is set
- `passed_tests ≤ total_tests`

**Constraints:** `frozen=True`, `extra="forbid"`.

---

## Dependencies

```
ExecutionResult
  └── TestExecutionResult   (domain/contracts/execution/test_execution_result.py)
      ├── TestStatus (enum)
      └── TestType   (enum)
```
