# AI Agent Architecture

## Overview

**AI Agent Architecture** describes the structural patterns for building systems where large language models (LLMs) dynamically reason, plan, and act by selecting tools, retrieving knowledge, and orchestrating multi-step workflows. Unlike traditional request-response applications, agent systems operate in loops — observing the environment, deciding on actions, executing them, and evaluating results until a goal is achieved.

The field draws on foundational work from multiple sources:

- **Anthropic** — "The most successful implementations use simple, composable patterns rather than complex frameworks" (Building Effective Agents, 2024)
- **Microsoft** — AI agent orchestration patterns for multi-agent coordination (Azure Architecture Center, 2025)
- **Andrew Ng** — Four agentic design patterns: Reflection, Tool Use, Planning, Multi-Agent Collaboration

Key principles:

- **Start simple** — Use the lowest level of complexity that reliably meets requirements. A single LLM call with a well-crafted prompt may suffice before introducing agent logic.
- **Augmented LLM as building block** — An LLM enhanced with retrieval, tools, and memory is the foundational unit.
- **Transparency** — Explicitly surface the agent's planning steps and tool interactions for observability and trust.
- **Composability** — Combine simple workflow patterns (chaining, routing, parallelization) rather than building monolithic agents.

## Core Concepts

### Complexity Spectrum

Before adopting a complex agent pattern, evaluate where your scenario falls:

```
┌───────────────────────────────────────────────────────────────────────┐
│                     Agent Complexity Spectrum                         │
├──────────────────┬────────────────────────────────────────────────────┤
│  Level           │  Description                                      │
├──────────────────┼────────────────────────────────────────────────────┤
│  Direct LLM Call │  Single model call with a well-crafted prompt.    │
│                  │  No agent logic, no tool access.                  │
├──────────────────┼────────────────────────────────────────────────────┤
│  Single Agent    │  One agent that reasons and acts by selecting     │
│  + Tools         │  from available tools, knowledge, and APIs.       │
│                  │  Can loop through multiple model calls.           │
├──────────────────┼────────────────────────────────────────────────────┤
│  Multi-Agent     │  Multiple specialized agents coordinate to        │
│  Orchestration   │  solve a problem. An orchestrator manages work    │
│                  │  distribution, context sharing, and aggregation.  │
└──────────────────┴────────────────────────────────────────────────────┘

        Increasing complexity →
        Increasing coordination overhead, latency, and cost →
```

> **Rule of thumb:** If prompt engineering can solve the problem, you do not need an agent. If a single agent with tools can handle the task, you do not need multi-agent orchestration.

### Core Agent Loop

Every agent — from simple to complex — follows the same fundamental loop:

```
┌──────────────────────────────────────────────────────┐
│                   Agent Loop                          │
│                                                      │
│   ┌──────────┐                                       │
│   │  Observe  │◄──────────────────────────┐          │
│   └─────┬────┘                            │          │
│         │                                 │          │
│         ▼                                 │          │
│   ┌──────────┐                            │          │
│   │  Reason   │  (LLM call with context)  │          │
│   └─────┬────┘                            │          │
│         │                                 │          │
│         ▼                                 │          │
│   ┌──────────┐      ┌─────────────┐      │          │
│   │  Decide   │─────▶│  Use Tools   │─────┘          │
│   └─────┬────┘      └─────────────┘                  │
│         │                                            │
│         ▼                                            │
│   ┌──────────────┐                                   │
│   │  Goal Met?    │── Yes ──▶  Return Result         │
│   │  Max Iter?    │── Yes ──▶  Stop / Escalate       │
│   └──────────────┘                                   │
└──────────────────────────────────────────────────────┘
```

### Workflow Patterns

#### 1. Prompt Chaining (Sequential)

Decompose a task into a sequence of steps where each LLM call processes the output of the previous one. Programmatic checks (gates) can validate intermediate steps.

```
Input ──▶ [LLM Step 1] ──▶ Gate ──▶ [LLM Step 2] ──▶ Gate ──▶ [LLM Step 3] ──▶ Result
```

**When to use:**

- Tasks that decompose cleanly into fixed subtasks
- Trading latency for higher accuracy by making each call simpler
- Progressive refinement (draft → review → polish)

#### 2. Routing

Classify an input and direct it to a specialized follow-up task, allowing separation of concerns and more focused prompts:

```
                        ┌──▶ [Specialist A] ──▶ Result
                        │
Input ──▶ [Classifier] ──┼──▶ [Specialist B] ──▶ Result
                        │
                        └──▶ [Specialist C] ──▶ Result
```

**When to use:**

- Complex tasks with distinct categories handled separately
- Routing simple queries to smaller cost-efficient models, complex queries to larger models
- Customer service triage (billing, technical, general)

#### 3. Parallelization (Fan-Out / Fan-In)

Run multiple LLM calls simultaneously and aggregate results. Two key variations:

- **Sectioning** — Break a task into independent subtasks run in parallel
- **Voting** — Run the same task multiple times for diverse outputs or higher confidence

```
                ┌──▶ [Agent A] ──▶ Result A ──┐
                │                              │
Input ──▶ Fan-Out ──▶ [Agent B] ──▶ Result B ──┼──▶ Aggregator ──▶ Final Result
                │                              │
                └──▶ [Agent C] ──▶ Result C ──┘
```

**When to use:**

- Independent analysis from multiple perspectives
- Guardrails (one instance handles queries, another screens for safety)
- Code review where multiple prompts look for different vulnerability types

#### 4. Orchestrator-Workers

A central LLM dynamically breaks down tasks, delegates to worker LLMs, and synthesizes results. Unlike parallelization, subtasks are **not** predefined:

```
                                 ┌──▶ [Worker 1] ──┐
                                 │                  │
Input ──▶ [Orchestrator LLM] ────┼──▶ [Worker 2] ──┼──▶ Synthesized Result
                                 │                  │
                                 └──▶ [Worker N] ──┘
```

**When to use:**

- Complex tasks where subtasks cannot be predicted upfront
- Coding agents that modify multiple files based on a description
- Research tasks gathering information from multiple sources

#### 5. Evaluator-Optimizer (Maker-Checker)

One LLM generates a response while another evaluates and provides feedback in a loop until quality criteria are met:

```
Input ──▶ [Generator] ──▶ Output ──▶ [Evaluator] ──┐
               ▲                                     │
               │            Feedback                  │
               └──────────────────────────────────────┘
                                                      │
                                              Approved ──▶ Result
```

**When to use:**

- Clear evaluation criteria exist
- Iterative refinement provides measurable improvements
- Literary translation, complex searches, content polishing

### Multi-Agent Orchestration Patterns

When a single agent with tools is insufficient, multi-agent patterns address the coordination challenge:

#### Sequential Orchestration

Agents chained in a predefined linear order — each processes the previous agent's output:

```
Input ──▶ [Agent 1] ──▶ [Agent 2] ──▶ ... ──▶ [Agent N] ──▶ Result
               │              │                     │
          Model/Tools    Model/Tools            Model/Tools

                    ←── Common State ──→
```

**When to use:**

- Multistage processes with clear linear dependencies
- Data transformation pipelines where each stage adds specific value
- Draft → review → polish refinement workflows

#### Concurrent Orchestration

Multiple agents work simultaneously on the same input and results are aggregated:

```
                    ┌──▶ [Agent 1] ──▶ Intermediate Result ──┐
                    │                                         │
Input ──▶ [Initiator] ──▶ [Agent 2] ──▶ Intermediate Result ──┼──▶ Aggregated Result
                    │                                         │
                    └──▶ [Agent N] ──▶ Intermediate Result ──┘
```

**When to use:**

- Tasks benefiting from multiple independent perspectives
- Ensemble reasoning, brainstorming, voting-based decisions
- Time-sensitive scenarios where parallel processing reduces latency

#### Handoff Orchestration

Dynamic delegation between specialized agents — each agent assesses the task and decides whether to handle it or transfer to a more appropriate agent:

```
Input ──▶ [Triage Agent] ──handoff──▶ [Technical Agent] ──handoff──▶ [Billing Agent]
               │                             │                            │
          Model/Knowledge              Model/Tools                  Model/APIs
               │                             │                            │
               ▼                             ▼                            ▼
           Result (if resolved)         Result (if resolved)         Result
```

**When to use:**

- Optimal agent is not known upfront
- Task requirements become clear only during processing
- Customer support triage and escalation

#### Group Chat Orchestration

Multiple agents participate in a shared conversation thread, coordinated by a chat manager:

```
Input ──▶ [Chat Manager] ──▶ Accumulating Thread ──▶ Consensus / Result
              │
              ├──▶ [Agent 1]   (environmental expert)
              ├──▶ [Agent 2]   (budget analyst)
              ├──▶ [Agent 3]   (community advocate)
              └──▶ [Human]     (optional human-in-the-loop)
```

**When to use:**

- Collaborative brainstorming and debate
- Decision processes requiring consensus
- Maker-checker quality validation loops
- Multidisciplinary problems requiring cross-functional dialogue

#### Magentic Orchestration

A manager agent builds and adapts a task ledger dynamically. Specialized agents execute tasks and report back. The plan evolves as the problem is understood:

```
Input ──▶ [Manager Agent] ──▶ Task Ledger ──▶ Goal Met? ──▶ Result
              │                    │               │
              │    ┌───────────────┘               │ No
              │    │                               │
              ├──▶ [Diagnostics Agent]    ◄────────┘
              ├──▶ [Infrastructure Agent]
              ├──▶ [Rollback Agent]
              └──▶ [Communication Agent]
```

**When to use:**

- Open-ended problems with no predetermined solution path
- Agents equipped with tools that interact with external systems
- Incident response, autonomous remediation
- A documented plan is needed before execution proceeds

## Implementation

### Augmented LLM (Building Block)

```
// The foundational building block — an LLM with access to tools and knowledge
CLASS AugmentedLLM
    PROPERTIES
        model          : LanguageModel
        tools          : List<ToolDefinition>
        knowledgeStore : KnowledgeRetriever
        memory         : ConversationMemory

    FUNCTION invoke(prompt : String, context : Map) -> LLMResponse
        // Build context from memory and retrieved knowledge
        relevantDocs = knowledgeStore.retrieve(prompt)
        history      = memory.getRecentMessages(limit = 10)

        fullPrompt = buildPrompt(
            systemInstructions = context.GET("system_prompt"),
            conversationHistory = history,
            retrievedContext    = relevantDocs,
            userMessage         = prompt,
            availableTools      = tools
        )

        response = model.generate(fullPrompt)

        memory.append(role = "user", content = prompt)
        memory.append(role = "assistant", content = response)

        RETURN response
END CLASS
```

### Tool-Using Agent

```
// Agent that reasons and acts in a loop using tools
CLASS ToolUsingAgent
    PROPERTIES
        llm            : AugmentedLLM
        tools          : Map<String, Tool>
        maxIterations  : Integer = 10
        systemPrompt   : String

    FUNCTION run(userQuery : String) -> AgentResult
        messages = [NEW Message(role = "user", content = userQuery)]
        iteration = 0

        WHILE iteration < maxIterations
            response = llm.invoke(messages, systemPrompt, tools)

            IF response.hasToolCalls() THEN
                FOR EACH toolCall IN response.toolCalls
                    tool = tools.GET(toolCall.name)
                    IF tool IS NULL THEN
                        THROW UnknownToolError(toolCall.name)
                    END IF

                    result = tool.execute(toolCall.arguments)

                    messages.ADD(NEW Message(
                        role    = "tool_result",
                        toolId  = toolCall.id,
                        content = result
                    ))
                END FOR
            ELSE
                // No tool calls — agent has produced a final answer
                RETURN NEW AgentResult(
                    answer     = response.content,
                    iterations = iteration + 1,
                    toolsUsed  = extractToolNames(messages)
                )
            END IF

            iteration = iteration + 1
        END WHILE

        RETURN NEW AgentResult(
            answer  = "Max iterations reached",
            status  = "INCOMPLETE"
        )
END CLASS
```

### Tool Definition

```
// Tool — An action the agent can invoke
INTERFACE Tool
    FUNCTION name() -> String
    FUNCTION description() -> String
    FUNCTION parameters() -> ParameterSchema
    FUNCTION execute(arguments : Map) -> ToolResult
END INTERFACE

// Example: Database Query Tool
CLASS DatabaseQueryTool IMPLEMENTS Tool
    CONSTRUCTOR(dbConnection : DatabaseConnection)

    FUNCTION name() -> String
        RETURN "query_database"

    FUNCTION description() -> String
        RETURN "Execute a read-only SQL query against the application database. " +
               "Use this when the user asks for data that requires database lookup."

    FUNCTION parameters() -> ParameterSchema
        RETURN SCHEMA {
            query : String (required) — "The SQL SELECT query to execute"
        }

    FUNCTION execute(arguments : Map) -> ToolResult
        query = arguments.GET("query")
        IF NOT query.STARTS_WITH("SELECT") THEN
            RETURN ToolResult(error = "Only SELECT queries are permitted")
        END IF
        rows = dbConnection.execute(query)
        RETURN ToolResult(data = rows)
END CLASS
```

### Multi-Agent Orchestrator

```
// Orchestrator that coordinates multiple specialized agents
CLASS MultiAgentOrchestrator
    PROPERTIES
        agents    : Map<String, ToolUsingAgent>
        router    : RoutingAgent
        strategy  : OrchestrationStrategy   // SEQUENTIAL, CONCURRENT, HANDOFF

    FUNCTION run(userQuery : String) -> OrchestratedResult
        MATCH strategy

            CASE SEQUENTIAL:
                context = userQuery
                FOR EACH agentName IN router.determineSequence(userQuery)
                    agent  = agents.GET(agentName)
                    result = agent.run(context)
                    context = result.answer  // Output becomes next input
                END FOR
                RETURN NEW OrchestratedResult(finalAnswer = context)

            CASE CONCURRENT:
                selectedAgents = router.selectAgents(userQuery)
                results = PARALLEL FOR EACH agentName IN selectedAgents
                    agents.GET(agentName).run(userQuery)
                END PARALLEL
                RETURN aggregateResults(results)

            CASE HANDOFF:
                currentAgent = router.selectInitialAgent(userQuery)
                WHILE currentAgent IS NOT NULL
                    result = currentAgent.run(userQuery)
                    IF result.handoffTo IS NOT NULL THEN
                        currentAgent = agents.GET(result.handoffTo)
                    ELSE
                        RETURN NEW OrchestratedResult(finalAnswer = result.answer)
                    END IF
                END WHILE

        END MATCH
END CLASS
```

### Memory and Context Management

```
// Conversation memory with compaction for long-running agents
CLASS ConversationMemory
    PROPERTIES
        messages     : List<Message>
        maxTokens    : Integer
        summarizer   : LanguageModel

    FUNCTION append(role : String, content : String) -> Void
        messages.ADD(NEW Message(role = role, content = content))

        IF estimateTokens(messages) > maxTokens THEN
            compactHistory()
        END IF

    FUNCTION compactHistory() -> Void
        // Summarize older messages to stay within context window
        oldMessages = messages[0 .. messages.SIZE() - 5]
        summary = summarizer.generate(
            "Summarize this conversation history concisely: " +
            formatMessages(oldMessages)
        )
        recentMessages = messages[messages.SIZE() - 5 .. ]
        messages = [NEW Message(role = "system", content = summary)]
        messages.ADD_ALL(recentMessages)

    FUNCTION getRecentMessages(limit : Integer) -> List<Message>
        RETURN messages[MAX(0, messages.SIZE() - limit) .. ]
END CLASS
```

## Project Structure

```
src/
├── agents/                         # Agent Definitions
│   ├── base/
│   │   ├── agent/                  # Base agent loop
│   │   ├── augmented_llm/          # LLM + tools + retrieval
│   │   └── memory/                 # Conversation memory
│   ├── specialized/
│   │   ├── research_agent/
│   │   ├── coding_agent/
│   │   └── customer_support_agent/
│   └── orchestrators/
│       ├── sequential/
│       ├── concurrent/
│       ├── handoff/
│       └── group_chat/
│
├── tools/                          # Tool Definitions
│   ├── database/
│   ├── search/
│   ├── file_system/
│   ├── api_clients/
│   └── registry/                   # Tool discovery and registration
│
├── knowledge/                      # Knowledge / Retrieval Layer
│   ├── vector_store/
│   ├── document_loader/
│   └── chunking/
│
├── guardrails/                     # Safety and Content Filtering
│   ├── input_filters/
│   ├── output_filters/
│   └── tool_call_validators/
│
├── observability/                  # Monitoring and Tracing
│   ├── tracing/
│   ├── metrics/
│   └── logging/
│
├── config/                         # Configuration and Wiring
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/                 # LLM-as-judge evaluations
```

## Key Design Considerations

### Guardrails and Safety

Apply content safety at **multiple points** in the orchestration:

1. **Input validation** — Filter user inputs before reaching the agent
2. **Tool call validation** — Validate tool arguments before execution
3. **Tool response filtering** — Sanitize tool outputs before passing to the LLM
4. **Output filtering** — Screen final responses before returning to the user

```
CLASS GuardedAgent
    CONSTRUCTOR(agent : ToolUsingAgent,
                inputFilter : ContentFilter,
                outputFilter : ContentFilter,
                toolValidator : ToolCallValidator)

    FUNCTION run(userQuery : String) -> AgentResult
        // 1. Filter input
        IF NOT inputFilter.isAllowed(userQuery) THEN
            RETURN AgentResult(answer = "I cannot help with that request.")
        END IF

        // 2. Run agent with tool validation
        result = agent.runWithValidation(userQuery, toolValidator)

        // 3. Filter output
        IF NOT outputFilter.isAllowed(result.answer) THEN
            RETURN AgentResult(answer = "Response filtered for safety.")
        END IF

        RETURN result
END CLASS
```

### Human-in-the-Loop

Design explicit checkpoints where human input is required or optional:

- **Approval gates** — Pause before executing high-impact tool calls (e.g., database writes, financial transactions)
- **Feedback loops** — Allow humans to provide corrections mid-workflow
- **Escalation paths** — Transfer to a human when confidence is low or the task exceeds agent capabilities

### Observability

Instrument all agent operations and handoffs:

- **Trace every LLM call** — Input tokens, output tokens, latency, model used
- **Log tool invocations** — Tool name, arguments, result, duration
- **Track agent transitions** — Which agent handled what, handoff reasons
- **Monitor token consumption** — Per agent, per orchestration run

### Iteration Limits and Error Handling

- Set **maximum iteration counts** to prevent infinite loops
- Implement **timeout mechanisms** for tool calls
- Use **circuit breakers** for unreliable external dependencies
- Validate agent output before passing to the next agent — low-confidence or malformed responses can cascade

## Benefits

1. **Flexibility** — Agents adapt to novel tasks without predefined workflows
2. **Composability** — Simple patterns combine into complex behaviors
3. **Specialization** — Each agent focuses on a specific domain, reducing prompt complexity
4. **Scalability** — Add or modify agents without redesigning the entire system
5. **Maintainability** — Test and debug individual agents in isolation
6. **Human Oversight** — Transparent planning steps and approval gates enable trust

## Trade-offs

| Advantage                         | Consideration                                         |
| --------------------------------- | ----------------------------------------------------- |
| Dynamic problem-solving           | Higher latency and cost than direct LLM calls         |
| Tool use extends LLM capabilities | Compounding errors across iterations                  |
| Multi-agent specialization        | Coordination overhead and increased failure modes     |
| Transparent reasoning traces      | Context windows grow, requiring compaction strategies |
| Human-in-the-loop support         | More complex testing (nondeterministic outputs)       |

## When to Use

✅ **Good fit for:**

- Tasks requiring dynamic tool use and multi-step reasoning
- Cross-functional problems spanning multiple domains
- Customer support with escalation paths and tool access
- Coding agents that modify files, run tests, and iterate
- Research tasks gathering and synthesizing information from multiple sources
- Workflows requiring human approval at key decision points

❌ **Not ideal for:**

- Single-step tasks solvable with a well-crafted prompt
- Latency-critical applications where every millisecond matters
- Tasks with deterministic, well-defined steps (use traditional workflows)
- Scenarios where LLM cost per interaction must be minimized
- Problems where errors cannot compound safely (high-stakes without human review)

## References

- [Building Effective Agents — Anthropic (2024)](https://www.anthropic.com/engineering/building-effective-agents)
- [AI Agent Orchestration Patterns — Microsoft Azure Architecture Center (2025)](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Baseline Foundry Chat Reference Architecture — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat)
- [Microsoft Agent Framework — Overview](https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview)
- [ReAct: Synergizing Reasoning and Acting in Language Models — Yao et al. (2023)](https://arxiv.org/abs/2210.03629)
- [Agentic Design Patterns — Andrew Ng / DeepLearning.AI](https://www.deeplearning.ai/the-batch/how-agents-can-improve-llm-performance/)
