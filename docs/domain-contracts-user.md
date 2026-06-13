# User Contracts (`domain/contracts/user/`)

**AI Interview Simulator** — Schemas for roles, seniority levels, and company profiles that drive interview configuration.

---

## Package Structure

```
domain/contracts/user/
├── __init__.py              # empty
├── role.py                  # RoleType enum, Role model, ROLE_DISTRIBUTION, ROLE_WEIGHTS
├── seniority_level.py       # SeniorityLevel enum
└── company_profile.py       # CompanyType enum, CompanyProfile model
```

---

## Enums

### `RoleType` (`role.py`)

Predefined technical IT roles (only IT roles are supported):

| Member | Value |
|---|---|
| `BACKEND_ENGINEER` | `"backend_engineer"` |
| `FRONTEND_ENGINEER` | `"frontend_engineer"` |
| `FULLSTACK_ENGINEER` | `"fullstack_engineer"` |
| `DEVOPS_ENGINEER` | `"devops_engineer"` |
| `DATA_ENGINEER` | `"data_engineer"` |
| `ML_ENGINEER` | `"ml_engineer"` |
| `QA_ENGINEER` | `"qa_engineer"` |
| `OTHER` | `"other"` |

### `SeniorityLevel` (`seniority_level.py`)

| Member | Value |
|---|---|
| `JUNIOR` | `"junior"` |
| `MID` | `"mid"` |
| `SENIOR` | `"senior"` |

### `CompanyType` (`company_profile.py`)

| Member | Value |
|---|---|
| `GOOGLE` | `"google"` |
| `AMAZON` | `"amazon"` |
| `MICROSOFT` | `"microsoft"` |
| `META` | `"meta"` |
| `APPLE` | `"apple"` |
| `NETFLIX` | `"netflix"` |
| `OTHER` | `"other"` |

---

## Models

### `Role` (`role.py`)

| Field | Type | Description |
|---|---|---|
| `type` | `RoleType` | Predefined role |
| `custom_name` | `str \| None` | Custom name when `type=OTHER` |

**Validation:** `type=OTHER` ⟹ `custom_name` required; otherwise `custom_name` must be `None`.

**Constraints:** `frozen=True`.

### `CompanyProfile` (`company_profile.py`)

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `CompanyType` | — | Predefined company |
| `custom_name` | `str \| None` | `None` | Custom name when `type=OTHER` |

**Validation:** `type=OTHER` ⟹ `custom_name` required (min 1 char); otherwise `custom_name` must be `None`.

**Constraints:** `frozen=True`.

---

## Constants

### `ROLE_DISTRIBUTION` (`role.py`)

Statistical baselines (mean, std) per role for percentile modeling:

| Role | Mean | Std |
|---|---|---|
| BACKEND_ENGINEER | 60.0 | 15.0 |
| FRONTEND_ENGINEER | 62.0 | 14.0 |
| FULLSTACK_ENGINEER | 63.0 | 14.0 |
| DEVOPS_ENGINEER | 65.0 | 13.0 |
| DATA_ENGINEER | 64.0 | 14.0 |
| ML_ENGINEER | 68.0 | 12.0 |
| QA_ENGINEER | 58.0 | 16.0 |
| OTHER | 60.0 | 15.0 |

### `ROLE_WEIGHTS` (`role.py`)

Dimension weightings per role used by `InterviewScoringEngine` to compute weighted scores:

| Role | Technical Depth | System Design | Problem Solving | Communication |
|---|---|---|---|---|
| BACKEND_ENGINEER | 0.35 | 0.30 | 0.20 | 0.15 |
| FRONTEND_ENGINEER | 0.30 | 0.20 | 0.25 | 0.25 |
| FULLSTACK_ENGINEER | 0.30 | 0.25 | 0.25 | 0.20 |
| DEVOPS_ENGINEER | 0.30 | 0.30 | 0.20 | 0.20 |
| DATA_ENGINEER | 0.35 | 0.25 | 0.25 | 0.15 |
| ML_ENGINEER | 0.40 | 0.20 | 0.25 | 0.15 |
| QA_ENGINEER | 0.25 | 0.15 | 0.30 | 0.30 |
| OTHER | 0.25 | 0.25 | 0.25 | 0.25 |

---

## Relationships

```
InterviewSetup
  ├── Role              (here)       — configures the interview
  └── CompanyProfile    (here)       — targets a specific company

QuestionBankItem
  ├── Role / RoleType   (here)       — which role the question targets
  └── SeniorityLevel    (here)       — seniority filter

CuratedQuestion
  ├── RoleType          (here)
  └── SeniorityLevel    (here)

PercentileCalculator   uses ROLE_DISTRIBUTION
DimensionScorer        uses ROLE_WEIGHTS
```
