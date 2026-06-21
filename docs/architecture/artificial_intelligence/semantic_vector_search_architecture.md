# Semantic / Vector Search Architecture

## Overview

**Semantic / Vector Search Architecture** defines the structural patterns for building search systems that understand meaning rather than matching keywords. By encoding text, images, and other data into dense vector representations (embeddings), vector search finds results based on semantic similarity — enabling queries like "affordable waterfront dining" to match "budget-friendly restaurants with ocean views" even when they share no common words.

Vector search has become the foundational retrieval layer for RAG systems, recommendation engines, duplicate detection, anomaly detection, and multi-modal search. The architecture spans embedding model selection, vector database design, index construction, hybrid retrieval strategies, and production scaling.

Key principles:

- **Semantic Understanding** — Move beyond keyword matching to meaning-based retrieval using dense embeddings
- **Approximate Is Acceptable** — Approximate nearest neighbor (ANN) search trades marginal recall for dramatic speed improvements
- **Hybrid Search** — Combine vector similarity with keyword search (BM25) for robust retrieval that handles both semantic and exact-match queries
- **Embedding Quality Drives Results** — The choice and quality of the embedding model has more impact than the choice of vector database
- **Index Design Matters** — The right index type, distance metric, and parameters determine the latency/recall/memory trade-off

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Semantic / Vector Search Architecture                       │
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Indexing Pipeline                              │      │
│   │                                                                  │      │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │      │
│   │  │  Source   │─▶│  Chunk    │─▶│  Embed    │─▶│  Index &      │   │      │
│   │  │  Ingest   │  │  & Parse  │  │  (Model)  │  │  Store        │   │      │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │      │
│   └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│   Query                                                                     │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────┐                                                          │
│   │  Query        │    Rewrite, expand, decompose                           │
│   │  Processing   │                                                         │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│          ├──────────────────────────────┐                                   │
│          ▼                              ▼                                    │
│   ┌──────────────┐            ┌──────────────┐                             │
│   │  Vector       │            │  Keyword      │                             │
│   │  Search       │            │  Search       │                             │
│   │  (ANN)        │            │  (BM25)       │                             │
│   └──────┬───────┘            └──────┬───────┘                             │
│          │                           │                                      │
│          └─────────┬─────────────────┘                                      │
│                    ▼                                                        │
│          ┌──────────────────┐                                               │
│          │  Fusion &         │   Reciprocal Rank Fusion (RRF)               │
│          │  Re-Ranking       │   or Cross-Encoder Re-ranking                │
│          └────────┬─────────┘                                               │
│                   │                                                         │
│                   ▼                                                         │
│            Ranked Results                                                   │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Embedding   │  │  Index        │  │  Filtering &      │  │
│                   │ Model Mgmt  │  │  Lifecycle    │  │  Multi-Tenancy    │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Embedding Models

The embedding model is the most critical design choice. It determines what "similarity" means to your system:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Embedding Model Selection                           │
├──────────────────┬──────────────────────────────────────────────────┤
│  Category        │  Examples and Characteristics                    │
├──────────────────┼──────────────────────────────────────────────────┤
│  General Purpose │  OpenAI text-embedding-3, Cohere embed-v3,      │
│                  │  BGE, E5. Good baseline for most domains.       │
├──────────────────┼──────────────────────────────────────────────────┤
│  Domain-Specific │  Fine-tuned on legal, medical, financial text.  │
│                  │  Higher relevance for specialized vocabularies. │
├──────────────────┼──────────────────────────────────────────────────┤
│  Multi-Lingual   │  Cohere multilingual, mE5, LaBSE.              │
│                  │  Cross-language semantic matching.              │
├──────────────────┼──────────────────────────────────────────────────┤
│  Multi-Modal     │  CLIP, SigLIP, ImageBind.                       │
│                  │  Text and image in same embedding space.        │
├──────────────────┼──────────────────────────────────────────────────┤
│  Code            │  CodeBERT, StarCoder embeddings.                │
│                  │  Semantic code search and similarity.           │
└──────────────────┴──────────────────────────────────────────────────┘
```

```
// Embedding Service — Abstracts model details from consumers
CLASS EmbeddingService
    PROPERTIES
        model       : EmbeddingModel
        dimensions  : Int
        maxTokens   : Int
        batchSize   : Int

    FUNCTION embed(text : String) -> Vector
        truncated = truncateToMaxTokens(text, maxTokens)
        RETURN model.encode(truncated)

    FUNCTION embedBatch(texts : List<String>) -> List<Vector>
        batches = partition(texts, batchSize)
        results = EMPTY LIST
        FOR EACH batch IN batches
            truncated = [truncateToMaxTokens(t, maxTokens) FOR t IN batch]
            results.ADD_ALL(model.encodeBatch(truncated))
        END FOR
        RETURN results

    // Dimensionality reduction for storage efficiency
    FUNCTION embedWithMRL(text : String,
                          targetDim : Int) -> Vector
        fullEmbedding = embed(text)
        // Matryoshka Representation Learning — truncate dimensions
        RETURN fullEmbedding[:targetDim]
END CLASS
```

### Vector Index Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ANN Index Comparison                               │
├──────────────────┬──────────┬──────────┬───────────┬───────────────┤
│  Index Type      │  Speed   │  Recall  │  Memory   │  Build Time   │
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  Flat (Exact)    │  Slow    │  100%    │  Low      │  None         │
│                  │          │          │           │  (brute force)│
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  HNSW            │  Fast    │  95-99%  │  High     │  Slow         │
│  (graph-based)   │          │          │ (in-memory│               │
│                  │          │          │  graph)   │               │
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  IVF             │  Fast    │  90-98%  │  Medium   │  Medium       │
│  (cluster-based) │          │          │           │ (K-means)     │
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  PQ              │  Very    │  85-95%  │  Very Low │  Medium       │
│  (compression)   │  Fast    │          │ (compress │               │
│                  │          │          │  vectors) │               │
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  IVF-PQ          │  Very    │  88-96%  │  Low      │  Slow         │
│  (combined)      │  Fast    │          │           │               │
├──────────────────┼──────────┼──────────┼───────────┼───────────────┤
│  DiskANN         │  Fast    │  95-99%  │  Low      │  Slow         │
│  (disk-based)    │          │          │ (on disk) │               │
└──────────────────┴──────────┴──────────┴───────────┴───────────────┘
```

```
// Vector Index Configuration
CLASS VectorIndexConfig
    PROPERTIES
        indexType    : IndexType         // HNSW, IVF, PQ, IVF_PQ, FLAT
        metric       : DistanceMetric    // COSINE, DOT_PRODUCT, L2
        dimensions   : Int

        // HNSW parameters
        hnswM        : Int = 16          // Max edges per node (higher = better recall, more memory)
        hnswEfConstruct : Int = 200      // Build-time quality (higher = better recall, slower build)
        hnswEfSearch : Int = 100         // Query-time quality (higher = better recall, slower query)

        // IVF parameters
        ivfNLists    : Int = 1024        // Number of clusters
        ivfNProbe    : Int = 32          // Clusters to search at query time

    FUNCTION selectForUseCase(datasetSize : Int,
                              latencyBudget : Milliseconds,
                              recallTarget : Float) -> VectorIndexConfig
        IF datasetSize < 10_000 THEN
            RETURN config(indexType = FLAT)    // Brute force is fine
        ELSE IF recallTarget > 0.98 THEN
            RETURN config(indexType = HNSW, hnswM = 32, hnswEfSearch = 200)
        ELSE IF datasetSize > 10_000_000 THEN
            RETURN config(indexType = IVF_PQ)  // Memory efficient at scale
        ELSE
            RETURN config(indexType = HNSW)    // Best default
        END IF
END CLASS
```

### Hybrid Search

Combine vector similarity with keyword matching for robust retrieval:

```
CLASS HybridSearchEngine
    PROPERTIES
        vectorIndex    : VectorIndex
        keywordIndex   : KeywordIndex       // BM25, Elasticsearch, etc.
        embedder       : EmbeddingService
        fusionStrategy : FusionStrategy

    FUNCTION search(query : String, topK : Int = 10,
                    filters : Map<String, Any> = {}) -> List<SearchResult>
        // 1. Vector search (semantic)
        queryEmbedding = embedder.embed(query)
        vectorResults = vectorIndex.search(queryEmbedding, topK = topK * 2,
                                           filters = filters)

        // 2. Keyword search (lexical)
        keywordResults = keywordIndex.search(query, topK = topK * 2,
                                             filters = filters)

        // 3. Fuse results
        fused = fusionStrategy.fuse(vectorResults, keywordResults)

        RETURN fused.take(topK)
END CLASS

// Reciprocal Rank Fusion — Simple, effective, parameter-free
CLASS ReciprocalRankFusion IMPLEMENTS FusionStrategy
    PROPERTIES
        k : Int = 60    // Smoothing constant

    FUNCTION fuse(resultSets : List<List<SearchResult>>) -> List<SearchResult>
        scores = Map<DocumentId, Float>()

        FOR EACH resultSet IN resultSets
            FOR EACH result, rank IN resultSet.withIndex()
                docId = result.documentId
                rrfScore = 1.0 / (k + rank + 1)
                scores[docId] = scores.getOrDefault(docId, 0.0) + rrfScore
            END FOR
        END FOR

        RETURN scores.sortedByValueDescending()
                     .map(docId -> getDocument(docId))
END CLASS
```

### Re-Ranking

Apply a more expensive, more accurate model to re-rank initial retrieval results:

```
CLASS CrossEncoderReranker
    PROPERTIES
        model       : CrossEncoderModel     // ms-marco-MiniLM, BGE-reranker
        batchSize   : Int = 32

    FUNCTION rerank(query : String,
                    candidates : List<SearchResult>,
                    topK : Int = 10) -> List<SearchResult>
        // Cross-encoder scores query-document pairs jointly
        // (more accurate than bi-encoder but O(n) not O(1))
        pairs = [(query, c.text) FOR c IN candidates]

        scores = EMPTY LIST
        FOR EACH batch IN partition(pairs, batchSize)
            batchScores = model.score(batch)
            scores.ADD_ALL(batchScores)
        END FOR

        // Re-order by cross-encoder scores
        scored = ZIP(candidates, scores)
        sorted = scored.sortByDescending(s -> s.score)
        RETURN sorted.take(topK)
END CLASS
```

### Chunking Strategies

How documents are split into chunks before embedding directly affects retrieval quality:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Chunking Strategies                               │
├──────────────────┬──────────────────────────────────────────────────┤
│  Strategy        │  Description                                     │
├──────────────────┼──────────────────────────────────────────────────┤
│  Fixed-Size      │  Split by token count (e.g., 512 tokens) with   │
│                  │  overlap. Simple but may split mid-sentence.     │
├──────────────────┼──────────────────────────────────────────────────┤
│  Recursive       │  Split by paragraph, then sentence, then token  │
│  Character       │  count. Respects natural text boundaries.       │
├──────────────────┼──────────────────────────────────────────────────┤
│  Semantic        │  Detect topic boundaries using embedding        │
│                  │  similarity between adjacent segments.           │
├──────────────────┼──────────────────────────────────────────────────┤
│  Document-       │  Use document structure (headers, sections) to  │
│  Structural      │  define chunk boundaries. Best for structured   │
│                  │  content like documentation or legal text.       │
├──────────────────┼──────────────────────────────────────────────────┤
│  Parent-Child    │  Index small chunks for retrieval precision but  │
│                  │  return the parent (larger) chunk for context.   │
│                  │  Best of both granularity levels.               │
├──────────────────┼──────────────────────────────────────────────────┤
│  Contextual      │  Use an LLM to prepend a short context summary  │
│                  │  to each chunk before embedding. Improves       │
│                  │  retrieval by adding document-level context.     │
└──────────────────┴──────────────────────────────────────────────────┘
```

```
CLASS ChunkingPipeline
    PROPERTIES
        strategy  : ChunkingStrategy
        chunkSize : Int = 512            // Target tokens per chunk
        overlap   : Int = 50             // Overlap between chunks

    FUNCTION chunk(document : Document) -> List<Chunk>
        MATCH strategy
            CASE RECURSIVE ->
                RETURN recursiveChunk(document.text, chunkSize, overlap,
                                     separators = ["\n\n", "\n", ". ", " "])
            CASE SEMANTIC ->
                RETURN semanticChunk(document.text, chunkSize,
                                     breakpointThreshold = 0.5)
            CASE PARENT_CHILD ->
                parentChunks = recursiveChunk(document.text, chunkSize * 3, 0)
                childChunks = EMPTY LIST
                FOR EACH parent IN parentChunks
                    children = recursiveChunk(parent.text, chunkSize, overlap)
                    FOR EACH child IN children
                        child.parentId = parent.id
                    END FOR
                    childChunks.ADD_ALL(children)
                END FOR
                RETURN childChunks
        END MATCH
END CLASS
```

### Multi-Tenancy and Filtering

Production vector search must support filtered queries and tenant isolation:

```
CLASS MultiTenantVectorStore
    PROPERTIES
        index     : VectorIndex
        tenantKey : String = "tenant_id"

    FUNCTION search(query : Vector, tenantId : String,
                    filters : Map<String, Any> = {},
                    topK : Int = 10) -> List<SearchResult>
        // Always scope to tenant
        allFilters = filters.merge({tenantKey: tenantId})

        // Pre-filtering: filter THEN search (exact, may miss vectors)
        // Post-filtering: search THEN filter (fast, may return < topK)
        // In-filter: search with filter applied during ANN traversal (best)
        RETURN index.search(query, topK = topK, filters = allFilters)

    FUNCTION upsert(documents : List<Document>,
                    tenantId : String) -> Void
        FOR EACH doc IN documents
            doc.metadata[tenantKey] = tenantId
        END FOR
        index.upsert(documents)
END CLASS
```

## Project Structure

```
src/
├── embedding/                      # Embedding Layer
│   ├── models/                     # Model wrappers (OpenAI, local)
│   ├── service/                    # Embedding service with batching
│   └── fine_tuning/                # Domain-specific fine-tuning
│
├── indexing/                       # Document Indexing Pipeline
│   ├── ingest/                     # Document loaders
│   ├── chunking/                   # Chunking strategies
│   ├── enrichment/                 # Metadata extraction
│   └── pipeline/                   # End-to-end indexing orchestrator
│
├── search/                         # Search Layer
│   ├── vector_search/              # ANN search
│   ├── keyword_search/             # BM25 / full-text search
│   ├── hybrid_search/              # Combined vector + keyword
│   ├── reranking/                  # Cross-encoder re-ranking
│   └── query_processing/          # Query rewriting, expansion
│
├── storage/                        # Vector Database Adapters
│   ├── pinecone/
│   ├── weaviate/
│   ├── qdrant/
│   ├── milvus/
│   ├── pgvector/
│   └── azure_ai_search/
│
├── multi_tenancy/                  # Tenant Isolation
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/                 # Retrieval quality benchmarks
```

## Key Design Considerations

### Distance Metrics

- **Cosine Similarity** — Most common for text embeddings; measures angle between vectors, invariant to magnitude
- **Dot Product** — Faster than cosine; equivalent when vectors are normalized
- **Euclidean (L2)** — Measures straight-line distance; sensitive to vector magnitude
- **Match your embedding model's training metric** — Using the wrong metric degrades recall significantly

### Embedding Model Fine-Tuning

General-purpose embeddings may underperform on specialized domains:

- **Contrastive fine-tuning** — Train on (query, relevant document, irrelevant document) triplets from your domain
- **Synthetic data** — Use an LLM to generate query-document pairs for training when labeled data is scarce
- **Hard negatives** — Mine difficult negative examples from existing retrieval results for more effective training

### Index Lifecycle Management

- **Incremental updates** — Most vector databases support upsert operations without full re-indexing
- **Index rebuilds** — Schedule periodic rebuilds when drift from incremental updates degrades quality
- **Embedding model versioning** — When the embedding model changes, all vectors must be re-embedded (no mixing model versions)
- **Backup and recovery** — Vector indexes can take hours to rebuild; maintain backups

### Evaluation Metrics

- **Recall@K** — Proportion of relevant documents in top-K results
- **MRR (Mean Reciprocal Rank)** — How high the first relevant result ranks
- **NDCG (Normalized Discounted Cumulative Gain)** — Quality-weighted ranking metric
- **Latency** — p50/p95/p99 query latency
- **Throughput** — Queries per second at target recall

## Benefits

1. **Semantic Understanding** — Finds relevant results even with no keyword overlap
2. **Multi-Modal Support** — Same architecture handles text, image, and audio search
3. **Scalability** — ANN indexes handle billions of vectors with sub-millisecond latency
4. **Flexibility** — Combine with keyword search, re-ranking, and filtering for robust retrieval
5. **Foundation for RAG** — Provides the retrieval layer for retrieval-augmented generation systems
6. **Language Agnostic** — Multi-lingual embeddings enable cross-language search

## Trade-offs

| Advantage                         | Consideration                                             |
| --------------------------------- | --------------------------------------------------------- |
| Semantic similarity matching      | Embedding quality determines result quality               |
| Sub-millisecond ANN query latency | ANN recall is approximate (not exact)                     |
| Scales to billions of vectors     | Memory requirements grow with dataset size and dimensions |
| Multi-modal support               | Different modalities need compatible embedding spaces     |
| Hybrid search robustness          | Two search systems to maintain and tune                   |
| Re-ranking improves precision     | Cross-encoder re-ranking adds latency                     |

## When to Use

✅ **Good fit for:**

- Retrieval-augmented generation (RAG) systems
- Semantic search over documents, knowledge bases, or product catalogs
- Recommendation engines based on content similarity
- Duplicate or near-duplicate detection
- Multi-modal search (text-to-image, image-to-image)
- Anomaly detection using embedding distance

❌ **Not ideal for:**

- Exact-match lookups where keyword search is sufficient (e.g., order IDs, product codes)
- Very small datasets (< 1,000 items) where brute-force search is fine
- Domains where keyword precision is more important than semantic understanding
- Applications that cannot tolerate approximate results

## References

- [Efficient and Robust Approximate Nearest Neighbor Search Using HNSW Graphs — Malkov & Yashunin (2020)](https://arxiv.org/abs/1603.09320)
- [DiskANN: Fast Accurate Billion-point Nearest Neighbor Search — Subramanya et al. (2019)](https://arxiv.org/abs/1904.07590)
- [MTEB: Massive Text Embedding Benchmark — Muennighoff et al. (2023)](https://arxiv.org/abs/2210.07316)
- [Matryoshka Representation Learning — Kusupati et al. (2022)](https://arxiv.org/abs/2205.13147)
- [ColBERT: Efficient and Effective Passage Search — Khattab & Zaharia (2020)](https://arxiv.org/abs/2004.12832)
- [Pinecone — Vector Database](https://www.pinecone.io/)
- [Weaviate — Vector Search Engine](https://weaviate.io/)
- [Qdrant — Vector Similarity Engine](https://qdrant.tech/)
- [Azure AI Search — Vector Search](https://learn.microsoft.com/en-us/azure/search/vector-search-overview)
