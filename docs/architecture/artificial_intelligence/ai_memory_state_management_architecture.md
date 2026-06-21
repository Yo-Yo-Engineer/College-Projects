# AI Memory & State Management Architecture

## Overview

**AI Memory & State Management Architecture** defines the structural patterns for giving AI systems persistent, contextual memory that extends beyond a single interaction. While LLMs have no inherent memory between requests — each call is stateless — production AI applications require remembering conversation history, user preferences, learned facts, and past decisions to deliver coherent, personalized, and improving experiences.

This architecture addresses the full spectrum of memory needs: from short-term conversation buffers that manage context windows, through session-level state for multi-turn interactions, to long-term persistent memory that spans days, months, or an entire user relationship.

Key principles:

- **Memory Is an Architectural Concern** — Memory management should be a dedicated subsystem, not ad-hoc prompt stuffing
- **Tiered Storage** — Different memory types have different retention, retrieval, and capacity requirements
- **Selective Recall** — Not all memories are relevant to every query; retrieve only what matters for the current context
- **Graceful Forgetting** — Memory systems must prune, consolidate, and forget to remain useful and manageable
- **Privacy by Design** — Users must be able to inspect, correct, and delete their memories

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  AI Memory & State Management Architecture                   │
│                                                                             │
│   User Input + Context                                                      │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Memory Manager                                 │      │
│   │                                                                  │      │
│   │   ┌────────────┐   ┌────────────┐   ┌─────────────────────┐    │      │
│   │   │  Working     │   │  Episodic   │   │  Semantic            │    │      │
│   │   │  Memory      │   │  Memory     │   │  Memory              │    │      │
│   │   │              │   │             │   │                     │    │      │
│   │   │  Current     │   │  Past       │   │  Facts, knowledge, │    │      │
│   │   │  conversation│   │  interactions│  │  user preferences  │    │      │
│   │   │  context     │   │  & events   │   │  & learned info    │    │      │
│   │   └──────┬───────┘   └──────┬──────┘   └──────────┬────────┘    │      │
│   │          │                  │                      │             │      │
│   │          └──────────────────┼──────────────────────┘             │      │
│   │                            │                                     │      │
│   │                     ┌──────▼──────┐                              │      │
│   │                     │  Memory      │                              │      │
│   │                     │  Retriever   │   Selects relevant memories │      │
│   │                     └──────┬──────┘                              │      │
│   └────────────────────────────┼─────────────────────────────────────┘      │
│                                │                                            │
│                                ▼                                            │
│                    ┌──────────────────────┐                                 │
│                    │  Context Assembler    │  Combines input + memories     │
│                    │                      │  within token budget            │
│                    └──────────┬───────────┘                                 │
│                               │                                             │
│                               ▼                                             │
│                    ┌──────────────────────┐                                 │
│                    │  LLM / Agent          │                                 │
│                    └──────────┬───────────┘                                 │
│                               │                                             │
│                               ▼                                             │
│                    ┌──────────────────────┐                                 │
│                    │  Memory Writer        │  Extracts new memories         │
│                    │                      │  from the interaction           │
│                    └──────────────────────┘                                 │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Memory      │  │  Retention    │  │  Privacy &         │  │
│                   │ Index       │  │  Policy       │  │  User Control      │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Memory Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Memory Type Taxonomy                            │
├──────────────────┬──────────────────────────────────────────────────┤
│  Type            │  Description                                     │
├──────────────────┼──────────────────────────────────────────────────┤
│  Working Memory  │  Current conversation context. The messages and │
│  (Short-term)    │  context actively in the LLM's context window.  │
│                  │  Lifespan: current request/conversation.         │
├──────────────────┼──────────────────────────────────────────────────┤
│  Episodic Memory │  Records of past interactions and events.        │
│                  │  "Last week, the user asked about deployment."   │
│                  │  Lifespan: days to months.                       │
├──────────────────┼──────────────────────────────────────────────────┤
│  Semantic Memory │  Distilled facts and knowledge about the user,  │
│                  │  domain, or system. "User prefers Python."       │
│                  │  Lifespan: persistent, updated over time.        │
├──────────────────┼──────────────────────────────────────────────────┤
│  Procedural      │  Learned behaviors and patterns. "When user     │
│  Memory          │  asks for code, include tests."                  │
│                  │  Lifespan: persistent, refined over time.        │
└──────────────────┴──────────────────────────────────────────────────┘
```

### Working Memory (Conversation Buffer)

Manage the current conversation within the LLM's context window:

```
// Conversation Buffer — Manages message history within token limits
CLASS ConversationBuffer
    PROPERTIES
        messages      : List<Message>
        maxTokens     : Int             // Context window budget for history
        tokenizer     : Tokenizer
        strategy      : TruncationStrategy

    FUNCTION addMessage(message : Message) -> Void
        messages.ADD(message)
        trimToFit()

    FUNCTION getContext() -> List<Message>
        RETURN messages

    FUNCTION trimToFit() -> Void
        WHILE countTokens(messages) > maxTokens
            MATCH strategy
                CASE SLIDING_WINDOW ->
                    // Remove oldest messages (keep system prompt)
                    messages.REMOVE_FIRST_NON_SYSTEM()

                CASE SUMMARIZE_AND_TRIM ->
                    // Summarize older messages, keep recent ones
                    oldMessages = messages.takeOldest(messages.size / 2)
                    summary = summarize(oldMessages)
                    messages.replaceOldest(SummaryMessage(summary))

                CASE TOKEN_BUDGET ->
                    // Remove messages by priority (keep system, recent, pinned)
                    messages.REMOVE_LOWEST_PRIORITY()
            END MATCH
        END WHILE

    FUNCTION countTokens(msgs : List<Message>) -> Int
        RETURN SUM(tokenizer.count(m.content) FOR m IN msgs)
END CLASS
```

### Summarization-Based Memory

When conversations exceed context limits, summarize older exchanges to preserve key information:

```
CLASS SummarizingMemory
    PROPERTIES
        recentMessages    : List<Message>       // Full recent messages
        summaryChain      : List<String>        // Progressive summaries
        summarizer        : LanguageModel
        recentWindowSize  : Int = 10            // Messages to keep in full
        summaryTrigger    : Int = 20            // Messages before summarizing

    FUNCTION addMessage(message : Message) -> Void
        recentMessages.ADD(message)

        IF recentMessages.size() >= summaryTrigger THEN
            consolidate()
        END IF

    FUNCTION consolidate() -> Void
        // Messages to summarize (keep the most recent window intact)
        toSummarize = recentMessages.dropLast(recentWindowSize)
        recentMessages = recentMessages.takeLast(recentWindowSize)

        // Build progressive summary
        previousSummary = summaryChain.lastOrNull() OR ""
        newSummary = summarizer.generate(
            prompt = buildSummaryPrompt(previousSummary, toSummarize)
        )
        summaryChain.ADD(newSummary)

    FUNCTION getContext() -> MemoryContext
        RETURN MemoryContext(
            summary = summaryChain.lastOrNull(),
            recentMessages = recentMessages
        )
END CLASS
```

### Episodic Memory

Store and retrieve records of past interactions for long-term context:

```
DATA Episode
    id          : String
    timestamp   : DateTime
    sessionId   : String
    userId      : String
    summary     : String            // Concise description of the interaction
    keyTopics   : List<String>
    outcome     : String            // What was achieved
    sentiment   : String            // User satisfaction signal
    embedding   : Vector            // For semantic retrieval
END DATA

CLASS EpisodicMemoryStore
    PROPERTIES
        store       : VectorStore
        extractor   : EpisodeExtractor      // LLM-based

    FUNCTION recordEpisode(conversation : List<Message>,
                           userId : String) -> Episode
        // Extract episode metadata from conversation
        extracted = extractor.extract(conversation)
        episode = Episode(
            id = generateId(),
            timestamp = NOW(),
            userId = userId,
            summary = extracted.summary,
            keyTopics = extracted.topics,
            outcome = extracted.outcome,
            sentiment = extracted.sentiment,
            embedding = embedder.embed(extracted.summary)
        )
        store.upsert(episode)
        RETURN episode

    FUNCTION recall(query : String, userId : String,
                    topK : Int = 5) -> List<Episode>
        queryEmbedding = embedder.embed(query)
        results = store.search(
            vector = queryEmbedding,
            filter = {"userId": userId},
            topK = topK
        )
        RETURN results

    FUNCTION recallByTime(userId : String,
                          since : DateTime) -> List<Episode>
        RETURN store.query(
            filter = {"userId": userId, "timestamp": {"gte": since}},
            orderBy = "timestamp DESC"
        )
END CLASS
```

### Semantic Memory

Maintain a structured knowledge base of facts learned about and from the user:

```
DATA MemoryFact
    id          : String
    userId      : String
    subject     : String            // What the fact is about
    predicate   : String            // The relationship or attribute
    value       : String            // The fact content
    confidence  : Float             // How confident we are
    source      : String            // Where this was learned
    createdAt   : DateTime
    lastUsed    : DateTime
    useCount    : Int
    embedding   : Vector
END DATA

CLASS SemanticMemoryStore
    PROPERTIES
        store       : VectorStore
        extractor   : FactExtractor         // LLM-based

    FUNCTION learnFromConversation(conversation : List<Message>,
                                   userId : String) -> List<MemoryFact>
        // Extract facts from conversation
        facts = extractor.extractFacts(conversation)
        newFacts = EMPTY LIST

        FOR EACH fact IN facts
            // Check for existing conflicting facts
            existing = findConflicting(fact, userId)
            IF existing != NULL THEN
                // Update existing fact if new information is more recent/reliable
                IF fact.confidence >= existing.confidence THEN
                    existing.value = fact.value
                    existing.confidence = fact.confidence
                    existing.source = fact.source
                    store.update(existing)
                END IF
            ELSE
                fact.userId = userId
                fact.embedding = embedder.embed(fact.subject + " " + fact.value)
                store.upsert(fact)
                newFacts.ADD(fact)
            END IF
        END FOR

        RETURN newFacts

    FUNCTION recall(query : String, userId : String,
                    topK : Int = 10) -> List<MemoryFact>
        queryEmbedding = embedder.embed(query)
        results = store.search(
            vector = queryEmbedding,
            filter = {"userId": userId},
            topK = topK
        )
        // Update usage statistics
        FOR EACH result IN results
            result.lastUsed = NOW()
            result.useCount += 1
            store.update(result)
        END FOR
        RETURN results
END CLASS
```

### Memory Retrieval and Context Assembly

Select and assemble relevant memories into the LLM's context:

```
CLASS MemoryRetriever
    PROPERTIES
        workingMemory   : ConversationBuffer
        episodicMemory  : EpisodicMemoryStore
        semanticMemory  : SemanticMemoryStore
        tokenBudget     : Int

    FUNCTION assembleContext(query : String,
                             userId : String) -> AssembledContext
        budget = TokenBudget(total = tokenBudget)

        // 1. Working memory (highest priority — always included)
        working = workingMemory.getContext()
        budget.deduct(countTokens(working))

        // 2. Semantic memory (user facts and preferences)
        facts = semanticMemory.recall(query, userId, topK = 20)
        relevantFacts = filterByRelevance(facts, query, threshold = 0.7)
        factText = formatFacts(relevantFacts)
        budget.deduct(countTokens(factText))

        // 3. Episodic memory (relevant past interactions)
        IF budget.remaining() > MIN_EPISODE_TOKENS THEN
            episodes = episodicMemory.recall(query, userId, topK = 5)
            episodeText = formatEpisodes(episodes)
            episodeText = truncateToFit(episodeText, budget.remaining())
        ELSE
            episodeText = ""
        END IF

        RETURN AssembledContext(
            systemPromptAdditions = factText,
            conversationHistory = working,
            episodicContext = episodeText
        )
END CLASS
```

### Memory Consolidation and Forgetting

Over time, memories must be pruned, merged, and consolidated:

```
CLASS MemoryConsolidator
    PROPERTIES
        semanticStore  : SemanticMemoryStore
        episodicStore  : EpisodicMemoryStore
        retentionPolicy: RetentionPolicy

    FUNCTION consolidate(userId : String) -> ConsolidationReport
        report = ConsolidationReport()

        // 1. Merge duplicate or similar facts
        facts = semanticStore.getAllForUser(userId)
        clusters = clusterBySimilarity(facts, threshold = 0.9)
        FOR EACH cluster IN clusters
            IF cluster.size() > 1 THEN
                merged = mergeFacts(cluster)
                semanticStore.replaceAll(cluster, merged)
                report.factsMerged += cluster.size() - 1
            END IF
        END FOR

        // 2. Decay unused memories
        stale = semanticStore.query(
            filter = {"userId": userId,
                      "lastUsed": {"lt": NOW() - retentionPolicy.stalePeriod},
                      "useCount": {"lt": retentionPolicy.minUseCount}}
        )
        FOR EACH fact IN stale
            fact.confidence *= retentionPolicy.decayFactor
            IF fact.confidence < retentionPolicy.forgetThreshold THEN
                semanticStore.delete(fact)
                report.factsRemoved += 1
            ELSE
                semanticStore.update(fact)
                report.factsDecayed += 1
            END IF
        END FOR

        // 3. Summarize old episodes
        oldEpisodes = episodicStore.recallByTime(
            userId, before = NOW() - retentionPolicy.episodeSummaryAge
        )
        IF oldEpisodes.size() > retentionPolicy.episodeBatchSize THEN
            summary = summarizeEpisodes(oldEpisodes)
            episodicStore.replaceWithSummary(oldEpisodes, summary)
            report.episodesConsolidated += oldEpisodes.size()
        END IF

        RETURN report
END CLASS
```

## Project Structure

```
src/
├── memory/                         # Memory Subsystem
│   ├── working/                    # Working Memory (Conversation)
│   │   ├── buffer/
│   │   ├── summarizer/
│   │   └── truncation/
│   ├── episodic/                   # Episodic Memory
│   │   ├── store/
│   │   ├── extractor/
│   │   └── recall/
│   ├── semantic/                   # Semantic Memory (Facts)
│   │   ├── store/
│   │   ├── extractor/
│   │   └── conflict_resolver/
│   └── procedural/                 # Procedural Memory (Patterns)
│       ├── store/
│       └── pattern_matcher/
│
├── retrieval/                      # Memory Retrieval
│   ├── retriever/
│   ├── context_assembler/
│   └── token_budgeting/
│
├── lifecycle/                      # Memory Lifecycle
│   ├── writer/                     # Memory extraction from conversations
│   ├── consolidation/             # Merge and prune
│   ├── retention/                  # Policy enforcement
│   └── decay/                      # Confidence decay
│
├── privacy/                        # User Controls
│   ├── inspection/                 # View memories
│   ├── correction/                 # Edit memories
│   └── deletion/                   # Delete memories
│
├── storage/                        # Storage Backends
│   ├── vector_store/
│   └── document_store/
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/
```

## Key Design Considerations

### Token Budget Management

Memory competes with the user query and system prompt for context window space:

- **Prioritize by type** — Working memory > semantic facts > episodic context
- **Dynamic allocation** — Adjust memory allocation based on query length and complexity
- **Relevance filtering** — Only include memories with high relevance scores; do not dump all memories into context
- **Compression** — Use summaries and fact extraction rather than raw conversation replay

### Conflict Resolution

Users may provide contradictory information over time:

- **Recency bias** — More recent information generally supersedes older information
- **Confidence scoring** — Explicitly stated facts outweigh inferred facts
- **Source tracking** — Track where each fact came from to resolve conflicts
- **User confirmation** — For high-stakes conflicts, ask the user to confirm

### Privacy and User Control

Memory creates privacy obligations:

- **Transparency** — Users should be able to view what the system remembers about them
- **Correction** — Users should be able to correct inaccurate memories
- **Deletion** — Users should be able to delete specific memories or all memories
- **Scope control** — Users should be able to decide what types of information are remembered
- **Cross-session consent** — Obtain explicit consent before persisting information across sessions

### Memory Extraction Quality

The quality of extracted memories directly determines system usefulness:

- **Explicit vs. implicit** — "My name is Alex" is explicit; "I usually deploy on Fridays" is implicit and harder to extract
- **Fact vs. opinion** — "Python is best" is an opinion/preference; "I use Python 3.12" is a fact
- **Temporal validity** — "I'm working on project X" may expire; "My email is..." likely does not
- **Extraction validation** — Verify extracted facts against the conversation before storing

## Benefits

1. **Personalization** — Systems adapt to individual users over time
2. **Continuity** — Multi-turn and multi-session conversations feel coherent
3. **Efficiency** — Avoid re-asking for information the system should already know
4. **Learning** — Systems improve through accumulated interaction history
5. **Context Preservation** — Important context survives beyond conversation limits
6. **User Trust** — Remembering preferences and history builds user confidence

## Trade-offs

| Advantage                             | Consideration                                     |
| ------------------------------------- | ------------------------------------------------- |
| Personalized, context-aware responses | Memory extraction adds LLM calls and cost         |
| Multi-session continuity              | Storage and retrieval infrastructure required     |
| Improved efficiency over time         | Memory retrieval adds latency to each request     |
| Richer context for generation         | Irrelevant memories can confuse or bias the model |
| User relationship building            | Privacy obligations and user control requirements |
| Conflict resolution over time         | Stale or incorrect memories can degrade quality   |

## When to Use

✅ **Good fit for:**

- Personal AI assistants that interact with the same users over time
- Customer support systems that need to remember user history and preferences
- AI agents that build knowledge through repeated interactions
- Coding assistants that learn project context and user preferences
- Enterprise chatbots that accumulate domain-specific knowledge
- Multi-session workflows where context must persist between sessions

❌ **Not ideal for:**

- Single-turn, stateless Q&A systems
- Anonymous or public-facing systems where personalization is not needed
- Privacy-restricted environments where persisting user data is not permitted
- Short-lived interactions where the overhead of memory management is not justified

## References

- [MemGPT: Towards LLMs as Operating Systems — Packer et al. (2023)](https://arxiv.org/abs/2310.08560)
- [Generative Agents: Interactive Simulacra of Human Behavior — Park et al. (2023)](https://arxiv.org/abs/2304.03442)
- [LangChain Memory Documentation](https://python.langchain.com/docs/modules/memory/)
- [Mem0: The Memory Layer for Personalized AI](https://github.com/mem0ai/mem0)
- [Cognitive Architectures for Language Agents — Sumers et al. (2023)](https://arxiv.org/abs/2309.02427)
- [A Survey on the Memory Mechanism of Large Language Model Based Agents — Zhang et al. (2024)](https://arxiv.org/abs/2404.13501)
