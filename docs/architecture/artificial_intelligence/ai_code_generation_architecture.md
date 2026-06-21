# AI Code Generation Architecture

## Overview

**AI Code Generation Architecture** defines the structural patterns for building systems that use large language models and retrieval-augmented pipelines to generate, complete, review, test, and transform code. These systems operate at the intersection of language understanding and software engineering — leveraging repository context, AST analysis, and developer intent to produce code that is syntactically correct, stylistically consistent, and functionally appropriate.

Modern AI code generation spans a spectrum from inline autocompletion (single-line suggestions triggered as the developer types) to autonomous multi-file agents that plan, implement, test, and iterate on complex tasks across an entire repository.

Key principles:

- **Context is king** — Quality of generated code scales directly with the quality and relevance of the context provided to the model
- **Repository awareness** — Code generation should understand project structure, dependency graphs, coding conventions, and existing implementations
- **Grounded generation** — Generated code should reference and reuse existing functions, types, and patterns rather than inventing duplicates
- **Iterative refinement** — Code generation benefits from feedback loops: type checking, linting, test execution, and self-review
- **Safety boundaries** — Generated code must be validated before execution, and destructive operations require human approval

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AI Code Generation Architecture                         │
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Developer Interface                            │      │
│   │   ┌──────────┐  ┌──────────────┐  ┌─────────────────────────┐  │      │
│   │   │  IDE /    │  │  CLI /       │  │  Chat /                 │  │      │
│   │   │  Editor   │  │  Terminal    │  │  Conversational        │  │      │
│   │   │  Plugin   │  │  Agent       │  │  Interface             │  │      │
│   │   └────┬─────┘  └──────┬───────┘  └───────────┬─────────────┘  │      │
│   └────────┼───────────────┼──────────────────────┼──────────────────┘      │
│            │               │                      │                         │
│            └───────────────┼──────────────────────┘                         │
│                            ▼                                                │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Context Engine                                 │      │
│   │   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │      │
│   │   │  Repository   │  │  Semantic     │  │  Active File       │  │      │
│   │   │  Indexer      │  │  Search       │  │  Context           │  │      │
│   │   │  (AST, Deps)  │  │  (Embeddings) │  │  (Cursor, Edits)  │  │      │
│   │   └──────────────┘  └──────────────┘  └────────────────────┘  │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Generation Pipeline                            │      │
│   │   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │      │
│   │   │  Prompt       │  │  LLM          │  │  Post-Processing  │  │      │
│   │   │  Assembly     │  │  Inference    │  │  & Validation     │  │      │
│   │   └──────────────┘  └──────────────┘  └────────────────────┘  │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Feedback Loop                                  │      │
│   │   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │      │
│   │   │  Type Check / │  │  Test         │  │  Self-Review /    │  │      │
│   │   │  Lint / Parse │  │  Execution    │  │  Reflection       │  │      │
│   │   └──────────────┘  └──────────────┘  └────────────────────┘  │      │
│   └──────────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Context Gathering

The most critical subsystem — retrieves relevant code context to ground the generation:

```
CLASS ContextEngine
    PROPERTIES
        repoIndex       : RepositoryIndex
        embeddingIndex  : VectorIndex
        fileGraph       : DependencyGraph

    FUNCTION gatherContext(request : CodeGenRequest) -> CodeContext
        context = CodeContext()

        // 1. Active file context — cursor position, surrounding code, recent edits
        context.activeFile = extractActiveFileContext(
            file = request.currentFile,
            cursorPosition = request.cursor,
            windowSize = LINES_ABOVE_BELOW
        )

        // 2. Open tabs — other files the developer is actively working with
        context.openTabs = summarizeOpenFiles(request.openFiles)

        // 3. Import / dependency context — definitions of referenced symbols
        referencedSymbols = extractReferencedSymbols(context.activeFile)
        context.definitions = resolveDefinitions(referencedSymbols, repoIndex)

        // 4. Semantic search — similar code across the repository
        IF request.hasNaturalLanguageQuery() THEN
            context.semanticResults = embeddingIndex.search(
                query = request.query,
                topK = 10,
                filter = languageFilter(request.language)
            )
        END IF

        // 5. Repository-level context — project structure, conventions
        context.repoStructure = repoIndex.getProjectStructure()
        context.conventions = repoIndex.getConventions()     // From config files, READMEs

        // 6. Trim to token budget
        context = prioritizeAndTruncate(context, maxTokens = TOKEN_BUDGET)

        RETURN context
END CLASS
```

### Repository Indexing

Build and maintain a searchable index of the repository:

```
CLASS RepositoryIndex
    PROPERTIES
        astIndex        : Map<FilePath, AST>
        symbolTable     : Map<SymbolName, List<Definition>>
        dependencyGraph : Graph<FilePath, DependencyEdge>
        chunkIndex      : VectorIndex

    FUNCTION buildIndex(repoPath : Path) -> Void
        files = discoverSourceFiles(repoPath)

        FOR EACH file IN files
            // Parse ASTs for symbol extraction
            ast = parseAST(file)
            astIndex[file.path] = ast

            // Extract symbols — functions, classes, types, variables
            symbols = extractSymbols(ast)
            FOR EACH symbol IN symbols
                symbolTable[symbol.name].ADD(Definition(
                    filePath = file.path,
                    range = symbol.range,
                    kind = symbol.kind,        // FUNCTION, CLASS, TYPE, etc.
                    signature = symbol.signature
                ))
            END FOR

            // Build dependency graph — imports, requires
            imports = extractImports(ast)
            FOR EACH imp IN imports
                dependencyGraph.addEdge(file.path, resolveImport(imp))
            END FOR

            // Chunk and embed for semantic search
            chunks = chunkBySymbol(ast)         // Each function/class is a chunk
            embeddings = embedBatch(chunks)
            chunkIndex.addBatch(chunks, embeddings)
        END FOR

    FUNCTION incrementalUpdate(changedFiles : List<FilePath>) -> Void
        FOR EACH file IN changedFiles
            removeFromIndex(file)
            IF file.exists() THEN
                indexFile(file)
            END IF
        END FOR
END CLASS
```

### Prompt Assembly

Construct the prompt with relevant context, respecting token limits:

```
CLASS PromptAssembler
    PROPERTIES
        tokenBudget : Int
        templateStore : Map<TaskType, PromptTemplate>

    FUNCTION assemble(task : CodeGenTask,
                      context : CodeContext) -> Prompt
        template = templateStore[task.type]     // COMPLETION, EDIT, GENERATE, REVIEW, TEST

        sections = ORDERED LIST OF
            Section(priority = 1, content = template.systemMessage)
            Section(priority = 2, content = formatActiveFile(context.activeFile))
            Section(priority = 3, content = formatDefinitions(context.definitions))
            Section(priority = 4, content = formatSemanticResults(context.semanticResults))
            Section(priority = 5, content = formatOpenTabs(context.openTabs))
            Section(priority = 6, content = formatRepoStructure(context.repoStructure))
            Section(priority = 7, content = formatConventions(context.conventions))

        // Fill prompt up to token budget, highest priority first
        prompt = Prompt()
        remainingTokens = tokenBudget

        FOR EACH section IN sections.sortByPriority()
            tokenCount = countTokens(section.content)
            IF tokenCount <= remainingTokens THEN
                prompt.addSection(section)
                remainingTokens -= tokenCount
            ELSE
                // Truncate section to fit remaining budget
                truncated = truncateToTokens(section.content, remainingTokens)
                prompt.addSection(Section(section.priority, truncated))
                BREAK
            END IF
        END FOR

        // Add the user's instruction or code prefix/suffix for infill
        prompt.addUserMessage(task.instruction)

        RETURN prompt
END CLASS

// Infill (Fill-in-the-Middle) for code completion
CLASS InfillFormatter
    FUNCTION format(prefix : String, suffix : String) -> Prompt
        // FIM format: <prefix>code before cursor<suffix>code after cursor<middle>
        RETURN Prompt(
            prefix = PREFIX_TOKEN + prefix,
            suffix = SUFFIX_TOKEN + suffix,
            trigger = MIDDLE_TOKEN
        )
END CLASS
```

### Code Completion Pipeline

The real-time completion flow triggered as the developer types:

````
CLASS CodeCompletionPipeline
    PROPERTIES
        contextEngine   : ContextEngine
        promptAssembler : PromptAssembler
        model           : InferenceEndpoint
        postProcessor   : CompletionPostProcessor
        debounceMs      : Int = 300

    FUNCTION onKeystroke(event : EditorEvent) -> List<Completion>
        // 1. Debounce — wait for typing pause
        IF timeSinceLastKeystroke() < debounceMs THEN
            RETURN EMPTY
        END IF

        // 2. Check if completion is appropriate
        IF NOT shouldTrigger(event) THEN
            RETURN EMPTY
        END IF

        // 3. Gather context
        request = CodeGenRequest(
            currentFile = event.file,
            cursor = event.cursorPosition,
            openFiles = editor.getOpenFiles()
        )
        context = contextEngine.gatherContext(request)

        // 4. Build prompt (FIM format for completions)
        prefix = context.activeFile.textBefore(event.cursorPosition)
        suffix = context.activeFile.textAfter(event.cursorPosition)
        prompt = promptAssembler.assembleInfill(prefix, suffix, context)

        // 5. Generate — stream tokens with early stopping
        rawCompletion = model.generate(
            prompt = prompt,
            maxTokens = MAX_COMPLETION_TOKENS,
            stopSequences = ["\n\n", "```", endOfBlock(event.language)],
            temperature = 0.0     // Deterministic for completions
        )

        // 6. Post-process — parse, dedent, filter
        completions = postProcessor.process(rawCompletion, context)

        RETURN completions

    FUNCTION shouldTrigger(event : EditorEvent) -> Boolean
        // Don't trigger in comments, strings, or after certain characters
        tokenAtCursor = event.tokenAtCursor()
        IF tokenAtCursor.type IN [COMMENT, STRING_LITERAL] THEN
            RETURN FALSE
        END IF
        RETURN TRUE
END CLASS
````

### Agentic Code Generation

Autonomous multi-step code generation with planning, implementation, and verification:

```
CLASS CodeAgent
    PROPERTIES
        contextEngine  : ContextEngine
        llm            : LanguageModel
        toolSet        : AgentToolSet
        maxIterations  : Int = 25

    FUNCTION execute(task : String) -> AgentResult
        // 1. Plan — break down the task
        plan = llm.generate(planningPrompt(task, contextEngine.getRepoOverview()))
        steps = parsePlan(plan)

        // 2. Execute each step with tool use
        FOR EACH step IN steps
            iteration = 0
            WHILE NOT step.isComplete AND iteration < maxIterations
                // Gather fresh context for this step
                context = contextEngine.gatherContext(step.toRequest())

                // Generate action — which tool to call
                action = llm.generate(actionPrompt(step, context, step.history))
                toolCall = parseToolCall(action)

                // Execute tool
                result = toolSet.execute(toolCall)
                step.history.ADD(ToolResult(toolCall, result))

                // Check if step is complete
                step.isComplete = evaluateCompletion(step, result)
                iteration += 1
            END WHILE
        END FOR

        // 3. Verify — run tests, check types
        verification = verify(steps)
        IF verification.hasFailures THEN
            // Self-heal — attempt to fix failures
            fixResult = selfHeal(verification.failures)
        END IF

        RETURN AgentResult(steps, verification)
END CLASS

CLASS AgentToolSet
    TOOLS
        readFile(path)                           // Read file contents
        writeFile(path, content)                 // Create or overwrite a file
        editFile(path, oldText, newText)          // Targeted edit
        searchCode(query)                        // Semantic search across repo
        searchFiles(pattern)                     // Glob-based file search
        runCommand(command)                      // Execute shell command (sandboxed)
        listDirectory(path)                      // List directory contents
        getErrors(path)                          // Get lint/type errors

    FUNCTION execute(toolCall : ToolCall) -> ToolResult
        // Validate safety — block destructive commands
        IF toolCall.tool == "runCommand" THEN
            IF isDestructive(toolCall.args.command) THEN
                RETURN ToolResult(error = "Blocked: destructive command requires approval")
            END IF
        END IF

        RETURN invokeToolSafely(toolCall)
END CLASS
```

### Code Review Generation

Automated code review using repository conventions and best practices:

```
CLASS CodeReviewGenerator
    PROPERTIES
        contextEngine : ContextEngine
        llm           : LanguageModel

    FUNCTION reviewDiff(diff : Diff) -> CodeReview
        // 1. Gather context for changed files
        changedFiles = diff.getChangedFilePaths()
        context = CodeContext()

        FOR EACH filePath IN changedFiles
            // Get full file with changes applied
            context.addFile(filePath, readFile(filePath))

            // Get definitions referenced by changed code
            symbols = extractReferencedSymbols(diff.getChanges(filePath))
            context.definitions.ADD(resolveDefinitions(symbols))
        END FOR

        // 2. Add project conventions and review guidelines
        context.conventions = contextEngine.getConventions()
        context.reviewGuidelines = loadReviewGuidelines()

        // 3. Generate review
        prompt = buildReviewPrompt(diff, context)
        review = llm.generate(prompt)

        // 4. Parse into structured review comments
        comments = parseReviewComments(review)

        // 5. Filter low-confidence and duplicate comments
        comments = filterComments(comments, confidenceThreshold = 0.7)

        RETURN CodeReview(comments)
END CLASS
```

### Test Generation

Generate test cases grounded in the implementation:

```
CLASS TestGenerator
    PROPERTIES
        contextEngine : ContextEngine
        llm           : LanguageModel
        testRunner    : TestRunner

    FUNCTION generateTests(
        targetFile : FilePath,
        targetFunction : String = NONE
    ) -> TestSuite
        // 1. Gather implementation context
        implementation = readFile(targetFile)
        IF targetFunction != NONE THEN
            implementation = extractFunction(implementation, targetFunction)
        END IF

        // 2. Find existing tests for this file
        existingTests = findExistingTests(targetFile)
        testingPatterns = extractTestingPatterns(existingTests)

        // 3. Get dependencies and mocking targets
        dependencies = contextEngine.getDependencies(targetFile)
        mockTargets = identifyMockTargets(dependencies)

        // 4. Generate tests
        prompt = buildTestPrompt(
            implementation = implementation,
            existingTests = existingTests,
            testingPatterns = testingPatterns,
            mockTargets = mockTargets,
            framework = detectTestFramework()
        )
        generatedTests = llm.generate(prompt)

        // 5. Validate — parse and run generated tests
        testFile = parseTestCode(generatedTests)
        runResult = testRunner.run(testFile)

        IF runResult.hasFailures THEN
            // Iterate — fix failing tests
            fixedTests = fixFailingTests(testFile, runResult.failures)
            testFile = fixedTests
        END IF

        RETURN testFile
END CLASS
```

## Project Structure

```
src/
├── context/                        # Context Engine
│   ├── repository_index/           # AST parsing, symbol extraction
│   ├── semantic_search/            # Embedding-based code search
│   ├── dependency_graph/           # Import and dependency resolution
│   ├── file_context/               # Active file and cursor context
│   └── conventions/                # Project convention detection
│
├── generation/                     # Generation Pipeline
│   ├── prompt_assembly/            # Prompt construction and templating
│   ├── completion/                 # Real-time code completion (FIM)
│   ├── chat/                       # Conversational code generation
│   ├── edit/                       # Targeted code edits
│   └── post_processing/           # Output parsing, deindenting, filtering
│
├── agents/                         # Agentic Code Generation
│   ├── planner/                    # Task decomposition
│   ├── tools/                      # Tool definitions (read, write, search, run)
│   ├── executor/                   # Tool execution and sandboxing
│   └── self_heal/                  # Error recovery and retry logic
│
├── review/                         # Code Review Generation
│   ├── diff_analysis/
│   ├── comment_generation/
│   └── guidelines/
│
├── testing/                        # Test Generation
│   ├── test_generator/
│   ├── mock_detection/
│   └── test_runner/
│
├── interfaces/                     # Developer Interfaces
│   ├── ide_extension/              # Editor plugin (VS Code, JetBrains)
│   ├── cli/                        # Terminal agent
│   └── api/                        # REST / gRPC service endpoint
│
├── config/
│
└── tests/
    ├── unit/
    ├── integration/
    └── eval/                       # Evaluation benchmarks (HumanEval, SWE-bench)
```

## Key Design Considerations

### Context Window Management

Token budgets are finite — optimize what goes into the prompt:

- **Priority-based filling** — Active file and referenced definitions are highest priority; repository structure is lowest
- **Chunking by symbol** — Index at function/class granularity, not arbitrary line counts
- **Deduplication** — Avoid sending the same code through multiple context sources
- **Recency weighting** — Recently edited files and symbols are more likely to be relevant

### Latency and Streaming

Code completion has strict latency requirements (<300ms perceived):

- **Speculative decoding** — Use a smaller draft model to generate candidate tokens verified by a larger model
- **Prompt caching** — Cache the KV state for common prompt prefixes to avoid recomputation
- **Early stopping** — Stop generation at natural boundaries (end of statement, block, or function)
- **Streaming** — Display tokens as they are generated rather than waiting for the full response
- **Request cancellation** — Cancel in-flight requests when the developer continues typing

### Evaluation

Measure code generation quality rigorously:

- **Benchmark suites** — HumanEval, MBPP, SWE-bench, MultiPL-E for cross-language evaluation
- **Acceptance rate** — Track what percentage of suggestions developers accept
- **Persistence rate** — Track how much accepted code persists after further editing
- **Functional correctness** — Run generated code against test suites
- **Edit distance** — Measure how much developers modify accepted suggestions

### Security and Safety

- **Sandbox execution** — Run generated code and commands in isolated environments
- **Secret detection** — Scan generated code for hardcoded credentials, API keys, or tokens
- **Dependency safety** — Validate that suggested imports reference real, non-malicious packages
- **Permission boundaries** — Destructive operations (file deletion, git push) require explicit approval
- **License compliance** — Filter or flag generated code that matches copyrighted training data

## Benefits

1. **Developer Productivity** — Reduces boilerplate and accelerates routine coding tasks
2. **Repository-Aware Suggestions** — Generated code follows project conventions and reuses existing patterns
3. **Reduced Context Switching** — Developers stay in their editor rather than searching documentation
4. **Automated Review and Testing** — Catches issues earlier and generates test coverage automatically
5. **Accessible to All Skill Levels** — Helps junior developers learn patterns and helps senior developers move faster

## Trade-offs

| Advantage                            | Consideration                                                |
| ------------------------------------ | ------------------------------------------------------------ |
| Accelerates common coding patterns   | May reduce deep understanding of underlying code             |
| Repository-aware context grounding   | Indexing and embedding storage add infrastructure overhead   |
| Real-time inline completions         | Strict latency requirements constrain model size and context |
| Agentic multi-step generation        | Autonomous agents need safety boundaries and human oversight |
| Automated test and review generation | Generated tests may have low coverage or miss edge cases     |
| Works across many languages          | Quality varies by language based on training data coverage   |

## When to Use

✅ **Good fit for:**

- Accelerating day-to-day development tasks (boilerplate, tests, documentation)
- Teams that want consistent code style and pattern adherence across a large codebase
- Automating code review for common issues (style, bugs, security vulnerabilities)
- Generating test scaffolding and increasing test coverage
- Enabling developers to work with unfamiliar codebases or languages

❌ **Not ideal for:**

- Safety-critical systems where every line of code must be manually verified and formally proven
- Highly novel algorithm development where no training data patterns exist
- Environments without access to LLM inference (air-gapped, severely constrained compute)
- Codebases with extremely proprietary domain languages unsupported by available models

## References

- [GitHub Copilot — AI Pair Programmer](https://github.com/features/copilot)
- [Codex: Evaluating Large Language Models Trained on Code — Chen et al. (2021)](https://arxiv.org/abs/2107.03374)
- [SWE-bench: Can Language Models Resolve Real-World GitHub Issues? — Jimenez et al. (2024)](https://arxiv.org/abs/2310.06770)
- [Repository-Level Prompt Generation for LLMs — Shrivastava et al. (2023)](https://arxiv.org/abs/2206.12839)
- [StarCoder: May the Source Be with You — Li et al. (2023)](https://arxiv.org/abs/2305.06161)
- [Efficient Training of Language Models to Fill in the Middle — Bavarian et al. (2022)](https://arxiv.org/abs/2207.14255)
- [SWE-agent: Agent-Computer Interfaces for Software Engineering — Yang et al. (2024)](https://arxiv.org/abs/2405.15793)
