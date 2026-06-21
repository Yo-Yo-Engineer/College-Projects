# Compound AI Systems Architecture

## Overview

**Compound AI Systems Architecture** defines the structural patterns for building AI applications that compose multiple interacting components — models, retrievers, tools, code modules, and verifiers — rather than relying on a single monolithic model. Coined by researchers at Berkeley AI Research (BAIR), the compound AI systems paradigm recognizes that the highest-performing AI applications in production achieve their results through system design, not just model capability.

While a single LLM call may handle simple tasks, production AI systems routinely combine retrieval engines, multiple specialized models, programmatic validators, code interpreters, and external tools into orchestrated pipelines. The architecture challenge shifts from "which model is best?" to "how do I compose components to maximize quality, cost, and reliability?"

Key principles:

- **System Over Model** — Optimize the composition of components rather than relying on any single model's capability
- **Component Specialization** — Use the right tool for each subtask: a small model for classification, a large model for generation, code for computation, retrieval for knowledge
- **Verifiable Outputs** — Incorporate programmatic validators, type checkers, and verifiers that provide guarantees no model can
- **Cost-Quality Optimization** — Route between components of different cost/capability profiles based on task complexity
- **Composability** — Components communicate through well-defined interfaces, enabling independent development and swapping

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Compound AI System Architecture                           │
│                                                                             │
│   User Input                                                                │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                       Orchestrator                                │      │
│   │   (Controls flow, selects components, manages state)             │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│          ┌───────────────────┼───────────────────┐                          │
│          │                   │                   │                          │
│          ▼                   ▼                   ▼                          │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                    │
│   │  Retriever   │    │  Generator   │    │  Code        │                   │
│   │  Component   │    │  Component   │    │  Executor    │                   │
│   │              │    │              │    │              │                   │
│   │  Vector DB   │    │  LLM (large  │    │  Sandboxed   │                   │
│   │  + Re-ranker │    │  or small)   │    │  runtime     │                   │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                    │
│          │                  │                   │                            │
│          └──────────────────┼───────────────────┘                           │
│                             │                                               │
│                             ▼                                               │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                       Verifier Layer                              │      │
│   │   ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────┐ │      │
│   │   │  Type     │  │  Schema       │  │  LLM-as-   │  │  Domain  │ │      │
│   │   │  Checker  │  │  Validator    │  │  Judge     │  │  Rules   │ │      │
│   │   └──────────┘  └──────────────┘  └────────────┘  └──────────┘ │      │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│                    ┌──────────────────┐                                     │
│                    │  Output Assembly  │                                     │
│                    └──────────────────┘                                     │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Component   │  │  Cost/Quality │  │  Observability     │  │
│                   │ Registry    │  │  Router       │  │  & Tracing         │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Component Types

A compound AI system is built from a vocabulary of component types, each with distinct performance characteristics:

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Component Taxonomy                                │
├──────────────────┬───────────────────────────────────────────────────┤
│  Component       │  Purpose                                          │
├──────────────────┼───────────────────────────────────────────────────┤
│  LLM (Large)     │  Complex reasoning, generation, planning.        │
│                  │  High capability, high cost, high latency.       │
├──────────────────┼───────────────────────────────────────────────────┤
│  LLM (Small)     │  Classification, extraction, simple generation.  │
│                  │  Lower cost, lower latency, narrower capability. │
├──────────────────┼───────────────────────────────────────────────────┤
│  Retriever       │  Fetch relevant documents, data, or examples.    │
│                  │  Grounds outputs in external knowledge.          │
├──────────────────┼───────────────────────────────────────────────────┤
│  Code Executor   │  Run deterministic computations, data transforms,│
│                  │  API calls. Provides exactness models cannot.    │
├──────────────────┼───────────────────────────────────────────────────┤
│  Verifier        │  Validate outputs using rules, schemas, tests,   │
│                  │  or a judge model. Catches errors before output. │
├──────────────────┼───────────────────────────────────────────────────┤
│  Ranker          │  Score and order candidates from other components.│
│                  │  Refines quality of retrieved or generated items. │
├──────────────────┼───────────────────────────────────────────────────┤
│  Classifier      │  Route inputs to the appropriate pipeline        │
│                  │  branch based on content, intent, or complexity. │
├──────────────────┼───────────────────────────────────────────────────┤
│  Cache           │  Store and reuse previous results for identical  │
│                  │  or semantically similar inputs.                 │
└──────────────────┴───────────────────────────────────────────────────┘
```

### Composition Patterns

```
// Pipeline — Sequential component chain
PATTERN Pipeline
    Input → Component A → Component B → Component C → Output

    EXAMPLE: Query → Retriever → Generator → Verifier → Response

// Router — Conditional branching based on input characteristics
PATTERN Router
    Input → Classifier
                ├── Simple    → Small LLM → Output
                ├── Complex   → Large LLM → Output
                └── Knowledge → Retriever → LLM → Output

// Ensemble — Multiple components produce candidates, best is selected
PATTERN Ensemble
    Input ──┬──▶ Model A ──┐
            ├──▶ Model B ──┼──▶ Selector / Ranker → Output
            └──▶ Model C ──┘

// Cascade — Progressive escalation from cheap to expensive
PATTERN Cascade
    Input → Small LLM ─── confident? ─── Yes ──▶ Output
                              │
                              No
                              │
                              ▼
                         Large LLM ─── confident? ─── Yes ──▶ Output
                                           │
                                           No
                                           │
                                           ▼
                                     Expert Model ──▶ Output

// Loop — Iterative refinement with feedback
PATTERN Loop
    Input → Generator → Verifier ─── pass? ─── Yes ──▶ Output
                 ▲                      │
                 │                      No
                 └──── Feedback ────────┘
```

### Model Router

Route requests to the appropriate model based on complexity, cost, and quality requirements:

```
CLASS ModelRouter
    PROPERTIES
        classifier   : ComplexityClassifier
        models       : Map<Tier, ModelEndpoint>
        costBudget   : CostTracker

    FUNCTION route(request : Request) -> ModelEndpoint
        complexity = classifier.classify(request)

        // Check cost budget
        IF costBudget.remainingBudget() < models[HIGH].estimatedCost(request) THEN
            complexity = MIN(complexity, MEDIUM)
        END IF

        MATCH complexity
            CASE LOW    -> RETURN models[LOW]       // Small/fast model
            CASE MEDIUM -> RETURN models[MEDIUM]    // Mid-tier model
            CASE HIGH   -> RETURN models[HIGH]      // Large/capable model
        END MATCH
END CLASS

DATA Tier
    LOW       // GPT-4o-mini, Claude Haiku, Gemini Flash
    MEDIUM    // GPT-4o, Claude Sonnet
    HIGH      // o1, Claude Opus, Gemini Ultra
END DATA
```

### Verifier-Driven Generation

Use programmatic and model-based verifiers to ensure output quality in a generate-then-verify loop:

```
CLASS VerifierPipeline
    PROPERTIES
        generator   : LLMComponent
        verifiers   : List<Verifier>
        maxRetries  : Int = 3

    FUNCTION generate(prompt : String) -> VerifiedOutput
        FOR attempt IN 1..maxRetries
            candidate = generator.generate(prompt)

            allPassed = TRUE
            feedback = EMPTY LIST

            FOR EACH verifier IN verifiers
                result = verifier.verify(candidate)
                IF NOT result.passed THEN
                    allPassed = FALSE
                    feedback.ADD(result.feedback)
                END IF
            END FOR

            IF allPassed THEN
                RETURN VerifiedOutput(content = candidate, attempts = attempt)
            END IF

            // Add feedback to prompt for next attempt
            prompt = appendFeedback(prompt, candidate, feedback)
        END FOR

        RETURN VerifiedOutput(content = candidate, attempts = maxRetries,
                              verified = FALSE)
END CLASS

// Verifier types
INTERFACE Verifier
    FUNCTION verify(output : String) -> VerificationResult
END INTERFACE

CLASS SchemaVerifier IMPLEMENTS Verifier
    // Validates JSON output against a schema
    FUNCTION verify(output : String) -> VerificationResult
        parsed = parseJSON(output)
        errors = validateAgainstSchema(parsed, expectedSchema)
        RETURN VerificationResult(passed = errors.isEmpty(), feedback = errors)
END CLASS

CLASS CodeExecutionVerifier IMPLEMENTS Verifier
    // Runs generated code and checks for errors
    FUNCTION verify(output : String) -> VerificationResult
        code = extractCode(output)
        result = sandbox.execute(code, timeout = 30_SECONDS)
        RETURN VerificationResult(passed = result.exitCode == 0,
                                  feedback = result.stderr)
END CLASS

CLASS LLMJudgeVerifier IMPLEMENTS Verifier
    // Uses a separate LLM to evaluate quality
    FUNCTION verify(output : String) -> VerificationResult
        judgment = judgeLLM.evaluate(output, criteria)
        RETURN VerificationResult(passed = judgment.score >= threshold,
                                  feedback = judgment.explanation)
END CLASS
```

### DSPy-Style Optimization

Instead of manually engineering prompts for each component, optimize the system end-to-end:

```
CLASS SystemOptimizer
    PROPERTIES
        system      : CompoundAISystem
        evalDataset : List<EvalExample>
        metric      : EvalMetric

    FUNCTION optimize() -> OptimizedSystem
        // 1. Evaluate current system performance
        baseline = evaluateSystem(system, evalDataset, metric)

        // 2. Identify bottleneck components
        componentScores = evaluatePerComponent(system, evalDataset)
        bottleneck = componentScores.worstPerforming()

        // 3. Optimize bottleneck (prompt tuning, few-shot selection, model swap)
        candidates = generateOptimizationCandidates(bottleneck)

        bestCandidate = NULL
        bestScore = baseline

        FOR EACH candidate IN candidates
            system.replaceComponent(bottleneck.id, candidate)
            score = evaluateSystem(system, evalDataset, metric)
            IF score > bestScore THEN
                bestScore = score
                bestCandidate = candidate
            END IF
        END FOR

        IF bestCandidate != NULL THEN
            system.replaceComponent(bottleneck.id, bestCandidate)
        END IF

        RETURN system
END CLASS
```

## Project Structure

```
src/
├── components/                     # Individual AI Components
│   ├── generators/
│   │   ├── llm_generator/
│   │   └── code_generator/
│   ├── retrievers/
│   │   ├── vector_retriever/
│   │   └── graph_retriever/
│   ├── verifiers/
│   │   ├── schema_verifier/
│   │   ├── code_verifier/
│   │   └── llm_judge/
│   ├── classifiers/
│   │   ├── intent_classifier/
│   │   └── complexity_classifier/
│   └── rankers/
│       └── cross_encoder_ranker/
│
├── composition/                    # System Composition Layer
│   ├── orchestrator/
│   ├── router/
│   ├── pipeline/
│   ├── ensemble/
│   └── cascade/
│
├── optimization/                   # System-Level Optimization
│   ├── prompt_optimizer/
│   ├── component_evaluator/
│   └── cost_optimizer/
│
├── caching/                        # Result Caching
│   ├── exact_cache/
│   └── semantic_cache/
│
├── observability/                  # System Tracing
│   ├── tracing/
│   ├── metrics/
│   └── cost_tracking/
│
├── config/                         # Configuration
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/                 # End-to-end system evaluations
```

## Key Design Considerations

### Interface Contracts Between Components

Each component should have a well-defined input/output contract:

- **Typed interfaces** — Define explicit data structures for component inputs and outputs
- **Serializable state** — Component outputs must be serializable for logging, caching, and debugging
- **Error propagation** — Components should return structured errors, not raw exceptions, to enable upstream retry or fallback logic
- **Metadata passthrough** — Include provenance metadata (which component produced what, latency, token count) for observability

### Cost Management

Compound systems can become expensive when multiple components run per request:

- **Cost budgets** — Set per-request cost ceilings and route to cheaper components when approaching limits
- **Caching** — Cache component outputs at multiple levels (exact match, semantic similarity)
- **Short-circuiting** — Skip downstream components when upstream results are sufficient
- **Batch processing** — Aggregate multiple requests to amortize overhead of expensive components

### Testing Compound Systems

Traditional unit tests are insufficient for nondeterministic, multi-component systems:

- **Component isolation** — Test each component independently with mocked inputs
- **Integration tests** — Test component pairs to verify interface contracts
- **System evaluations** — Run evaluation datasets through the full system and measure end-to-end metrics
- **Regression suites** — Maintain golden datasets to detect quality degradation after changes
- **A/B testing** — Compare system variants in production on live traffic

### Failure Modes

Compound systems have unique failure patterns:

- **Error cascading** — An upstream component error propagates and amplifies through downstream components
- **Latency amplification** — Sequential components add latency; parallel components are bounded by the slowest
- **Cost explosion** — Retry loops or ensembles can multiply cost unexpectedly
- **Silent degradation** — A component may return plausible but incorrect results that downstream components cannot catch

## Benefits

1. **Higher Quality** — Combining retrieval, generation, and verification exceeds what any single model achieves alone
2. **Cost Efficiency** — Route simple requests to cheap models, reserve expensive models for complex tasks
3. **Reliability** — Programmatic verifiers provide guarantees that probabilistic models cannot
4. **Flexibility** — Swap individual components (models, retrievers, verifiers) without redesigning the system
5. **Optimizable** — Tune the system composition, not just individual prompts
6. **Transparency** — Trace decisions through each component for debugging and audit

## Trade-offs

| Advantage                          | Consideration                                         |
| ---------------------------------- | ----------------------------------------------------- |
| Higher quality through composition | Increased system complexity and debugging difficulty  |
| Cost optimization via routing      | Requires complexity classification accuracy           |
| Programmatic verification          | Verification itself adds latency and cost             |
| Component swappability             | Interface contracts must be maintained across changes |
| System-level optimization          | Larger search space than single-model prompt tuning   |
| Multiple failure recovery paths    | More failure modes to handle                          |

## When to Use

✅ **Good fit for:**

- Production AI applications where quality must exceed what a single model call provides
- Cost-sensitive deployments that need to route between model tiers
- Tasks requiring verifiable outputs (code generation, structured data extraction, compliance)
- Systems where retrieval must be combined with generation and validation
- Applications with diverse query types needing different processing pipelines
- Teams optimizing AI system performance through systematic evaluation

❌ **Not ideal for:**

- Simple tasks where a single well-prompted LLM call is sufficient
- Prototyping or proof-of-concept work where speed of development matters most
- Extremely latency-sensitive applications where multi-component overhead is prohibitive
- Small teams without capacity to maintain multi-component systems

## References

- [The Shift from Models to Compound AI Systems — Berkeley AI Research (2024)](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/)
- [DSPy: Compiling Declarative Language Model Calls — Khattab et al. (2023)](https://arxiv.org/abs/2310.03714)
- [FrugalGPT: How to Use Large Language Models While Reducing Cost — Chen et al. (2023)](https://arxiv.org/abs/2305.05176)
- [Gorilla: Large Language Model Connected with APIs — Patil et al. (2023)](https://arxiv.org/abs/2305.15334)
- [RouteLLM: Learning to Route LLMs — Ong et al. (2024)](https://arxiv.org/abs/2406.18665)
- [AlpacaEval: Automatic Evaluation of Instruction-Following Models](https://github.com/tatsu-lab/alpaca_eval)
