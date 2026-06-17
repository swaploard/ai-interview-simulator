# Services Architecture Overview

## Top-Level Entry Points

| File | Purpose |
|---|---|
| `llm_interview_service.py` | Thin wrapper around OpenAIClient — loads a prompt template and evaluates a single question-answer pair |
| `question_evaluation_service.py` | Standalone per-question evaluator using gpt-4o-mini with FAANG-level system prompt, retry logic |
| `execution_engine.py` | Dispatches CODING questions to `CodingExecutor` and DATABASE questions to `SQLExecutor` |
| `interview_evaluation_service.py` | Central evaluation orchestrator — runs scoring engine, extracts signals, generates summaries/decisions |
| `narrative_service.py` | LLM-powered generation of executive summaries, decision explanations, and narrative evaluations |
| `report_export_service.py` | Exports final reports as PDF (WeasyPrint) or JSON |
| `report_insight_builder.py` | Builds dimension impact insights and strength/weakness analysis for the report |
| `score_calculator.py` | Simple correctness scorer with performance adjustments |
| `score_calibration_service.py` | Applies LLM bias correction curve to raw scores |
| `feedback_aggregation.py` | Computes overall confidence and severity from feedback blocks |
| `feedback_bundle_factory.py` | Assembles a `FeedbackBundle` domain object |

## Subdirectories by Domain

### Interview Lifecycle Core

| Subdirectory | Description |
|---|---|
| `interview_orchestration/` | Top-level orchestrator — creates intent, retrieves candidates, delegates to planner, validates result |
| `interview_planning/` | Constraint-based question selection with multi-phase planning (required areas → fill → fallback → validation) |
| `planning/` | Supporting algorithms: semantic cluster suppression, difficulty progression, spike suppression, rarity bonuses |
| `planning_validation/` | Validates planning results and produces recovery actions if invalid |
| `replanning/` | Recovery mechanism — expands candidate pool and retries up to 3 times |
| `interview_selection/` | Question selection, sorting, deduplication, difficulty stage assignment (warm-up/core/challenge) |
| `interview_memory/` | Tracks covered areas, retrieval history, and failures across a session |
| `interview_policy/` | Data model for target difficulty, preferred areas, max per area, priority flags |

### Evaluation & Scoring

| Subdirectory | Description |
|---|---|
| `interview_evaluation/` | Subcomponents: `DimensionBuilder`, `ImprovementBuilder`, `NarrativeGenerator`, `ExecutiveSummaryGenerator`, `DecisionExplanationGenerator` |
| `interview_scoring/` | Computes dimension scores, applies gating policies, calculates percentiles and hiring probability |
| `decision_engine/` | Policy-based rules producing a `HireDecision` (STRONG_HIRE / HIRE / NO_HIRE / STRONG_NO_HIRE) |
| `feedback/` | Extracts dimension-relevant signals from execution results, maps and aggregates them |
| `execution_analysis/` | Inspects execution results for runtime errors, logic failures, pass rate |
| `explanation/` | LLM-based explanations for test case failures |

### Execution Engines

| Subdirectory | Description |
|---|---|
| `coding_engine/` | Sandboxed execution of candidate code with test harness, test runner, output parser |
| `sql_engine/` | Executes candidate SQL against a real database, validates results |

### Question Intelligence & Data

| Subdirectory | Description |
|---|---|
| `question_ingestion/` | Ingests questions from GitHub repos, markdown files, CSV datasets |
| `question_intelligence/` | Retrieval, generation, deduplication, clustering, semantic search, difficulty ordering |
| `question_corpus/` | Question data layer — loaders, adapters, repositories, vector stores |
| `embedding/` | Abstraction over embedding models for semantic search/deduplication |
| `candidate_pool/` | Builds and manages the pool of candidate questions for a session |

### Other Support Services

| Subdirectory | Description |
|---|---|
| `ai_hint_engine/` | Generates AI hints during live interviews |
| `hint_rules/` | Domain-specific hint rules (e.g., SQL) |
| `answer_improvement/` | Suggests improvements to candidate answers |
| `humanizer/` | Post-processes LLM responses to sound natural |
| `prompt_builders/` | Constructs prompts for LLM sub-tasks (evaluation, humanizer) |
| `observability/` | Tracks LLM token/cost usage per interview |
| `telemetry/` | Records planning decisions for analysis |

## Data Flow

```
Orchestration → Planning → Selection → [Per-Question Loop] → Evaluation → Scoring → Decision → Report
                     ↑                                                    |
                     └── Replanning (on validation failure)               ↓
                                                                   Narrative / Explanation
```

**Per-Question Loop:**
```
LLM evaluation (llm_interview_service / question_evaluation_service)
  OR
Code/SQL execution (execution_engine → coding_engine / sql_engine)
  → execution_analysis (pass/fail, errors)
  → test case explanation (optional)
  → score_calculator + score_calibration_service
  → signal_extractor (dimension signals)
```

**Final Evaluation (InterviewEvaluationService):**
```
Aggregate all QuestionResults
  → InterviewScoringEngine (dimension scores, gating, percentiles)
  → DecisionEngine (hire decision)
  → NarrativeService (executive summary, decision explanation)
  → DimensionBuilder + ImprovementBuilder
  → ReportInsightBuilder + FeedbackBundleFactory
  → ReportExportService (PDF/JSON)
```
