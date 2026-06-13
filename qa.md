# Q&A: Retrieval and Dense Retrieval in AI Interview Simulator

## How is retrieval managed?

Retrieval is managed through two parallel systems — a **current primary system** (`question_corpus`) and a **legacy/auxiliary system** (`question_intelligence`). Both sit on top of **ChromaDB** (vector database) and use OpenAI's `text-embedding-3-small` for embeddings.

### Current System: `services/question_corpus/`

The public entry point is `QuestionRetrievalRuntime` (`question_retrieval_runtime.py`). It exposes two retrieval paths:

| Method | Purpose |
|--------|---------|
| `retrieve_questions(query, AdaptiveRetrievalContext)` | Full adaptive retrieval pipeline |
| `search(query, k)` / `search_with_filters(query, filters, k)` | Raw Chroma vector search |

**Adaptive Retrieval Pipeline** (`adaptive_retrieval_service.py`):

1. **Filter relaxation** — `AdaptiveRetrievalPolicy` builds progressively relaxed `RetrievalFilters` stages (strict → relax role → relax seniority → bare area), trying each until enough candidates are found.
2. **Vector search** — `ChromaRetrievalService` queries ChromaDB with cosine similarity + metadata filters.
3. **Hybrid scoring** — `HybridRetrievalScorer` blends semantic distance (75%) with document quality score (25%).
4. **Diversity reranking** — `DiversityReranker` applies a greedy MMR-style penalty using embedding cosine similarity.
5. **Coverage & boost** — `CoveragePenaltyEngine` penalizes already-covered domains; `WeakDomainBoostEngine` boosts weak domains.
6. **Candidate selection** — `PerformanceResponsiveCandidateSelector` picks candidates targeting the desired difficulty with variety and spike suppression.

### Auxiliary System: `services/question_intelligence/`

Provides additional retrieval strategies:

- **RetrievalQueryBuilder** — Builds diverse natural language queries from role/level/area hints (rotates through templates).
- **Hybrid retrieval** — `HybridRetrievalEngine` combines BM25 keyword search with semantic search.
- **Fallback retrieval** — `FallbackRetrievalEngine` handles empty results by relaxing filters in stages.
- **Reranking** — `CoverageConstrainedReranker` and `RetrievalReranker` for redundancy-aware reordering.
- **Pipelines** — `CodingPipelineRetrievalHelper` / `SqlPipelineRetrievalHelper` for specialized question retrieval.

### Orchestration Flow

```
Interview State
  → RetrievalPlanningIntent (domain contract)
    → OrchestrationIntentAdapter
      → AdaptiveRetrievalContext
        → QuestionRetrievalRuntime.retrieve_questions()
          → AdaptiveRetrievalService.retrieve()
            → [Filter stages → Chroma search → Hybrid score → Diversity rerank → Coverage/Boost → Select]
              → List[RetrievalCandidate]
                → RetrievalCandidateMapper
                  → QuestionBankItem (domain model)
```

---

## How is Dense Retrieval managed?

Dense retrieval is the core retrieval mechanism — all searches use **embedding-based cosine similarity** against ChromaDB.

### Embedding Model

| Parameter | Value |
|-----------|-------|
| Provider | OpenAI (`text-embedding-3-small`) |
| Dimensions | 1536 |
| Configuration | `infrastructure/config/settings.py` (env var `EMBEDDING_MODEL`) |
| Factory | `infrastructure/embeddings/embedding_factory.py` → `OpenAIEmbeddings()` |

### Embedding Generation & Storage

1. **Corpus building** (`chroma_corpus_builder.py`): Documents are embedded via `OpenAIEmbeddings` and stored in Chroma collection `"interview_questions"` (persisted at `storage/chroma/interview_corpus`).
2. **Document builder** (`retrieval_document_builder.py`): Each `RetrievalDocument` stores both the text and its embedding vector.
3. **Embedding cache** (`retrieval_embedding_repository.py`): In-memory dictionary mapping `document_id → embedding vector` for fast access during reranking/scoring.

### Similarity Computation

The canonical implementation is `infrastructure/embeddings/embedding_similarity_engine.py`:

```python
def similarity(embedding_a, embedding_b) -> float:
    dot_product = sum(a * b for a, b in zip(embedding_a, embedding_b))
    norms = sqrt(sum(x*x for x in a)) * sqrt(sum(x*x for x in b))
    return dot_product / norms  # cosine similarity
```

This is used in:
- `DiversityReranker` — penalty = redundancy_weight × max cosine similarity to already-selected candidates
- `HybridRetrievalScorer` — semantic_score = max(0, 1 - distance)
- `RetrievalReranker` — redundancy penalty based on embedding proximity
- `SemanticDeduplicator` — removes questions above 0.85 cosine similarity threshold

### Known Architecture Issues (from `EMBEDDING_ARCHITECTURE_ANALYSIS.md`)

- **Two embedding models active**: OpenAI `text-embedding-3-small` (1536d) and legacy SentenceTransformer `all-MiniLM-L6-v2` (384d) coexist, causing potential dimension mismatches.
- **Duplicated similarity engines**: 4 separate cosine similarity implementations across the codebase.
- **Redundant embedding generation**: Embeddings are computed twice during corpus building (once in `RetrievalDocumentBuilder`, again in Chroma's `from_documents`).
- **Embedding loss in pipeline**: The `RetrievalDocument → LangChain Document` conversion in `LangChainDocumentAdapter` strips stored embeddings, breaking the `DiversityReranker`'s embedding-based penalty.

---

## Key Files Reference

| Component | Path |
|-----------|------|
| Retrieval entry point | `services/question_corpus/question_retrieval_runtime.py` |
| Adaptive retrieval | `services/question_corpus/retrieval/adaptive_retrieval_service.py` |
| Chroma vector search | `services/question_corpus/retrieval/chroma_retrieval_service.py` |
| Hybrid scorer | `services/question_corpus/retrieval/hybrid_retrieval_scorer.py` |
| Diversity reranker | `services/question_corpus/retrieval/diversity_reranker.py` |
| Embedding factory | `infrastructure/embeddings/embedding_factory.py` |
| Embedding similarity | `infrastructure/embeddings/embedding_similarity_engine.py` |
| Settings | `infrastructure/config/settings.py` |
| Chroma constants | `services/question_corpus/constants/vector_store_constants.py` |
| Query builder | `services/question_intelligence/retrieval_query_builder.py` |
| BM25 hybrid | `services/question_intelligence/hybrid/hybrid_retrieval_engine.py` |
| Fallback retrieval | `services/question_intelligence/fallback/fallback_retrieval_engine.py` |
| Chroma corpus builder | `services/question_corpus/vectorstores/chroma_corpus_builder.py` |
| Legacy embeddings | `services/embedding/embedding_model_provider.py` |
