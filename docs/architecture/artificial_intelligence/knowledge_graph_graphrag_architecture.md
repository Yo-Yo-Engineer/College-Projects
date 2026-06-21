# Knowledge Graph / GraphRAG Architecture

## Overview

**Knowledge Graph / GraphRAG Architecture** defines the structural patterns for building AI systems that organize information as interconnected entities and relationships, then leverage that graph structure for enhanced retrieval and reasoning. Unlike flat-vector RAG — which retrieves isolated text chunks based on embedding similarity — GraphRAG uses entity extraction, relationship mapping, and community detection to answer questions that require synthesizing information across many documents.

The pattern gained significant industry traction through Microsoft Research's GraphRAG paper (2024), which demonstrated that graph-based approaches substantially outperform traditional RAG for global questions ("What are the main themes across all documents?") while maintaining strong performance on local questions.

Key principles:

- **Entities and Relationships as First-Class Citizens** — Model domain knowledge as typed nodes and edges, not just text chunks
- **Multi-Hop Reasoning** — Answer questions that require connecting facts across multiple documents or knowledge fragments
- **Community Structure** — Detect clusters of related entities and generate summaries at multiple levels of abstraction
- **Hybrid Retrieval** — Combine graph traversal with vector search for the best of both approaches
- **Incremental Construction** — Build and update the knowledge graph continuously as new documents arrive

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Knowledge Graph / GraphRAG Architecture                  │
│                                                                             │
│   Documents                                                                 │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Indexing Pipeline                              │      │
│   │                                                                  │      │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │      │
│   │  │  Chunk    │─▶│  Entity  │─▶│  Relation │─▶│  Community    │   │      │
│   │  │  & Parse  │  │  Extract │  │  Extract  │  │  Detection   │   │      │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────┬───────┘   │      │
│   │                                                     │           │      │
│   │                                              ┌──────▼───────┐  │      │
│   │                                              │  Community    │  │      │
│   │                                              │  Summarize   │  │      │
│   │                                              └──────────────┘  │      │
│   └──────────────────────────────────────────────────────────────────┘      │
│                          │                  │                                │
│                          ▼                  ▼                                │
│              ┌──────────────────┐  ┌──────────────────┐                    │
│              │  Graph Database   │  │  Vector Index     │                    │
│              │  (Entities,       │  │  (Chunk + Entity  │                    │
│              │   Relationships,  │  │   Embeddings)     │                    │
│              │   Communities)    │  │                    │                    │
│              └────────┬─────────┘  └────────┬──────────┘                    │
│                       │                      │                               │
│   User Query          │                      │                               │
│       │               │                      │                               │
│       ▼               ▼                      ▼                               │
│   ┌──────────────────────────────────────────────────────┐                  │
│   │                  Query Engine                         │                  │
│   │                                                      │                  │
│   │   ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │                  │
│   │   │  Local       │  │  Global       │  │  Hybrid   │ │                  │
│   │   │  Search      │  │  Search       │  │  Search   │ │                  │
│   │   │  (subgraph   │  │  (community   │  │  (graph + │ │                  │
│   │   │   traversal) │  │   summaries)  │  │   vector) │ │                  │
│   │   └─────────────┘  └──────────────┘  └───────────┘ │                  │
│   └──────────────────────────┬───────────────────────────┘                  │
│                              │                                              │
│                              ▼                                              │
│                    ┌──────────────────┐                                     │
│                    │  LLM Generation   │                                     │
│                    │  (with graph      │                                     │
│                    │   context)        │                                     │
│                    └──────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Entity and Relationship Extraction

The foundation of a knowledge graph is extracting structured entities and their relationships from unstructured text using an LLM:

```
// Entity and Relationship types
DATA Entity
    id          : String
    name        : String
    type        : String            // PERSON, ORG, CONCEPT, TECHNOLOGY, etc.
    description : String
    attributes  : Map<String, Any>
    sourceChunks: List<String>      // Provenance back to source documents
END DATA

DATA Relationship
    id          : String
    sourceEntity: String            // Entity ID
    targetEntity: String            // Entity ID
    type        : String            // WORKS_FOR, DEPENDS_ON, RELATED_TO, etc.
    description : String
    weight      : Float             // Strength/confidence
    sourceChunks: List<String>
END DATA

// LLM-based extraction
CLASS EntityRelationshipExtractor
    PROPERTIES
        llm         : LanguageModel
        entityTypes : List<String>
        relTypes    : List<String>

    FUNCTION extract(chunk : TextChunk) -> ExtractionResult
        prompt = buildExtractionPrompt(
            text = chunk.content,
            entityTypes = entityTypes,
            relationshipTypes = relTypes
        )
        response = llm.generate(prompt, outputFormat = JSON)
        entities = parseEntities(response)
        relationships = parseRelationships(response)

        // Validate and deduplicate
        entities = deduplicateEntities(entities)
        relationships = validateRelationships(relationships, entities)

        RETURN ExtractionResult(entities, relationships)
END CLASS
```

### Entity Resolution

Across multiple documents, the same entity may appear with different names or spellings. Entity resolution consolidates duplicates:

```
CLASS EntityResolver
    PROPERTIES
        embedder        : EmbeddingModel
        similarityThreshold : Float = 0.85

    FUNCTION resolve(entities : List<Entity>) -> List<Entity>
        // Group entities by type
        grouped = groupBy(entities, e -> e.type)
        resolved = EMPTY LIST

        FOR EACH type, group IN grouped
            embeddings = [embedder.encode(e.name + " " + e.description)
                         FOR e IN group]

            // Find clusters of similar entities
            clusters = clusterBySimilarity(group, embeddings,
                                           threshold = similarityThreshold)

            FOR EACH cluster IN clusters
                canonical = selectCanonical(cluster)   // Most frequent or longest description
                canonical.aliases = [e.name FOR e IN cluster IF e != canonical]
                canonical.sourceChunks = FLATTEN([e.sourceChunks FOR e IN cluster])
                resolved.ADD(canonical)
            END FOR
        END FOR

        RETURN resolved
END CLASS
```

### Community Detection and Summarization

Communities are clusters of densely connected entities. Summarizing communities enables answering global questions:

```
┌─────────────────────────────────────────────────────────────┐
│                  Community Hierarchy                         │
│                                                             │
│   Level 0 (leaf):   Individual entities and relationships   │
│                     └── Detailed, specific facts            │
│                                                             │
│   Level 1:          Small communities (5-20 entities)       │
│                     └── Topic-level summaries               │
│                                                             │
│   Level 2:          Large communities (20-100 entities)     │
│                     └── Theme-level summaries               │
│                                                             │
│   Level 3 (root):   Entire graph                            │
│                     └── Corpus-level summary                │
└─────────────────────────────────────────────────────────────┘
```

```
CLASS CommunityDetector
    PROPERTIES
        algorithm : CommunityAlgorithm    // Leiden, Louvain

    FUNCTION detect(graph : KnowledgeGraph,
                    maxLevels : Int = 3) -> List<CommunityLevel>
        levels = EMPTY LIST

        FOR level IN 0..maxLevels
            communities = algorithm.partition(graph, resolution = level)
            FOR EACH community IN communities
                community.summary = summarizeCommunity(community)
            END FOR
            levels.ADD(CommunityLevel(level, communities))
        END FOR

        RETURN levels

    FUNCTION summarizeCommunity(community : Community) -> String
        entities = community.getEntities()
        relationships = community.getRelationships()
        prompt = buildSummaryPrompt(entities, relationships)
        RETURN llm.generate(prompt)
END CLASS
```

### Query Strategies

GraphRAG supports multiple query strategies optimized for different question types:

```
// Local Search — For specific, entity-centric questions
CLASS LocalSearch
    PROPERTIES
        graph       : KnowledgeGraph
        vectorIndex : VectorIndex
        llm         : LanguageModel

    FUNCTION search(query : String, hops : Int = 2) -> Context
        // 1. Find seed entities via vector similarity
        seedEntities = vectorIndex.searchEntities(query, topK = 5)

        // 2. Expand subgraph around seed entities
        subgraph = graph.expandNeighborhood(seedEntities, maxHops = hops)

        // 3. Retrieve related text chunks
        relatedChunks = graph.getSourceChunks(subgraph.entities)

        // 4. Assemble context
        RETURN Context(
            entities = subgraph.entities,
            relationships = subgraph.relationships,
            textChunks = relatedChunks
        )
END CLASS

// Global Search — For broad, thematic questions
CLASS GlobalSearch
    PROPERTIES
        communities : List<CommunityLevel>
        llm         : LanguageModel

    FUNCTION search(query : String, level : Int = 1) -> Context
        // 1. Select community level based on query scope
        targetLevel = communities[level]

        // 2. Rank communities by relevance to query
        ranked = rankCommunities(query, targetLevel.communities)

        // 3. Use community summaries as context
        RETURN Context(
            communitySummaries = [c.summary FOR c IN ranked.top(10)]
        )
END CLASS

// Hybrid Search — Combines local specificity with global breadth
CLASS HybridSearch
    PROPERTIES
        localSearch  : LocalSearch
        globalSearch : GlobalSearch
        llm          : LanguageModel

    FUNCTION search(query : String) -> String
        localContext  = localSearch.search(query)
        globalContext = globalSearch.search(query)

        prompt = assemblePrompt(query, localContext, globalContext)
        RETURN llm.generate(prompt)
END CLASS
```

### Incremental Graph Updates

Production knowledge graphs must support continuous updates without full rebuilds:

```
CLASS IncrementalGraphUpdater
    PROPERTIES
        extractor   : EntityRelationshipExtractor
        resolver    : EntityResolver
        graph       : KnowledgeGraph
        communities : CommunityDetector

    FUNCTION addDocuments(documents : List<Document>) -> UpdateReport
        newEntities = EMPTY LIST
        newRelationships = EMPTY LIST

        FOR EACH doc IN documents
            chunks = chunkDocument(doc)
            FOR EACH chunk IN chunks
                result = extractor.extract(chunk)
                newEntities.ADD_ALL(result.entities)
                newRelationships.ADD_ALL(result.relationships)
            END FOR
        END FOR

        // Resolve against existing graph entities
        existingEntities = graph.getAllEntities()
        resolved = resolver.resolve(existingEntities + newEntities)

        // Apply changes
        graph.upsertEntities(resolved)
        graph.upsertRelationships(newRelationships)

        // Recompute affected communities (only impacted subgraphs)
        affectedNodes = getAffectedNodes(newEntities, newRelationships)
        communities.recomputePartial(graph, affectedNodes)

        RETURN UpdateReport(
            entitiesAdded = count(newEntities),
            entitiesMerged = count(resolved) - count(newEntities),
            relationshipsAdded = count(newRelationships)
        )
END CLASS
```

## Project Structure

```
src/
├── indexing/                       # Graph Construction Pipeline
│   ├── chunking/
│   ├── entity_extraction/
│   ├── relationship_extraction/
│   ├── entity_resolution/
│   └── community_detection/
│
├── storage/                        # Graph and Index Storage
│   ├── graph_database/             # Neo4j, Neptune, CosmosDB Gremlin
│   ├── vector_index/               # Embeddings for entities + chunks
│   └── document_store/             # Source document storage
│
├── query/                          # Query Strategies
│   ├── local_search/
│   ├── global_search/
│   ├── hybrid_search/
│   └── query_router/
│
├── summarization/                  # Community Summarization
│   ├── community_summarizer/
│   └── hierarchy_builder/
│
├── graph_ops/                      # Graph Operations
│   ├── traversal/
│   ├── subgraph_extraction/
│   └── incremental_update/
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/
```

## Key Design Considerations

### Graph Database Selection

- **Property Graph Databases** (Neo4j, Amazon Neptune, Azure Cosmos DB Gremlin) — Rich relationship modeling, Cypher/Gremlin query languages, ideal for complex traversals
- **RDF Triple Stores** (Apache Jena, Stardog) — Standards-based (SPARQL), strong for ontology-driven domains
- **In-Memory Graphs** (NetworkX, igraph) — Fast prototyping, suitable for smaller knowledge graphs that fit in memory
- **Embedded in Vector DB** — Some vector databases support graph-like metadata relationships; lighter-weight but less powerful

### Extraction Quality

Entity and relationship extraction quality directly determines graph usefulness:

- Use **structured output** (JSON mode) in the extraction LLM to ensure parseable results
- Provide **domain-specific entity types** and relationship types rather than using generic categories
- Include **few-shot examples** in extraction prompts from the target domain
- Implement **extraction validation** — verify entities appear in source text, relationships reference valid entities
- Run **multiple extraction passes** for high-value documents (extract, review, refine)

### Scalability

- **Indexing cost** — Entity extraction requires an LLM call per chunk; budget for indexing costs proportional to corpus size
- **Community recomputation** — Full community detection on large graphs expensive; use incremental approaches
- **Graph storage** — Large graphs with millions of entities need sharded storage or managed graph databases
- **Query latency** — Multi-hop traversals on large graphs can be slow; precompute common paths and cache subgraphs

### Ontology Design

Define a clear ontology before extraction to ensure consistency:

- **Entity types** — Keep to 5-15 well-defined types relevant to the domain
- **Relationship types** — Define directional relationships with clear semantics
- **Attributes** — Standardize entity attributes (dates, categories, metrics) for structured querying
- **Hierarchies** — Support is-a relationships for type hierarchies (e.g., Software → Database → Neo4j)

## Benefits

1. **Multi-Hop Reasoning** — Answer questions requiring synthesis across many documents
2. **Global Understanding** — Community summaries provide corpus-level insights unavailable to flat RAG
3. **Explainable Retrieval** — Graph paths show exactly how retrieved information connects
4. **Entity-Centric Search** — Query by entity and explore neighborhoods rather than just keyword matching
5. **Incremental Knowledge** — Add documents to evolve the graph without rebuilding
6. **Domain Modeling** — Typed entities and relationships map naturally to domain ontologies

## Trade-offs

| Advantage                          | Consideration                                       |
| ---------------------------------- | --------------------------------------------------- |
| Multi-hop reasoning capability     | LLM extraction cost per chunk during indexing       |
| Global thematic understanding      | Community detection adds indexing latency           |
| Explainable graph-path retrieval   | Entity resolution errors compound through the graph |
| Rich entity-centric queries        | Requires graph database infrastructure              |
| Incremental updates                | Ontology changes may require re-extraction          |
| Domain-specific knowledge modeling | Higher system complexity than flat-vector RAG       |

## When to Use

✅ **Good fit for:**

- Corpora where questions require synthesizing facts across many documents
- Knowledge-intensive domains with rich entity relationships (legal, biomedical, compliance)
- Research systems requiring exploration of entity networks and connections
- Enterprise knowledge bases with structured domain ontologies
- Use cases requiring explainable retrieval paths
- Systems answering both specific ("What does X do?") and thematic ("What are the major trends?") questions

❌ **Not ideal for:**

- Small document collections where flat-vector RAG is sufficient
- Real-time indexing requirements where extraction latency is prohibitive
- Domains with few meaningful entity relationships (e.g., simple FAQ systems)
- Cost-constrained scenarios where per-chunk LLM extraction is too expensive
- Rapidly changing content where graph reconstruction frequency is burdensome

## References

- [From Local to Global: A Graph RAG Approach — Microsoft Research (2024)](https://arxiv.org/abs/2404.16130)
- [GraphRAG: Unlocking LLM Discovery on Narrative Private Data — Microsoft (2024)](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)
- [Neo4j Graph Database](https://neo4j.com/)
- [Amazon Neptune — AWS](https://aws.amazon.com/neptune/)
- [Azure Cosmos DB — Graph API](https://learn.microsoft.com/en-us/azure/cosmos-db/gremlin/)
- [Leiden Algorithm for Community Detection — Traag et al. (2019)](https://arxiv.org/abs/1810.08473)
- [Knowledge Graphs — Hogan et al. (2021)](https://arxiv.org/abs/2003.02320)
