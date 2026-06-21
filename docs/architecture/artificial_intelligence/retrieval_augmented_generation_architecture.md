# Retrieval-Augmented Generation (RAG)

## Overview

**Retrieval-Augmented Generation (RAG)** is an architecture pattern that enhances large language model (LLM) responses by grounding them in external knowledge retrieved at query time. Instead of relying solely on the model's training data — which is static and may be outdated, incomplete, or hallucinated — RAG retrieves relevant documents from a knowledge base and includes them in the LLM's context window, producing more accurate, current, and verifiable answers.

The pattern was formalized by Lewis et al. (2020) at Facebook AI Research and has since become the dominant approach for enterprise AI applications that require domain-specific knowledge.

Key principles:

- **Grounding** — Anchor LLM responses in retrieved factual content to reduce hallucination
- **Separation of Knowledge and Reasoning** — The knowledge base (what the system knows) is decoupled from the language model (how the system reasons)
- **Freshness** — Knowledge can be updated independently of model retraining by adding, modifying, or removing documents in the index
- **Transparency** — Retrieved source documents provide citations, enabling users to verify answers
- **Cost Efficiency** — Augmenting a general-purpose model with retrieval is often more practical than fine-tuning a model on proprietary data

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RAG Architecture                                        │
│                                                                             │
│   User Query                                                                │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────┐         ┌──────────────────────────────────────────┐     │
│   │  Query        │────────▶│           Retrieval Pipeline             │     │
│   │  Processing   │         │                                          │     │
│   └──────────────┘         │  ┌────────────┐    ┌─────────────────┐  │     │
│                             │  │  Embedding  │───▶│  Vector Search   │  │     │
│                             │  │  Model      │    │  (Similarity)    │  │     │
│                             │  └────────────┘    └────────┬────────┘  │     │
│                             │                             │           │     │
│                             │                    ┌────────▼────────┐  │     │
│                             │                    │  Re-Ranking      │  │     │
│                             │                    │  (Relevance)     │  │     │
│                             │                    └────────┬────────┘  │     │
│                             └────────────────────────────│───────────┘     │
│                                                          │                  │
│                                                          ▼                  │
│                                                 Retrieved Documents         │
│                                                          │                  │
│       ┌──────────────┐                                   │                  │
│       │  System       │                                   │                  │
│       │  Prompt       │──────┐                            │                  │
│       └──────────────┘      │                            │                  │
│                              ▼                            ▼                  │
│                        ┌──────────────────────────────────────┐              │
│                        │         LLM (Generation)              │              │
│                        │   System prompt + retrieved context   │              │
│                        │   + user query → grounded answer      │              │
│                        └──────────────┬───────────────────────┘              │
│                                       │                                      │
│                                       ▼                                      │
│                                 Grounded Response                            │
│                              (with source citations)                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Ingestion Pipeline

Before retrieval can occur, documents must be processed and indexed:

```
┌──────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│  Source   │───▶│  Extract   │───▶│  Chunk     │───▶│  Embed     │───▶│  Index     │
│  Documents│    │  & Parse   │    │  (Split)   │    │  (Vectors) │    │  (Store)   │
└──────────┘    └───────────┘    └───────────┘    └───────────┘    └───────────┘
```

#### Document Loading and Parsing

```
// Document Loader — Extracts text from various source formats
INTERFACE DocumentLoader
    FUNCTION load(source : DataSource) -> List<RawDocument>
END INTERFACE

CLASS RawDocument
    PROPERTIES
        content    : String
        metadata   : Map<String, Any>
        sourceUri  : String
        format     : String          // PDF, HTML, DOCX, Markdown, etc.
        loadedAt   : DateTime
END CLASS

// Multi-format loader
CLASS MultiFormatDocumentLoader IMPLEMENTS DocumentLoader
    PROPERTIES
        parsers : Map<String, DocumentParser>

    FUNCTION load(source : DataSource) -> List<RawDocument>
        documents = EMPTY LIST
        FOR EACH file IN source.listFiles()
            parser = parsers.GET(file.format)
            IF parser IS NULL THEN
                LOG.WARN("No parser for format: " + file.format)
                CONTINUE
            END IF
            text = parser.extractText(file)
            documents.ADD(NEW RawDocument(
                content   = text,
                metadata  = file.metadata,
                sourceUri = file.uri,
                format    = file.format,
                loadedAt  = NOW()
            ))
        END FOR
        RETURN documents
END CLASS
```

#### Chunking Strategies

Splitting documents into appropriately sized pieces is critical for retrieval quality:

```
// Chunking Strategy — Splits documents into retrievable segments
INTERFACE ChunkingStrategy
    FUNCTION chunk(document : RawDocument) -> List<DocumentChunk>
END INTERFACE

DATA DocumentChunk
    content       : String
    metadata      : Map<String, Any>
    sourceUri     : String
    chunkIndex    : Integer
    tokenCount    : Integer
END DATA

// Recursive character text splitter with overlap
CLASS RecursiveTextSplitter IMPLEMENTS ChunkingStrategy
    PROPERTIES
        chunkSize    : Integer = 512    // Target tokens per chunk
        chunkOverlap : Integer = 50     // Overlapping tokens between chunks
        separators   : List<String> = ["\n\n", "\n", ". ", " "]

    FUNCTION chunk(document : RawDocument) -> List<DocumentChunk>
        chunks = EMPTY LIST
        text   = document.content

        // Split recursively using separator hierarchy
        segments = recursiveSplit(text, separators, chunkSize)

        index = 0
        FOR EACH segment IN segments
            chunks.ADD(NEW DocumentChunk(
                content    = segment,
                metadata   = document.metadata,
                sourceUri  = document.sourceUri,
                chunkIndex = index,
                tokenCount = countTokens(segment)
            ))
            index = index + 1
        END FOR

        RETURN chunks

    PRIVATE FUNCTION recursiveSplit(text, separators, maxSize) -> List<String>
        IF countTokens(text) <= maxSize THEN
            RETURN [text]
        END IF

        // Try each separator in order (most meaningful first)
        FOR EACH separator IN separators
            parts = text.SPLIT(separator)
            IF parts.SIZE() > 1 THEN
                RETURN mergeSplits(parts, separator, maxSize, chunkOverlap)
            END IF
        END FOR

        // Fallback: hard split by character count
        RETURN hardSplit(text, maxSize, chunkOverlap)
END CLASS

// Semantic chunking — Splits by topic boundaries using embeddings
CLASS SemanticChunker IMPLEMENTS ChunkingStrategy
    CONSTRUCTOR(embeddingModel : EmbeddingModel,
                similarityThreshold : Decimal = 0.75)

    FUNCTION chunk(document : RawDocument) -> List<DocumentChunk>
        sentences = splitIntoSentences(document.content)
        embeddings = embeddingModel.embedBatch(sentences)

        // Find breakpoints where topic shifts
        breakpoints = EMPTY LIST
        FOR i FROM 1 TO embeddings.SIZE() - 1
            similarity = cosineSimilarity(embeddings[i - 1], embeddings[i])
            IF similarity < similarityThreshold THEN
                breakpoints.ADD(i)
            END IF
        END FOR

        // Merge sentences between breakpoints into chunks
        RETURN mergeAtBreakpoints(sentences, breakpoints, document)
END CLASS
```

#### Embedding and Indexing

```
// Embedding Model — Converts text to dense vector representations
INTERFACE EmbeddingModel
    FUNCTION embed(text : String) -> Vector
    FUNCTION embedBatch(texts : List<String>) -> List<Vector>
END INTERFACE

// Vector Store — Stores and searches document embeddings
INTERFACE VectorStore
    FUNCTION upsert(chunks : List<DocumentChunk>, embeddings : List<Vector>) -> Void
    FUNCTION search(queryVector : Vector, topK : Integer,
                    filters : Map OR NULL) -> List<SearchResult>
    FUNCTION delete(filter : Map) -> Integer
END INTERFACE

DATA SearchResult
    chunk          : DocumentChunk
    score          : Decimal         // Similarity score (0 to 1)
    embedding      : Vector
END DATA

// Indexing Pipeline — Orchestrates the full ingestion process
CLASS IndexingPipeline
    CONSTRUCTOR(
        loader         : DocumentLoader,
        chunker        : ChunkingStrategy,
        embeddingModel : EmbeddingModel,
        vectorStore    : VectorStore
    )

    FUNCTION index(source : DataSource) -> IndexingResult
        // 1. Load documents
        documents = loader.load(source)

        // 2. Chunk documents
        allChunks = EMPTY LIST
        FOR EACH doc IN documents
            chunks = chunker.chunk(doc)
            allChunks.ADD_ALL(chunks)
        END FOR

        // 3. Generate embeddings
        texts = [chunk.content FOR EACH chunk IN allChunks]
        embeddings = embeddingModel.embedBatch(texts)

        // 4. Store in vector database
        vectorStore.upsert(allChunks, embeddings)

        RETURN NEW IndexingResult(
            documentsProcessed = documents.SIZE(),
            chunksCreated      = allChunks.SIZE(),
            indexedAt           = NOW()
        )
END CLASS
```

### Retrieval Pipeline

#### Query Processing and Retrieval

```
// Retrieval Pipeline — Finds relevant documents for a query
CLASS RetrievalPipeline
    CONSTRUCTOR(
        embeddingModel : EmbeddingModel,
        vectorStore    : VectorStore,
        reranker       : Reranker OR NULL
    )

    FUNCTION retrieve(query : String,
                      topK : Integer = 5,
                      filters : Map OR NULL) -> List<RetrievedDocument>

        // 1. Embed the query
        queryVector = embeddingModel.embed(query)

        // 2. Vector similarity search
        initialResults = vectorStore.search(
            queryVector = queryVector,
            topK        = topK * 3,       // Over-fetch for re-ranking
            filters     = filters
        )

        // 3. Re-rank for relevance (if reranker is available)
        IF reranker IS NOT NULL THEN
            rerankedResults = reranker.rerank(
                query   = query,
                results = initialResults,
                topK    = topK
            )
        ELSE
            rerankedResults = initialResults[0 .. topK]
        END IF

        // 4. Map to retrieved documents
        RETURN [NEW RetrievedDocument(
            content  = result.chunk.content,
            source   = result.chunk.sourceUri,
            score    = result.score,
            metadata = result.chunk.metadata
        ) FOR EACH result IN rerankedResults]
END CLASS
```

#### Advanced Retrieval Strategies

```
// Hybrid Search — Combines vector search with keyword search
CLASS HybridRetriever
    CONSTRUCTOR(
        vectorRetriever  : RetrievalPipeline,
        keywordRetriever : KeywordSearchEngine,
        alpha            : Decimal = 0.7     // Weight for vector vs keyword
    )

    FUNCTION retrieve(query : String, topK : Integer) -> List<RetrievedDocument>
        vectorResults  = vectorRetriever.retrieve(query, topK)
        keywordResults = keywordRetriever.search(query, topK)

        // Reciprocal Rank Fusion to merge results
        RETURN reciprocalRankFusion(
            resultSets = [vectorResults, keywordResults],
            weights    = [alpha, 1.0 - alpha],
            topK       = topK
        )
END CLASS

// Multi-Query Retrieval — Generates multiple query variants for broader recall
CLASS MultiQueryRetriever
    CONSTRUCTOR(
        llm       : LanguageModel,
        retriever : RetrievalPipeline
    )

    FUNCTION retrieve(query : String, topK : Integer) -> List<RetrievedDocument>
        // Generate query variations using the LLM
        variants = llm.generate(
            "Generate 3 alternative phrasings of this question " +
            "that might retrieve different relevant documents: " + query
        )

        // Retrieve for each variant
        allResults = EMPTY LIST
        FOR EACH variant IN parseVariants(variants)
            results = retriever.retrieve(variant, topK)
            allResults.ADD_ALL(results)
        END FOR

        // Deduplicate and rank
        RETURN deduplicateBySource(allResults, topK)
END CLASS
```

### Generation Pipeline

#### Prompt Construction and Response Generation

```
// RAG Generation Pipeline — Combines retrieval with LLM generation
CLASS RAGPipeline
    CONSTRUCTOR(
        retriever      : RetrievalPipeline,
        llm            : LanguageModel,
        systemPrompt   : String,
        maxContextDocs : Integer = 5
    )

    FUNCTION answer(userQuery : String,
                    conversationHistory : List<Message> OR NULL) -> RAGResponse

        // 1. Retrieve relevant context
        retrievedDocs = retriever.retrieve(
            query = userQuery,
            topK  = maxContextDocs
        )

        // 2. Build the prompt with retrieved context
        contextBlock = formatContext(retrievedDocs)

        messages = EMPTY LIST

        messages.ADD(NEW Message(
            role    = "system",
            content = systemPrompt + "\n\n" +
                      "Use the following context to answer the user's question. " +
                      "If the context does not contain enough information, say so. " +
                      "Always cite your sources.\n\n" +
                      "CONTEXT:\n" + contextBlock
        ))

        // Include conversation history if available
        IF conversationHistory IS NOT NULL THEN
            messages.ADD_ALL(conversationHistory)
        END IF

        messages.ADD(NEW Message(role = "user", content = userQuery))

        // 3. Generate grounded response
        response = llm.generate(messages)

        // 4. Extract citations from response
        citations = extractCitations(response, retrievedDocs)

        RETURN NEW RAGResponse(
            answer    = response.content,
            sources   = retrievedDocs,
            citations = citations
        )

    PRIVATE FUNCTION formatContext(docs : List<RetrievedDocument>) -> String
        blocks = EMPTY LIST
        FOR i FROM 0 TO docs.SIZE() - 1
            blocks.ADD(
                "[Source " + (i + 1) + ": " + docs[i].source + "]\n" +
                docs[i].content + "\n"
            )
        END FOR
        RETURN JOIN(blocks, separator = "\n---\n")
END CLASS
```

### Evaluation

RAG systems require evaluation across both retrieval and generation:

```
// RAG Evaluation Framework
CLASS RAGEvaluator
    CONSTRUCTOR(
        llmJudge    : LanguageModel,     // LLM-as-judge for evaluation
        testDataset : List<RAGTestCase>
    )

    FUNCTION evaluate(ragPipeline : RAGPipeline) -> EvaluationReport
        results = EMPTY LIST

        FOR EACH testCase IN testDataset
            response = ragPipeline.answer(testCase.query)

            metrics = NEW EvalMetrics()

            // Retrieval quality
            metrics.contextRelevance = scoreContextRelevance(
                query    = testCase.query,
                contexts = response.sources
            )
            metrics.contextRecall = computeRecall(
                retrieved     = response.sources,
                expectedDocs  = testCase.relevantDocuments
            )

            // Generation quality
            metrics.faithfulness = scoreFaithfulness(
                answer   = response.answer,
                contexts = response.sources
            )
            metrics.answerRelevance = scoreAnswerRelevance(
                query  = testCase.query,
                answer = response.answer
            )

            // Hallucination detection
            metrics.hallucination = detectHallucination(
                answer   = response.answer,
                contexts = response.sources
            )

            results.ADD(metrics)
        END FOR

        RETURN aggregateMetrics(results)

    PRIVATE FUNCTION scoreFaithfulness(answer, contexts) -> Decimal
        // Use LLM-as-judge to verify all claims in the answer
        // are supported by the retrieved context
        verdict = llmJudge.generate(
            "Given the following context and answer, determine what fraction " +
            "of the claims in the answer are supported by the context. " +
            "Return a score between 0.0 and 1.0.\n\n" +
            "CONTEXT: " + formatDocs(contexts) + "\n\n" +
            "ANSWER: " + answer
        )
        RETURN parseScore(verdict)
END CLASS
```

## Project Structure

```
src/
├── ingestion/                      # Document Ingestion Pipeline
│   ├── loaders/                    # PDF, HTML, DOCX, Markdown parsers
│   ├── chunking/                   # Chunking strategies
│   ├── embedding/                  # Embedding model wrappers
│   └── indexing/                   # Vector store indexing
│
├── retrieval/                      # Retrieval Pipeline
│   ├── vector_search/
│   ├── keyword_search/
│   ├── hybrid/                     # Hybrid search strategies
│   ├── reranking/                  # Cross-encoder re-rankers
│   └── multi_query/                # Query expansion
│
├── generation/                     # Generation Pipeline
│   ├── prompt_templates/
│   ├── llm_clients/
│   ├── citation_extraction/
│   └── response_formatting/
│
├── evaluation/                     # RAG Evaluation
│   ├── metrics/                    # Faithfulness, relevance, recall
│   ├── test_datasets/
│   └── judges/                     # LLM-as-judge implementations
│
├── knowledge/                      # Knowledge Management
│   ├── sources/                    # Data source connectors
│   ├── refresh/                    # Incremental update pipelines
│   └── metadata/                   # Document metadata management
│
├── api/                            # Serving Layer
│   ├── endpoints/
│   └── middleware/                  # Auth, rate limiting, caching
│
├── config/
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/
```

## Key Design Considerations

### Chunking Strategy Selection

The chunking approach significantly impacts retrieval quality:

| Strategy            | Best For                                    | Trade-off                         |
| ------------------- | ------------------------------------------- | --------------------------------- |
| Fixed-size          | Simple, predictable chunk sizes             | May split semantic units          |
| Recursive splitting | General-purpose documents                   | Requires good separator hierarchy |
| Semantic chunking   | Topic-diverse documents                     | Slower; requires embedding step   |
| Document-aware      | Structured docs (Markdown, HTML headings)   | Format-specific implementation    |
| Sentence-window     | Fine-grained retrieval with broader context | More complex retrieval logic      |

### When to Use Hybrid Search

- **Vector search alone** — Works well when queries and documents use similar vocabulary
- **Keyword search alone** — Works well for exact-match queries (product IDs, error codes)
- **Hybrid** — Best when queries vary between semantic and keyword-based; use reciprocal rank fusion to merge

### Caching Strategies

- **Embedding cache** — Cache query embeddings for repeated queries
- **Retrieval cache** — Cache retrieved document sets for popular queries
- **Response cache** — Cache full responses for identical query + context combinations (use with caution as knowledge updates may invalidate)

### Knowledge Freshness

Design incremental update pipelines rather than full re-indexing:

- **Add** — Index new documents as they arrive
- **Update** — Re-embed and re-index modified documents (detect via hashes or timestamps)
- **Delete** — Remove stale documents from the index

## Benefits

1. **Reduced Hallucination** — Grounding responses in retrieved facts significantly improves accuracy
2. **Current Knowledge** — Knowledge base can be updated without retraining the model
3. **Domain Specificity** — Enables general-purpose models to answer domain-specific questions
4. **Verifiability** — Source citations allow users to verify and trust responses
5. **Cost Efficiency** — Often cheaper than fine-tuning or training custom models
6. **Data Privacy** — Proprietary data stays in your vector store; no need to send it for model training

## Trade-offs

| Advantage                         | Consideration                                         |
| --------------------------------- | ----------------------------------------------------- |
| Grounded, factual responses       | Retrieval quality directly limits answer quality      |
| Updatable knowledge base          | Chunking strategy significantly impacts results       |
| Source citations for verification | Added latency from retrieval + generation steps       |
| No model retraining needed        | Vector store requires ongoing maintenance             |
| Works with any LLM                | Context window limits the amount of retrieved context |

## When to Use

✅ **Good fit for:**

- Enterprise knowledge bases and documentation search
- Customer support chatbots requiring accurate, cited answers
- Legal, medical, or compliance applications needing verifiable responses
- Internal tools answering questions about company processes and policies
- Applications requiring up-to-date information beyond model training data

❌ **Not ideal for:**

- Creative generation tasks (storytelling, brainstorming) where grounding is unnecessary
- Tasks where the LLM's training data is sufficient and current
- Real-time applications with extremely strict latency requirements (sub-100ms)
- Small, static knowledge bases where fine-tuning may be more effective
- Tasks requiring deep multi-hop reasoning across many documents (consider agentic RAG)

### RAG Variants

| Variant      | Description                                                        |
| ------------ | ------------------------------------------------------------------ |
| Naive RAG    | Basic retrieve-then-generate pipeline                              |
| Advanced RAG | Adds query rewriting, re-ranking, and hybrid search                |
| Modular RAG  | Composable retrieval, generation, and routing modules              |
| Agentic RAG  | Agent decides when and how to retrieve, with iterative refinement  |
| Graph RAG    | Retrieves from knowledge graphs in addition to vector stores       |
| Self-RAG     | Model decides whether to retrieve and self-evaluates its responses |

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks — Lewis et al. (2020)](https://arxiv.org/abs/2005.11401)
- [Baseline Foundry Chat Reference Architecture — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat)
- [RAG and Generative AI — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Choose an Azure Service for Vector Search — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/vector-search)
- [Chunking Large Documents for Vector Search — Greg Kamradt](https://www.pinecone.io/learn/chunking-strategies/)
- [RAGAS: Automated Evaluation of Retrieval-Augmented Generation — Shahul et al.](https://arxiv.org/abs/2309.15217)
