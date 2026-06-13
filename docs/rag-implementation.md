# RAG (Retrieval-Augmented Generation) Implementation

**AI Interview Simulator** — Uses RAG to dynamically retrieve relevant interview questions from a curated corpus, augmenting LLM-generated questions with real, pre-written content.

---

## 1. What is RAG?

**Retrieval-Augmented Generation** is a pattern that combines:
- **Retrieval** — fetch relevant documents from a knowledge base using semantic search
- **Generation** — use the retrieved content (or its signals) to produce a better LLM response

Instead of relying solely on the LLM's training data, RAG grounds the output in a specific, queryable corpus. This improves accuracy, reduces hallucination, and enables domain-specific knowledge without fine-tuning.

In this project, RAG retrieves interview questions from a curated question bank stored in ChromaDB. The "generation" side uses retrieved questions directly or enriches them via LLM.

---

## 2. Architecture Overview

```
                    ┌──────────────────────┐
                    │   Question Corpus     │
                    │ (curated questions)   │
                    └──────┬───────────────┘
                           │ indexed via embeddings
                           ▼
                    ┌──────────────────────┐
                    │   ChromaDB Vector     │
                    │   Store (collection:  │
                    │  "interview_questions")│
                    └──────┬───────────────┘
                           │ similarity_search
                           ▼
               ┌───────────────────────────┐
               │   Retrieval Pipeline       │
               │  ┌─────────────────────┐   │
               │  │ 1. Build Query       │   │
               │  │ 2. Staged Fetch      │   │
               │  │ 3. Score + Rerank    │   │
               │  │ 4. Penalty + Boost   │   │
               │  │ 5. Final Selection   │   │
               │  └─────────────────────┘   │
               └──────────┬────────────────┘
                          │ candidate questions
                          ▼
               ┌───────────────────────────┐
               │   Generation / Assembly    │
               │  ┌─────────────────────┐   │
               │  │ Retrieved enough?    │──┤
               │  │  YES → use directly  │  │
               │  │  NO  → LLM generate  │  │
               │  │  Coding → enrich w/  │  │
               │  │           test cases │  │
               │  └─────────────────────┘   │
               └───────────────────────────┘
```

---

## 3. Key Components

### 3.1 Vector Store

**ChromaDB** with collection `"interview_questions"`, persisted at `storage/chroma/interview_corpus/`.

| Detail | Value |
|---|---|
| Vector DB | ChromaDB 0.5.3 |
| Collection | `interview_questions` |
| Persistence | `storage/chroma/interview_corpus/` |
| Embedding model | `text-embedding-3-small` (OpenAI) |
| LangChain integration | `langchain-chroma` — `Chroma` wrapper class |

A legacy vector store also exists at `data/vector_store/` (collection `"question_bank"`), but the production path uses the corpus store above.

### 3.2 Embedding Model

**OpenAI `text-embedding-3-small`** via `langchain-openai`'s `OpenAIEmbeddings`.

Every question is indexed with an `embedding_text` that augments the raw question text:

```
Role: {role}
Area: {area}
Seniority: {seniority}
Domains: {domains}
Topics: {expected_topics}
--- 
{question_text}
```

This augmented text produces richer embeddings that capture the context around each question.

A secondary `SentenceTransformer` (`all-MiniLM-L6-v2`) exists in `services/embedding/` but is not used in the active retrieval pipeline.

### 3.3 Indexing Pipeline

How questions get into the vector store:

```
Curated Questions (JSON / folders)
  → FolderCorpusLoader
  → CorpusIdDeduplicator (removes duplicate IDs)
  → RetrievalCorpusBuilder
      → RetrievalDocumentBuilder
          Creates RetrievalDocument with:
            - text: question display text
            - embedding_text: augmented text (role, area, seniority, domains, topics)
            - embedding: OpenAI embedding of embedding_text
            - metadata: {role, area, seniority, difficulty, domains, tags, quality_score, source}
      → RetrievalEmbeddingRepository (in-memory cache: document_id → embedding)
  → LangChainDocumentAdapter (converts RetrievalDocument → LangChain Document)
  → ChromaCorpusBuilder (deletes old → Chroma.from_documents())
```

### 3.4 Retrieval Query Building

The query sent to ChromaDB is a natural language string, not a metadata filter. Built by `RetrievalQueryBuilder`:

```
Query = {area_hints} + {role_hints} + {level_hints} + {adaptive_topics} + {theme_anchor}
```

Sources of query terms:

| Source | What it provides | Example |
|---|---|---|
| `retrieval_area_hints.py` | Topic words per interview area | database → "sql, indexing, query optimization, joins, normalization, transactions" |
| `retrieval_role_hints.py` | Role-specific keywords | backend → "backend, api, rest, database, microservices, scalability" |
| `retrieval_level_hints.py` | Seniority-appropriate terms | senior → "system design, technical leadership, architecture" |
| Adaptive topics | Weak domains from session memory | weak on "caching" → include "caching" |
| Theme anchor | Strong domain focus | session theme is "distributed systems" → include |

Query templates rotate deterministically based on memory state and session hash (5 variants cycling through hint lists).

### 3.5 Adaptive Retrieval Pipeline

The production retrieval path in `AdaptiveRetrievalService`:

```
retrieve_questions(query, adaptive_context)
  │
  ├── 1. Build 4 Relaxation Stages
  │     Stage 0: role + seniority + area + difficulty_range  (strictest)
  │     Stage 1: seniority + area + difficulty_range
  │     Stage 2: area + difficulty_range
  │     Stage 3: area only                                    (most relaxed)
  │
  ├── 2. Staged Fetch from ChromaDB
  │     For each stage (0 → 3):
  │       FilterBuilder converts RetrievalFilters → Chroma $and/$gte/$lte dict
  │       Chroma.similarity_search_with_score(query, k, filter=where_doc)
  │       Remove already-asked questions (QuestionRepetitionFilter)
  │       If pool >= min_pool_size: STOP
  │       (min_pool = 5 for fresh technical_background, else 1)
  │
  ├── 3. Hybrid Score each candidate
  │     semantic_score = 1.0 - L2_distance
  │     quality_score  = metadata["quality_score"] (default 0.5)
  │     final_score    = semantic × 0.75 + quality × 0.25
  │
  ├── 4. Diversity Reranking (MMR-like)
  │     Greedy selection: pick highest final_score, then
  │       redundancy_penalty = min(max_similarity_to_selected × 0.45, 0.25)
  │       diversity_score = final_score - redundancy_penalty
  │     Uses in-memory embeddings from RetrievalEmbeddingRepository
  │
  ├── 5. Coverage Penalty
  │     adaptive_score -= len(overlap) × 0.15 (penalizes same domains as already selected)
  │
  ├── 6. Weak Domain Boost
  │     adaptive_score += len(overlap) × 0.10 (boosts domains the candidate struggles in)
  │     theme_affinity_boost += 0.08 if domain matches session theme
  │
  └── 7. Performance-Responsive Selection
        Filters session duplicates (topic repeat, semantic dup, cluster overlap, novelty tier)
        Picks best candidate per slot using sort key:
          (target_distance_to_difficulty, spike_flag, variety_penalty..., rank)
```

### 3.6 Metadata Filtering (ChromaDB `$and` / `$gte` / `$lte`)

Each relaxation stage produces a `RetrievalFilters` object, converted to ChromaDB's `where` filter:

```python
{
    "$and": [
        {"role": {"$eq": "backend_engineer"}},
        {"seniority": {"$eq": "senior"}},
        {"area": {"$eq": "tech_background"}},
        {"difficulty": {"$gte": 2}},
        {"difficulty": {"$lte": 4}},
    ]
}
```

These are metadata-level filters (not vector search). ChromaDB returns documents matching both the semantic query AND the metadata filter.

---

## 4. The "Generation" Part

### Written Questions

The `WrittenQuestionPipeline`:
1. Retrieves via `QuestionRetrievalRuntime` → adaptive pipeline
2. Maps `RetrievalCandidate` → `Question` (type=WRITTEN)
3. If fewer than `questions_per_area` retrieved → `QuestionGenerator.generate()` via LLM (gpt-4o-mini)
4. Selects by difficulty distribution: 20% EASY, 60% MEDIUM, 20% HARD

### Coding Questions

The `CodingQuestionPipeline`:
1. Retrieves via dedicated `CodingPipelineRetrievalHelper.retrieve_candidates()`
2. Filters: only prompts matching actionable coding patterns (`solve|implement|write.*function|algorithm|leetcode`)
3. Enriches: `CodingQuestionGenerator.enrich_from_prompt()` uses LLM to generate full coding spec + test cases
4. Fill remaining slots: `CodingQuestionGenerator.generate()` (fresh LLM generation)

### SQL Questions

The `SQLQuestionPipeline`:
1. Same pattern as coding with SQL-specific regex filtering
2. `SQLQuestionGenerator` for enrichment and generation

---

## 5. Scoring & Ranking Summary

| Component | File | Effect |
|---|---|---|
| **Semantic similarity** | `HybridRetrievalScorer` | `score = 1.0 - distance` × weight 0.75 |
| **Quality score** | `HybridRetrievalScorer` | From `metadata.quality_score` (default 0.5) × weight 0.25 |
| **Diversity rerank** | `DiversityReranker` | Penalizes similarity to already-selected candidates (`-min(sim×0.45, 0.25)`) |
| **Coverage penalty** | `CoveragePenaltyEngine` | `-len(domain_overlap) × 0.15` |
| **Weak domain boost** | `WeakDomainBoostEngine` | `+len(domain_overlap) × 0.10`, `+0.08` theme affinity |
| **Difficulty targeting** | `PerformanceResponsiveCandidateSelector` | Sort key prioritizes candidates near target difficulty |
| **Variety scoring** | `SessionVarietyScorer` | Tuple penalizing topic repeats, semantic dupes, cluster overlap |

---

## 6. Retrieval Relaxation Strategy

The system relaxes filters in stages when too few candidates are found:

```
Stage 0: role + seniority + area + difficulty     (strictest)
Stage 1:           seniority + area + difficulty
Stage 2:                       area + difficulty
Stage 3:                       area               (most relaxed)
```

This ensures the system finds enough candidates even when the corpus is sparse for a specific combination.

---

## 7. Memory & Adaptation Loop

The RAG pipeline is adaptive — it learns from the interview as it progresses:

```
InterviewMemoryUpdater
  → tracks: asked_question_ids, covered_domains, weak/strong domains, scores
  → updates: InterviewRetrievalMemory
  
RetrievalQueryBuilder
  → uses memory to build context-aware queries (weak domains become query terms)
  
AdaptiveContextBuilder
  → target_difficulty adjusts based on average_score:
      >= 0.85 → difficulty 5
      >= 0.70 → difficulty 4
      else    → difficulty 3
  
InterviewThemeMemory
  → stores a theme anchor from strong domains
  → propagates into: query terms, generation prompts, affinity boosts
```

---

## 8. Data Flow Diagram

```
                    Interview Setup
                         │
                         ▼
            OrchestrationIntentBuilder
            (builds RetrievalPlanningIntent)
                         │
                         ▼
     ┌──────── QuestionRetrievalRuntime ────────┐
     │                                           │
     ▼                                           ▼
AdaptiveRetrievalService                   HybridRetrievalScorer
  ├─ Relaxation Stages                      ├─ semantic_score (0.75)
  ├─ ChromaDB fetch                         └─ quality_score (0.25)
  └─ QuestionRepetitionFilter                     │
     │                                              ▼
     ▼                                     DiversityReranker
  CoveragePenaltyEngine                    (MMR-like, pairwise sim)
     │                                              │
     ▼                                              ▼
  WeakDomainBoostEngine           PerformanceResponsiveSelector
     │                                              │
     └──────────────┬───────────────────────────────┘
                    ▼
         WrittenQuestionPipeline
         CodingQuestionPipeline
         SQLQuestionPipeline
                    │
                    ▼
         Retrieved + Generated Questions
                    │
                    ▼
              Interview State
```

---

## 9. Key Files Reference

| File | Role |
|---|---|
| `services/question_corpus/question_retrieval_runtime.py` | Public entry point for all question retrieval |
| `services/question_corpus/retrieval/adaptive_retrieval_service.py` | Staged adaptive retrieval logic |
| `services/question_corpus/retrieval/chroma_retrieval_service.py` | ChromaDB wrapper (similarity_search_with_score) |
| `services/question_corpus/retrieval/chroma_filter_builder.py` | Metadata filter construction |
| `services/question_corpus/retrieval/hybrid_retrieval_scorer.py` | Semantic + quality score combination |
| `services/question_corpus/retrieval/diversity_reranker.py` | MMR-like diversity reranking |
| `services/question_corpus/retrieval/coverage_penalty_engine.py` | Domain overlap penalty |
| `services/question_corpus/retrieval/weak_domain_boost_engine.py` | Weak area boosting |
| `services/question_corpus/retrieval/adaptive_retrieval_policy.py` | Relaxation stage definitions |
| `services/question_corpus/retrieval/adaptive_context_builder.py` | Target difficulty from memory |
| `services/question_corpus/retrieval/performance_responsive_candidate_selector.py` | Final candidate selection |
| `services/question_corpus/builders/retrieval_document_builder.py` | Document creation + embedding generation |
| `services/question_corpus/vectorstores/chroma_corpus_builder.py` | ChromaDB index building |
| `services/question_corpus/constants/vector_store_constants.py` | Collection name, paths |
| `services/question_intelligence/retrieval_query_builder.py` | Natural language query construction |
| `services/question_intelligence/retrieval/retrieval_strategy.py` | k, fetch_k, MMR config |
| `infrastructure/embeddings/embedding_factory.py` | OpenAI embedding model factory |
| `infrastructure/embeddings/embedding_similarity_engine.py` | Cosine similarity (pure Python) |
| `infrastructure/config/settings.py` | EMBEDDING_MODEL default |
| `services/question_intelligence/question_generator.py` | LLM question generation (fallback) |
| `services/question_intelligence/coding_question_generator.py` | LLM coding enrichment + generation |
| `services/question_intelligence/sql_question_generator.py` | LLM SQL enrichment + generation |
| `services/question_corpus/retrieval/interview_memory_updater.py` | Retrieval memory tracking |
| `services/question_corpus/retrieval/adaptive_context_builder.py` | Memory → context adaptation |
| `services/question_intelligence/interview_theme_memory.py` | Theme anchor for coherence |
| `services/question_corpus/builders/retrieval_corpus_builder.py` | Corpus assembly pipeline |
