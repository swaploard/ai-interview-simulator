# AI Interview Simulator — Project Overview

An **AI-driven interview simulation platform** with semantic retrieval, structured evaluation, and executable skill validation (coding + SQL). Built with **Clean Architecture**, orchestrated via **LangGraph**, and served through a **Gradio UI**.

---

## Quick Facts

| Attribute | Value |
|-----------|-------|
| Python | ≥ 3.11 |
| UI Framework | Gradio 4.44.1 |
| LLM | OpenAI GPT-4o-mini |
| Embeddings | OpenAI text-embedding-3-small |
| Vector Store | ChromaDB |
| State Machine | LangGraph StateGraph |
| Persistence | SQLite |
| PDF Export | WeasyPrint |

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│                    UI (Gradio)                      │
│  app/ui/ — views, components, state machine, events │
├─────────────────────────────────────────────────────┤
│               LangGraph Workflow                     │
│  app/graph/ — StateGraph driving interview lifecycle │
├─────────────────────────────────────────────────────┤
│                Service Layer                         │
│  services/ — ~40 modules (scoring, eval, planning,  │
│              hints, retrieval, execution, feedback)  │
├─────────────────────────────────────────────────────┤
│             Infrastructure Layer                     │
│  infrastructure/ — LLM adapters, vector stores,      │
│                    embeddings, SQLite, settings      │
├─────────────────────────────────────────────────────┤
│                Domain Layer                          │
│  domain/ — entities, policies, events, contracts     │
└─────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
ai-interview-simulator/
│
├── app/                           # Application layer
│   ├── main.py                    # Entry point
│   ├── application/               # Use cases (evaluate_answer)
│   ├── ports/                     # Protocol interfaces (LLMPort)
│   ├── contracts/                 # Output contract, feedback bundle
│   ├── core/                      # Logger, build info, execution score policy
│   ├── execution/                 # PythonSandbox (secure code execution)
│   ├── graph/                     # LangGraph orchestration
│   │   ├── interview_graph.py     # StateGraph definition + routing
│   │   └── nodes/                 # 14 node implementations
│   ├── prompts/                   # Jinja2 prompt templates
│   ├── runtime/                   # Graph factory, I/O normalization
│   ├── settings/                  # App-level constants
│   └── ui/                        # Gradio frontend
│       ├── app.py                 # build_app() — gr.Blocks construction
│       ├── ui_state.py            # UIState enum
│       ├── layout/                # Layout builders
│       ├── views/                 # Setup, Written, Coding, DB, Report views
│       ├── components/            # Chat, Loader components
│       ├── presenters/            # Result/feedback presenters
│       ├── state_machine/         # UIState resolution
│       ├── bindings/              # Event-to-handler bindings
│       ├── builders/              # UI response builders
│       ├── adapters/              # Output adapters + DTOs
│       ├── handlers/              # Start, Submit, Navigation, Export handlers
│       └── constants/             # Loader steps
│
├── domain/                        # Core business logic
│   ├── contracts/                 # Pydantic models
│   │   ├── interview_state/       # InterviewState (mixins: results, progress, events, computed, validation)
│   │   ├── question/              # Question, QuestionType, QuestionDifficulty
│   │   ├── retrieval/             # RetrievalPlanningIntent
│   │   └── ...                    # Answer, Feedback, Evaluation, Execution, etc.
│   ├── events/                    # AnswerSubmitted, EvaluationCompleted, etc.
│   └── policies/                  # DecisionPolicy, HintPolicy, NavigationPolicy
│
├── infrastructure/                # External concerns
│   ├── config/settings.py         # Pydantic settings (API key, model name)
│   ├── embeddings/                # Embedding factory + similarity engine
│   ├── llm/                       # LLM adapter, factory, observability, metrics, pricing
│   ├── persistence/sqlite/        # SQLite connection, models, repository
│   └── vector_store/              # Chroma question store (legacy)
│
├── services/                      # Business logic (~39 modules)
│   ├── interview_orchestration/   # End-to-end orchestrator
│   ├── interview_planning/        # Constraint-based planning
│   ├── interview_evaluation/      # Scoring, narrative, decision engine
│   ├── interview_scoring/         # Dimension weights, percentiles
│   ├── question_corpus/           # PRIMARY retrieval system
│   │   ├── question_retrieval_runtime.py  # Public entry point
│   │   ├── retrieval/             # Adaptive, Chroma, hybrid scoring, diversity rerank
│   │   ├── builders/              # RetrievalDocument + corpus builders
│   │   ├── vectorstores/          # Chroma corpus builder
│   │   ├── contracts/             # RetrievalCandidate, Filters, Documents
│   │   ├── adapters/              # LangChain adapters
│   │   ├── repositories/          # Embedding cache
│   │   └── constants/             # Vector store config
│   ├── question_intelligence/     # Question gen, retrieval strategies, clustering
│   ├── coding_engine/             # Code execution, sandbox, test runners
│   ├── sql_engine/                # SQL execution, evaluation, schema summarization
│   ├── feedback/                  # Signal extraction, dimension aggregation
│   ├── ai_hint_engine/            # LLM hint generation
│   ├── humanizer/                 # Natural language humanization
│   ├── decision_engine/           # Hire/no-hire decisions
│   ├── narrative_service.py       # LLM-generated summaries
│   ├── score_calibration_service.py  # LLM bias correction
│   ├── report_export_service.py   # PDF/JSON export
│   └── ...                        # Replanning, candidate pool, telemetry, etc.
│
├── interface/cli/                 # CLI adapter
├── data/                          # Sample questions, vector store persistence
├── datasets/                      # Curated datasets + registries
├── storage/                       # Chroma corpus persistence
├── scripts/                       # Test/utility harnesses by domain
├── tests/                         # Unit, integration, graph, orchestration tests
├── docs/                          # Architecture docs, ADRs, roadmap
├── gradio_app.py                  # HuggingFace Spaces entry point
├── pyproject.toml                 # Project metadata
├── requirements.txt               # Dependencies
└── .env.example                   # OPENAI_API_KEY, MODEL_NAME
```

---

## LangGraph Workflow

The interview lifecycle is a **StateGraph[InterviewState]** with 12 nodes:

```
entry ──► router ──► execution ──► evaluation
  │          │                        │
  │          └──► written             │
  │                                   │
  └──► navigation                     │
       │                              ▼
       │                            hint
       │                              │
       │                           feedback
       │                              │
       │                          decision
       │                         /        \
       │                   [await user]  navigation
       │                       │            │
       │                       ▼            │
       │                       END       completion
       │                                    │
       │                             evaluation_aggregate
       │                                    │
       └──────────────────────────────► report ──► END
```

**Key routing:**
- `written` questions → LLM evaluation directly
- `coding`/`database` questions → sandbox execution → auto-evaluation
- After decision: if awaiting user input → END (pause); else → loop to navigation
- When all questions answered → aggregate scores → generate report → END

---

## Key Design Patterns

1. **Clean Architecture** — Strict layering: domain → application → infrastructure
2. **Immutable State** — `InterviewState` transitions via `model_copy(update=...)`
3. **Protocol Ports** — `LLMPort` is a `Protocol` for dependency inversion
4. **Mixin Composition** — `InterviewState` composed from 6+ mixins
5. **Policy Objects** — Pure, testable domain policies (decision, hint, navigation)
6. **StateGraph** — LangGraph drives the entire interview lifecycle
7. **Observability** — LLM metrics collection (latency, tokens, cost)
8. **Adaptive Retrieval** — Multi-stage filter relaxation + hybrid scoring + diversity reranking

---

## Configuration

| Setting | Default | Source |
|---------|---------|--------|
| `OPENAI_API_KEY` | — | `.env` |
| `MODEL_NAME` | `gpt-4o-mini` | `.env` / `settings.py` |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | `settings.py` |
| `QUESTIONS_PER_AREA` | `1` | `app/settings/constants.py` |
| `DEDUPLICATION_THRESHOLD` | `0.85` | `app/settings/constants.py` |

---

## Data Flow

```
User (Gradio)
  → UI Event Handler mutates InterviewState
    → LangGraph.invoke(state)
      → execution/written → evaluation → hint → feedback → decision
      → [loop or completion → aggregate → report]
    → Updated InterviewState returned
  → UIStateMachine resolves UIState
  → UI Router updates visible sections
  → Presenters build component updates
  → Rendered to user
```
