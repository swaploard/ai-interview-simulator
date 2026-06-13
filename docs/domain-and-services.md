# Domain & Services Layer Reference

**AI Interview Simulator** — Clean Architecture layers: domain entities/policies at the core, business services one level up.

---

## 1. Domain Layer (`domain/`)

Pure business logic with zero external dependencies. All models are Pydantic `BaseModel` subclasses, most frozen for immutability.

### 1.1 Contracts (`domain/contracts/`)

Structured Pydantic schemas grouped by concern.

| Subdirectory | What it holds |
|---|---|
| `interview/` | `Answer`, `InterviewSetup`, `InterviewEvaluation`, `InterviewMetrics`, `InterviewCostMetrics`, `InterviewProgress` (enum), `InterviewType` (enum), `InterviewArea` (enum), `InterviewLevel` (enum), `HireDecision` (enum), `InterviewMemoryContext`, `RetrievalAdaptationSignal` |
| `interview_state/` | `InterviewState` composed from 7 mixins: `Base`, `Results`, `Progress`, `Events`, `Computed`, `Validation`, `Factory` |
| `question/` | `Question`, `QuestionType`/`QuestionDifficulty` (enums), `QuestionEvaluation`, `QuestionResult`, `QuestionProvenance`, `QuestionOriginType` (enum), `QuestionRuntimeLineage`, `QuestionRuntimeTelemetry`, `GeneratedQuestion`, `QuestionBankItem` |
| `execution/` | `CodingSpec`, `CodingTestCase`, `TestCase`, `ExecutionResult`, `TestExecutionResult` |
| `feedback/` | `FeedbackBundle`, `FeedbackBlockResult`, `FeedbackSignal`, `LearningSuggestion`, `Quality` (enum), `Severity` (enum), `Confidence`, `EvaluationDecision`, `EvaluationReport`, `ErrorType` (enum), `DecisionExplanationSchema` |
| `corpus/` | `CuratedQuestion`, `QuestionCorpus` |
| `retrieval/` | `RetrievalPlanningIntent` |
| `ai/` | `AIHint`, `AIHintInput`, `HintLevel` (enum) |
| `shared/` | `ActionType` (enum), `PerformanceDimension`, `PerformanceDimensionType` (enum), `DIMENSION_LABELS` |
| `user/` | `Role`, `RoleType` (enum), `SeniorityLevel` (enum), `CompanyProfile`, `CompanyType` (enum) |

#### Key Model: `InterviewState`

The central runtime state. Composed via multiple inheritance:

```
InterviewStateBase         — all Pydantic fields (questions, answers, progress, results, etc.)
InterviewStateResultsMixin — register_evaluation(), register_execution(), get_result_for_question()
InterviewStateProgressMixin— clear_result_for_question()
InterviewStateEventsMixin  — apply_event() — event sourcing style
InterviewStateComputedMixin— current_question, get_attempt_for_question(), add_answer()
InterviewStateValidationMixin — model_validator ensuring progress consistency
InterviewStateFactoryMixin— create_initial(), create_empty()
```

Mutations return new copies via `model_copy(update=...)` (immutable pattern).

### 1.2 Domain Events (`domain/events/`)

| Event | Fields |
|---|---|
| `InterviewEvent` (base) | `type`, `timestamp` |
| `AnswerSubmittedEvent` | `question_id`, `content` |
| `EvaluationCompletedEvent` | `question_id`, `score` |
| `ExecutionCompletedEvent` | `question_id`, `success` |

Applied to state via `InterviewStateEventsMixin.apply_event()`.

### 1.3 Domain Policies (`domain/policies/`)

Plain classes (not Pydantic) encapsulating injectable business rules:

| Policy | Methods | Logic |
|---|---|---|
| `HintPolicy` | `resolve(quality, attempts, has_error) -> HintLevel` | Quality + attempt-based hint escalation (NONE → BASIC → TARGETED → SOLUTION) |
| `DecisionPolicy` | `decide(quality, attempts, max_attempts) -> str` | Returns `"retry"` or `"next"` based on quality and remaining attempts |
| `NavigationPolicy` | `select_next_question_index(questions, current_index) -> int`, `find_question_by_difficulty(...)` | Static helpers for question navigation |

---

## 2. Services Layer (`services/`)

~40 modules implementing business logic. Each service has a single responsibility.

### 2.1 Orchestration

| Service | File | Responsibility |
|---|---|---|
| `InterviewOrchestrator` | `interview_orchestration/interview_orchestrator.py` | **Main entry point.** Builds retrieval intent → retrieves questions → builds candidate pool → plans → validates → replans if needed → assembles final interview → produces `OrchestrationResult` |
| `OrchestrationIntentBuilder` | `interview_orchestration/orchestration_intent_builder.py` | Builds `RetrievalPlanningIntent` with role/level-specific focus areas |
| `PairwiseSemanticDistanceEngine` | `interview_orchestration/pairwise_semantic_distance_engine.py` | Cosine similarity between question pairs |
| `InterviewSemanticSpreadScorer` | `interview_orchestration/interview_semantic_spread_scorer.py` | Scores semantic diversity of question sets |

### 2.2 Planning & Selection

| Service | File | Responsibility |
|---|---|---|
| `ConstraintBasedPlanner` | `interview_planning/constraint_based_planner.py` | Multi-phase planner: required areas → constraint fill → fallback → validation |
| `PlanningValidator` | `planning_validation/planning_validator.py` | Validates plan against constraints, suggests recovery actions |
| `RecoveryReplanner` | `replanning/recovery_replanner.py` | Retries planning up to 3 times with relaxed constraints |
| `RecoveryCandidateExpander` | `replanning/recovery_candidate_expander.py` | Expands candidate pool when role scope changes |
| `RetrievalRecoveryService` | `replanning/retrieval_recovery_service.py` | Fetches extra questions for expanded roles |
| `AdaptiveInterviewAssembler` | `interview_selection/adaptive_interview_assembler.py` | Assembles final interview with stages (WARMUP, CORE, ADVANCED) |
| `InterviewQuestionSelector` | `interview_selection/interview_question_selector.py` | Two-pass selection: maximize coverage, then semantic balance |
| `CandidatePoolBuilder` | `candidate_pool/candidate_pool_builder.py` | Filters question bank by role/seniority compatibility |
| `PlannerSelectionScoringEngine` | `planning/planner_selection_scoring_engine.py` | Scores candidates: difficulty weight, cluster suppression, novelty/rarity bonuses, spike penalty |
| `SemanticClusterSuppressor` | `planning/semantic_cluster_suppressor.py` | Penalizes similarity > 0.7 with selected questions |
| `SemanticNoveltyBonusEngine` | `planning/semantic_novelty_bonus_engine.py` | Boosts questions with avg similarity < 0.35 |
| `CategoryRarityBonusEngine` | `planning/category_rarity_bonus_engine.py` | Boosts underrepresented categories |
| `DifficultySpikeSuppressor` | `planning/difficulty_spike_suppressor.py` | Penalizes jumps > 2 levels |
| `DifficultyProgressionAnalyzer` | `planning/difficulty_progression_analyzer.py` | Scores monotonic difficulty progression |
| `PolicyFactory` | `interview_policy/policy_factory.py` | Creates role/level-specific `InterviewPolicy` |

### 2.3 Evaluation & Scoring

| Service | File | Responsibility |
|---|---|---|
| `InterviewEvaluationService` | `interview_evaluation_service.py` | **Main evaluation orchestrator.** Coordinates scoring → signals → enrichment → decision → narrative → `InterviewEvaluation` |
| `InterviewScoringEngine` | `interview_scoring/interview_scoring_engine.py` | Computes dimension scores, weighted breakdown, overall score, gating, percentile, confidence, level, hire decision |
| `DimensionScorer` | `interview_scoring/components/dimension_scorer.py` | Maps question areas → performance dimensions → averages scores |
| `PercentileCalculator` | `interview_scoring/components/percentile_calculator.py` | CDF-based percentile using normal distribution |
| `ConfidenceCalculator` | `interview_scoring/components/confidence_calculator.py` | Confidence from score variance |
| `GatingPolicy` | `interview_scoring/components/gating_policy.py` | Flags critical unevaluated dimensions |
| `ScoreCalculator` | `score_calculator.py` | Score from test pass/fail with time adjustment |
| `ScoreCalibrationService` | `score_calibration_service.py` | Bias-correction curve (reduces scores 2-10 pts) |
| `QuestionEvaluationService` | `question_evaluation_service.py` | LLM evaluation of single written answer |
| `LLMInterviewService` | `llm_interview_service.py` | Legacy LLM answer evaluation |

### 2.4 Execution (Coding & SQL)

| Service | File | Responsibility |
|---|---|---|
| `ExecutionEngine` | `execution_engine.py` | Dispatches to `CodingExecutor` or `SQLEvaluator` based on `QuestionType` |
| `CodingExecutor` | `coding_engine/coding_executor.py` | Orchestrates code execution: harness build → sandbox → parse |
| `ExecutionSandbox` | `coding_engine/execution_sandbox.py` | Secure subprocess execution with timeout |
| `TestcaseRunner` | `coding_engine/test_case_runner.py` | Runs test cases through harness |
| `HarnessBuilder` | `coding_engine/harness/harness_builder.py` | Assembles test harness from blocks (imports, user code, entry point, comparator, runner) |
| `ExecutionAnalyzer` | `execution_analysis/execution_analyzer.py` | Classifies errors: SIGNATURE, SYNTAX, RUNTIME, LOGIC |
| `SQLExecutor` | `sql_engine/sql_executor.py` | Executes SQL against in-memory SQLite |
| `SQLEvaluator` | `sql_engine/sql_evaluator.py` | Compares candidate SQL against reference |
| `SQLResultValidator` | `sql_engine/sql_result_validator.py` | Deterministic result validation |
| `SQLDatabase` | `sql_engine/sql_database.py` | In-memory SQLite with employees/departments/projects schema |
| `SchemaSummaryGenerator` | `sql_engine/schema_summary_generator.py` | Textual schema summary |

### 2.5 Feedback & Decisions

| Service | File | Responsibility |
|---|---|---|
| `DecisionEngine` | `decision_engine/decision_engine.py` | Hire/no-hire decision from dimension scores (HIRE ≥ 85, LEAN_HIRE ≥ 70, etc.) |
| `SignalExtractor` | `feedback/signal_extractor.py` | Extracts performance signals from execution results |
| `FeedbackDimensionMapper` | `feedback/dimension_mapper.py` | Maps error types → performance dimensions (LOGIC → Problem Solving, SYNTAX → Technical Depth) |
| `FeedbackDimensionAggregator` | `feedback/dimension_aggregator.py` | Aggregates feedback block dimension metadata |
| `FeedbackBundleFactory` | `feedback_bundle_factory.py` | Creates `FeedbackBundle` with computed severity/confidence/quality |
| `TestCaseExplanationService` | `explanation/test_case_explanation_service.py` | LLM explanations for failed test cases |

### 2.6 Narrative & Humanizer

| Service | File | Responsibility |
|---|---|---|
| `NarrativeService` | `narrative_service.py` | LLM-generated executive summaries, decision explanations, dimension justifications |
| `HumanizerService` | `humanizer/humanizer_service.py` | Makes interview responses sound natural: policy → prompt → LLM → parse |
| `HumanizerPolicyEngine` | `humanizer/humanizer_policy_engine.py` | Decides strategy: FOLLOW_UP, DIRECT_QUESTION, REMARK_PLUS_QUESTION, PLAIN_QUESTION |
| `DimensionBuilder` | `interview_evaluation/builders/dimension_builder.py` | Builds `PerformanceDimension` list from scores + narrative |
| `ImprovementBuilder` | `interview_evaluation/builders/improvement_builder.py` | Builds improvement suggestions |
| `NarrativeControlBuilder` | `interview_evaluation/builders/narrative_control_builder.py` | Classifies balance: BALANCED, SLIGHTLY_UNEVEN, UNBALANCED |
| `DecisionExplanationGenerator` | `interview_evaluation/generators/decision_explanation_generator.py` | Drivers/blockers explanation with dedup |
| `ExecutiveSummaryGenerator` | `interview_evaluation/generators/executive_summary_generator.py` | Wraps `NarrativeService.generate_executive_summary()` |
| `NarrativeGenerator` | `interview_evaluation/generators/narrative_generator.py` | Dimension justifications + improvements via LLM |

### 2.7 Hints & Memory

| Service | File | Responsibility |
|---|---|---|
| `AIHintService` | `ai_hint_engine/ai_hint_service.py` | LLM hint generation (BASIC/TARGETED/SOLUTION levels) |
| `SQLHintRules` | `hint_rules/sql_hint_rules.py` | Rule-based hints for SQL issues (missing LIMIT, ORDER BY, SELECT *) |
| `AnswerImprover` | `answer_improvement/answer_improver.py` | LLM answer improvement given feedback |
| `InterviewMemoryUpdater` | `interview_memory/interview_memory_updater.py` | Updates `InterviewMemoryContext` after each question |

### 2.8 Reporting & Observability

| Service | File | Responsibility |
|---|---|---|
| `ReportExportService` | `report_export_service.py` | PDF (WeasyPrint) and JSON export |
| `ReportInsightBuilder` | `report_insight_builder.py` | Human-readable insights from scores/percentiles |
| `InterviewCostCalculator` | `observability/interview_cost_calculator.py` | Per-operation token cost using OpenAI pricing |
| `PlannerTelemetryBuilder` | `telemetry/planner/planner_telemetry_builder.py` | Planning telemetry (candidate counts, scores, penalties, bonuses) |

### 2.9 Embedding

| Service | File | Responsibility |
|---|---|---|
| `EmbeddingModelProvider` | `embedding/embedding_model_provider.py` | Singleton `SentenceTransformer` loader/cache |

---

## 3. Service Interaction Flow

```
User submits setup
  │
  ▼
InterviewOrchestrator
  ├── OrchestrationIntentBuilder    → RetrievalPlanningIntent
  ├── QuestionRetrievalRuntime      → question candidates
  ├── CandidatePoolBuilder          → eligible questions
  ├── PolicyFactory                 → InterviewPolicy
  ├── InterviewConstraints          → planning rules
  ├── RecoveryReplanner
  │     └── ConstraintBasedPlanner  → selected questions
  │           ├── RequiredAreaSelectionPhase
  │           ├── ConstraintFillPhase
  │           ├── FallbackCompletionPhase
  │           └── PlanningValidationPhase
  ├── PlanningValidator             → recovery actions if needed
  └── AdaptiveInterviewAssembler   → final question set with stages
  │
  ▼
User answers question
  │
  ├── [Written] → QuestionEvaluationService (LLM) → ScoreCalibrationService
  │
  ├── [Coding]  → ExecutionEngine → CodingExecutor
  │                   ├── HarnessBuilder
  │                   ├── ExecutionSandbox (subprocess)
  │                   └── TestcaseRunner
  │
  └── [SQL]     → ExecutionEngine → SQLExecutor → SQLEvaluator
  │
  ▼
InterviewEvaluationService
  ├── InterviewScoringEngine       → dimension scores, percentile, confidence
  ├── SignalExtractor              → performance signals from execution
  ├── ExecutionAnalyzer            → error classification
  ├── DecisionEngine               → hire/no-hire
  ├── ReadableDimensionMapper      → strongest/weakest
  ├── DecisionExplanationGenerator → drivers/blockers
  ├── ExecutiveSummaryGenerator    → via NarrativeService (LLM)
  ├── NarrativeGenerator           → dimension justifications (LLM)
  ├── DimensionBuilder             → PerformanceDimension list
  └── ImprovementBuilder           → suggestions
  │
  ▼
FeedbackBundleFactory              → structured feedback
  │
  ▼
HumanizerService                   → natural language response
  │
  ▼
InterviewMemoryUpdater             → update memory context
  │
  ▼
[Loop or Report]
  └── ReportExportService → PDF/JSON
```

---

## 4. Key Design Patterns

| Pattern | Where | Why |
|---|---|---|
| **Clean Architecture** | `domain/` → `services/` → `app/` → `infrastructure/` | Strict dependency inversion; domain knows nothing outside itself |
| **Immutable State** | `InterviewState` (frozen, `model_copy(update=...)`) | Predictable transitions, easy debugging, thread-safe |
| **Mixin Composition** | `InterviewState` from 7 mixins | Separates concerns without deep inheritance |
| **Event Sourcing** | `InterviewEvent` + `apply_event()` | Audit trail of all state changes |
| **Policy Objects** | `HintPolicy`, `DecisionPolicy`, `NavigationPolicy` | Testable, injectable business rules |
| **Strategy** | Harness strategies (`FunctionStrategy`, `ClassMethodStrategy`) | Pluggable code execution paths |
| **Pipeline** | `ConstraintBasedPlanner` phases | Sequential processing with fallback |
| **Recovery** | `RecoveryReplanner` (3 retries with relaxation) | Graceful degradation when planning fails |
| **Singleton** | `EmbeddingModelProvider` | One model in memory |
