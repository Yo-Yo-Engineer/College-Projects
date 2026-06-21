# LLM Application Architecture

## Overview

**LLM Application Architecture** defines the structural patterns for building production systems powered by large language models. Unlike traditional software — where logic is deterministic and encoded in source code — LLM applications delegate reasoning to probabilistic models, requiring distinct architectural concerns: prompt management, model routing, output validation, evaluation, cost governance, and responsible AI guardrails.

This document covers the end-to-end architecture from user request to grounded response, including the GenAI gateway pattern for managing model endpoints at scale.

Key principles:

- **Model as a Service** — Treat the LLM as an external, swappable dependency behind an abstraction layer, not a hardcoded component
- **Prompt Engineering as Configuration** — Prompts are versioned, tested, and managed artifacts — not inline strings
- **Evaluation-Driven Development** — Measure LLM output quality with automated evaluations before and after deployment
- **Defense in Depth** — Apply guardrails at input, output, and tool-use boundaries
- **Cost Awareness** — LLM inference is metered; architecture must support model routing, caching, and token budgeting

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       LLM Application Architecture                           │
│                                                                             │
│   User Request                                                              │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────┐                                                          │
│   │  API Gateway   │  (auth, rate limiting, request validation)              │
│   └──────┬───────┘                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────┐   │
│   │  Input        │────▶│  Prompt       │────▶│  GenAI Gateway           │   │
│   │  Guardrails   │     │  Builder      │     │  (model routing, LB,     │   │
│   └──────────────┘     └──────────────┘     │   retry, token mgmt)     │   │
│                                              └───────────┬──────────────┘   │
│                                                          │                  │
│                              ┌────────────────────────────┤                  │
│                              │                            │                  │
│                              ▼                            ▼                  │
│                    ┌──────────────────┐        ┌──────────────────┐         │
│                    │  LLM Provider A   │        │  LLM Provider B   │        │
│                    │  (e.g. GPT-4)     │        │  (e.g. Claude)    │        │
│                    └────────┬─────────┘        └────────┬─────────┘        │
│                             │                           │                   │
│                             └─────────┬─────────────────┘                   │
│                                       │                                     │
│                                       ▼                                     │
│                             ┌──────────────────┐                            │
│                             │  Output           │                            │
│                             │  Guardrails       │                            │
│                             └────────┬─────────┘                            │
│                                      │                                      │
│                                      ▼                                      │
│                             ┌──────────────────┐                            │
│                             │  Response         │                            │
│                             │  Formatter        │                            │
│                             └────────┬─────────┘                            │
│                                      │                                      │
│                                      ▼                                      │
│                              Structured Response                            │
│                                                                             │
│   Cross-Cutting: ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│                   │ Observ-     │  │  Evaluation   │  │  Prompt Versioning │  │
│                   │ ability     │  │  Pipeline     │  │  & Registry        │  │
│                   └────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Prompt Management

Prompts are first-class artifacts — versioned, tested, and managed separately from application code:

```
// Prompt Registry — Versioned store of prompt templates
INTERFACE PromptRegistry
    FUNCTION get(name : String, version : String OR NULL) -> PromptTemplate
    FUNCTION register(template : PromptTemplate) -> VersionInfo
    FUNCTION listVersions(name : String) -> List<VersionInfo>
    FUNCTION promote(name : String, version : String, stage : Stage) -> Void
END INTERFACE

DATA PromptTemplate
    name         : String
    version      : String
    systemPrompt : String
    userTemplate : String          // Template with {{placeholders}}
    modelConfig  : ModelConfig     // Temperature, max tokens, etc.
    metadata     : Map<String, Any>
    createdAt    : DateTime
END DATA

// Prompt Builder — Assembles final prompts from templates and context
CLASS PromptBuilder
    CONSTRUCTOR(
        registry     : PromptRegistry,
        contextStore : ContextStore
    )

    FUNCTION build(templateName : String,
                   variables : Map<String, String>,
                   context : RequestContext) -> BuiltPrompt

        template = registry.get(templateName, version = context.promptVersion)

        // Resolve template variables
        systemPrompt = template.systemPrompt
        userMessage  = resolveTemplate(template.userTemplate, variables)

        // Attach additional context (e.g. retrieved documents, user profile)
        additionalContext = contextStore.getContextForRequest(context)

        messages = EMPTY LIST

        messages.ADD(NEW Message(
            role    = "system",
            content = systemPrompt
        ))

        IF additionalContext IS NOT EMPTY THEN
            messages.ADD(NEW Message(
                role    = "system",
                content = "CONTEXT:\n" + formatContext(additionalContext)
            ))
        END IF

        messages.ADD(NEW Message(role = "user", content = userMessage))

        RETURN NEW BuiltPrompt(
            messages    = messages,
            modelConfig = template.modelConfig,
            metadata    = { templateName: templateName, version: template.version }
        )
END CLASS
```

### GenAI Gateway

The GenAI gateway is intelligent middleware that sits between the application and LLM providers, handling model routing, load balancing, token management, and cost governance:

```
// GenAI Gateway — Central point for all LLM inference requests
CLASS GenAIGateway
    CONSTRUCTOR(
        endpoints      : List<LLMEndpoint>,
        router         : ModelRouter,
        rateLimiter    : TokenRateLimiter,
        responseCache  : ResponseCache,
        costTracker    : CostTracker,
        retryPolicy    : RetryPolicy
    )

    FUNCTION invoke(request : LLMRequest) -> LLMResponse
        // 1. Check cache for identical requests
        cachedResponse = responseCache.get(request.cacheKey())
        IF cachedResponse IS NOT NULL THEN
            costTracker.recordCacheHit(request)
            RETURN cachedResponse
        END IF

        // 2. Route to appropriate model endpoint
        selectedEndpoint = router.route(request)

        // 3. Check rate limits (tokens per minute / requests per minute)
        IF NOT rateLimiter.allowRequest(selectedEndpoint, request.estimatedTokens) THEN
            // Try fallback endpoint
            selectedEndpoint = router.routeFallback(request)
            IF selectedEndpoint IS NULL THEN
                THROW RateLimitExceededError("All endpoints exhausted")
            END IF
        END IF

        // 4. Execute with retry
        response = retryPolicy.execute(
            FUNCTION() ->
                selectedEndpoint.complete(request)
        )

        // 5. Track cost
        costTracker.record(
            endpoint    = selectedEndpoint,
            inputTokens = response.usage.inputTokens,
            outputTokens = response.usage.outputTokens,
            consumer    = request.consumerTag
        )

        // 6. Cache response
        responseCache.put(request.cacheKey(), response, ttl = request.cacheTTL)

        RETURN response
END CLASS

// Model Router — Selects the best endpoint based on request characteristics
CLASS ModelRouter
    PROPERTIES
        endpoints     : List<LLMEndpoint>
        routingRules  : List<RoutingRule>

    FUNCTION route(request : LLMRequest) -> LLMEndpoint
        // Apply routing rules in priority order
        FOR EACH rule IN routingRules
            IF rule.matches(request) THEN
                RETURN rule.selectEndpoint(endpoints)
            END IF
        END FOR

        // Default: round-robin across available endpoints
        RETURN selectRoundRobin(endpoints)

    FUNCTION routeFallback(request : LLMRequest) -> LLMEndpoint OR NULL
        // Try endpoints in priority order, skipping rate-limited ones
        FOR EACH endpoint IN endpoints.SORTED_BY(priority)
            IF endpoint.isAvailable() AND NOT endpoint.isRateLimited() THEN
                RETURN endpoint
            END IF
        END FOR
        RETURN NULL
END CLASS
```

### Routing Strategies

```
// Routing rules for directing requests to appropriate models

// Complexity-based routing — Simple queries to smaller models, complex to larger
CLASS ComplexityBasedRouter IMPLEMENTS RoutingRule
    FUNCTION matches(request : LLMRequest) -> Boolean
        RETURN TRUE  // Always applicable as fallback

    FUNCTION selectEndpoint(endpoints : List<LLMEndpoint>) -> LLMEndpoint
        complexity = estimateComplexity(request)

        MATCH complexity
            CASE LOW:
                RETURN findEndpoint(endpoints, preferModel = "small-fast")
            CASE MEDIUM:
                RETURN findEndpoint(endpoints, preferModel = "balanced")
            CASE HIGH:
                RETURN findEndpoint(endpoints, preferModel = "large-capable")
        END MATCH
END CLASS

// Cost-optimized routing — Prefer provisioned throughput, spill over to pay-as-you-go
CLASS CostOptimizedRouter IMPLEMENTS RoutingRule
    FUNCTION selectEndpoint(endpoints : List<LLMEndpoint>) -> LLMEndpoint
        // 1. Prefer provisioned throughput units (pre-paid, lower cost)
        provisioned = endpoints.FILTER(e -> e.billingType == "PTU")
        FOR EACH endpoint IN provisioned
            IF endpoint.hasCapacity() THEN
                RETURN endpoint
            END IF
        END FOR

        // 2. Fall back to pay-as-you-go
        payg = endpoints.FILTER(e -> e.billingType == "PAYG")
        RETURN selectLeastLoaded(payg)
END CLASS
```

### Guardrails

Apply content safety at the application boundary — before and after the LLM call:

```
// Input Guardrails — Filter and validate user input before reaching the LLM
CLASS InputGuardrails
    CONSTRUCTOR(
        contentFilter   : ContentSafetyFilter,
        piiDetector     : PIIDetector,
        promptInjection : PromptInjectionDetector,
        topicFilter     : TopicFilter
    )

    FUNCTION validate(userInput : String, context : RequestContext) -> GuardrailResult
        checks = EMPTY LIST

        // 1. Prompt injection detection
        injectionScore = promptInjection.detect(userInput)
        checks.ADD(NEW Check(
            name   = "prompt_injection",
            passed = injectionScore < INJECTION_THRESHOLD,
            score  = injectionScore
        ))

        // 2. Content safety (harmful, hateful, violent content)
        safetyResult = contentFilter.analyze(userInput)
        checks.ADD(NEW Check(
            name   = "content_safety",
            passed = safetyResult.isSafe,
            detail = safetyResult.categories
        ))

        // 3. PII detection and masking
        piiEntities = piiDetector.detect(userInput)
        IF piiEntities IS NOT EMPTY THEN
            maskedInput = piiDetector.mask(userInput, piiEntities)
            checks.ADD(NEW Check(
                name       = "pii_detected",
                passed     = TRUE,     // Masked, not blocked
                detail     = "PII masked: " + piiEntities.SIZE() + " entities",
                transforms = { maskedInput: maskedInput }
            ))
        END IF

        // 4. Off-topic filtering
        IF NOT topicFilter.isOnTopic(userInput, context.allowedTopics) THEN
            checks.ADD(NEW Check(
                name   = "topic_filter",
                passed = FALSE,
                detail = "Input does not match allowed topics"
            ))
        END IF

        allPassed = ALL(check.passed FOR EACH check IN checks)
        RETURN NEW GuardrailResult(passed = allPassed, checks = checks)
END CLASS

// Output Guardrails — Validate LLM response before returning to user
CLASS OutputGuardrails
    CONSTRUCTOR(
        contentFilter    : ContentSafetyFilter,
        groundednessCheck : GroundednessChecker,
        formatValidator  : OutputFormatValidator
    )

    FUNCTION validate(response : LLMResponse,
                      context : RequestContext) -> GuardrailResult
        checks = EMPTY LIST

        // 1. Content safety on output
        safetyResult = contentFilter.analyze(response.content)
        checks.ADD(NEW Check(
            name   = "output_safety",
            passed = safetyResult.isSafe
        ))

        // 2. Groundedness — Does the response stay within provided context?
        IF context.retrievedDocuments IS NOT EMPTY THEN
            groundedness = groundednessCheck.evaluate(
                response = response.content,
                sources  = context.retrievedDocuments
            )
            checks.ADD(NEW Check(
                name   = "groundedness",
                passed = groundedness.score >= GROUNDEDNESS_THRESHOLD,
                score  = groundedness.score
            ))
        END IF

        // 3. Output format validation (JSON, structured data)
        IF context.expectedFormat IS NOT NULL THEN
            formatResult = formatValidator.validate(response.content, context.expectedFormat)
            checks.ADD(NEW Check(
                name   = "format_validation",
                passed = formatResult.isValid
            ))
        END IF

        allPassed = ALL(check.passed FOR EACH check IN checks)
        RETURN NEW GuardrailResult(passed = allPassed, checks = checks)
END CLASS
```

### Evaluation Pipeline

Automated evaluation measures LLM output quality across dimensions:

```
// LLM Evaluation Pipeline — Automated quality assessment
CLASS LLMEvaluationPipeline
    CONSTRUCTOR(
        evaluators  : List<Evaluator>,
        testDataset : List<EvalTestCase>,
        llmJudge    : LanguageModel
    )

    FUNCTION evaluate(pipeline : LLMPipeline) -> EvaluationReport
        results = EMPTY LIST

        FOR EACH testCase IN testDataset
            response = pipeline.invoke(testCase.input)

            metrics = NEW EvalMetrics()

            FOR EACH evaluator IN evaluators
                score = evaluator.evaluate(
                    input          = testCase.input,
                    output         = response.content,
                    expectedOutput = testCase.expectedOutput,
                    context        = testCase.context
                )
                metrics.SET(evaluator.name, score)
            END FOR

            results.ADD(metrics)
        END FOR

        RETURN aggregateAndReport(results)
END CLASS

// Common evaluators
CLASS RelevanceEvaluator IMPLEMENTS Evaluator
    PROPERTIES
        name = "relevance"

    FUNCTION evaluate(input, output, expectedOutput, context) -> Decimal
        // LLM-as-judge: Does the output answer the input question?
        RETURN llmJudge.score(
            criteria = "Does the response directly and completely answer the question?",
            input    = input,
            output   = output
        )
END CLASS

CLASS CoherenceEvaluator IMPLEMENTS Evaluator
    PROPERTIES
        name = "coherence"

    FUNCTION evaluate(input, output, expectedOutput, context) -> Decimal
        // LLM-as-judge: Is the output well-structured and logically coherent?
        RETURN llmJudge.score(
            criteria = "Is the response coherent, well-organized, and logically structured?",
            input    = input,
            output   = output
        )
END CLASS

CLASS SimilarityEvaluator IMPLEMENTS Evaluator
    PROPERTIES
        name = "similarity"

    FUNCTION evaluate(input, output, expectedOutput, context) -> Decimal
        // Embedding-based semantic similarity with expected output
        outputVector   = embeddingModel.embed(output)
        expectedVector = embeddingModel.embed(expectedOutput)
        RETURN cosineSimilarity(outputVector, expectedVector)
END CLASS
```

### Structured Output

```
// Structured Output — Constrain LLM responses to valid schemas
CLASS StructuredOutputHandler
    CONSTRUCTOR(llm : LanguageModel)

    FUNCTION generateStructured(prompt : BuiltPrompt,
                                 outputSchema : JSONSchema) -> ParsedOutput
        // Include schema in system prompt
        schemaInstruction = "You MUST respond with valid JSON matching this schema:\n" +
                            SERIALIZE_TO_JSON(outputSchema)

        prompt.messages[0].content += "\n\n" + schemaInstruction

        // Generate with constrained decoding (if model supports it)
        response = llm.generate(
            messages      = prompt.messages,
            responseFormat = { type: "json_schema", schema: outputSchema }
        )

        // Parse and validate
        parsed = PARSE_JSON(response.content)
        validation = validateAgainstSchema(parsed, outputSchema)

        IF NOT validation.isValid THEN
            // Retry with error feedback
            RETURN retryWithFeedback(prompt, outputSchema, validation.errors)
        END IF

        RETURN NEW ParsedOutput(data = parsed, rawResponse = response)
END CLASS
```

### Choosing a Strategy: RAG vs Fine-Tuning vs Prompt Engineering

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    Strategy Selection Guide                               │
├────────────────────┬─────────────────────────────────────────────────────┤
│  Strategy          │  When to Use                                        │
├────────────────────┼─────────────────────────────────────────────────────┤
│  Prompt            │  Behavior or output format changes. Context fits    │
│  Engineering       │  within the model's context window. No proprietary  │
│                    │  knowledge needed beyond training data.             │
├────────────────────┼─────────────────────────────────────────────────────┤
│  RAG               │  Application requires access to current,            │
│                    │  domain-specific, or private knowledge. Content     │
│                    │  changes frequently. Citations needed.              │
├────────────────────┼─────────────────────────────────────────────────────┤
│  Fine-Tuning       │  Model must learn a new style, domain language,    │
│                    │  or task-specific behavior not achievable via       │
│                    │  prompting. Latency is critical and context must    │
│                    │  be minimized. Have labeled training examples.      │
├────────────────────┼─────────────────────────────────────────────────────┤
│  Combined          │  Fine-tune for style and domain language; add RAG  │
│  (RAG + Fine-Tune) │  for current knowledge. Apply prompt engineering   │
│                    │  to control output format and behavior.            │
└────────────────────┴─────────────────────────────────────────────────────┘

        Increasing effort and cost →
        Prompt Engineering → RAG → Fine-Tuning → Combined
```

### Observability

```
// LLM Observability — Instrument all inference requests
CLASS LLMObservabilityMiddleware
    CONSTRUCTOR(
        tracer       : DistributedTracer,
        metricsStore : MetricsStore,
        logger       : StructuredLogger
    )

    FUNCTION wrap(request : LLMRequest, invoker : Function) -> LLMResponse
        span = tracer.startSpan("llm_inference")
        span.setAttribute("model", request.model)
        span.setAttribute("consumer", request.consumerTag)
        span.setAttribute("template", request.metadata.templateName)

        startTime = NOW()

        TRY
            response = invoker(request)

            // Record metrics
            latency = NOW() - startTime
            metricsStore.record("llm_latency", latency, tags = { model: request.model })
            metricsStore.record("llm_input_tokens", response.usage.inputTokens)
            metricsStore.record("llm_output_tokens", response.usage.outputTokens)

            // Log for audit trail
            logger.info("LLM inference complete", {
                model         : request.model,
                inputTokens   : response.usage.inputTokens,
                outputTokens  : response.usage.outputTokens,
                latencyMs     : latency,
                consumer      : request.consumerTag,
                promptVersion : request.metadata.version,
                cached        : FALSE
            })

            span.setStatus("OK")
            RETURN response

        CATCH error
            span.setStatus("ERROR", error.message)
            metricsStore.increment("llm_errors", tags = { model: request.model })
            THROW error

        FINALLY
            span.end()
END CLASS
```

## Project Structure

```
src/
├── prompts/                        # Prompt Management
│   ├── templates/                  # Prompt template definitions
│   ├── registry/                   # Versioned prompt storage
│   ├── builder/                    # Prompt assembly logic
│   └── variables/                  # Variable resolvers
│
├── gateway/                        # GenAI Gateway
│   ├── router/                     # Model routing strategies
│   ├── endpoints/                  # LLM provider adapters
│   ├── rate_limiting/              # Token and request rate limiters
│   ├── cache/                      # Response caching
│   ├── retry/                      # Retry and circuit breaker policies
│   └── cost_tracking/              # Per-consumer cost accounting
│
├── guardrails/                     # Input / Output Safety
│   ├── input/
│   │   ├── prompt_injection/
│   │   ├── content_safety/
│   │   ├── pii_detection/
│   │   └── topic_filtering/
│   ├── output/
│   │   ├── content_safety/
│   │   ├── groundedness/
│   │   └── format_validation/
│   └── policies/                   # Configurable guardrail policies
│
├── structured_output/              # Schema-constrained generation
│   ├── schemas/
│   └── parsers/
│
├── evaluation/                     # Evaluation Pipeline
│   ├── evaluators/                 # Relevance, coherence, similarity, etc.
│   ├── datasets/                   # Test datasets
│   ├── judges/                     # LLM-as-judge implementations
│   └── reports/
│
├── observability/                  # Monitoring and Tracing
│   ├── tracing/
│   ├── metrics/
│   ├── logging/
│   └── dashboards/
│
├── api/                            # Application API
│   ├── endpoints/
│   └── middleware/
│
├── config/
│
└── tests/
    ├── unit/
    ├── integration/
    └── evaluation/
```

## Benefits

1. **Model Portability** — Gateway abstraction enables switching LLM providers without application changes
2. **Cost Governance** — Token tracking, caching, model routing, and rate limiting control spend
3. **Quality Assurance** — Evaluation pipelines catch regressions before deployment
4. **Safety** — Multi-layer guardrails protect against prompt injection, PII leakage, and harmful content
5. **Prompt Lifecycle** — Version-controlled prompts enable A/B testing, rollback, and audit
6. **Observability** — Full tracing of inference requests supports debugging and optimization

## Trade-offs

| Advantage                          | Consideration                                          |
| ---------------------------------- | ------------------------------------------------------ |
| Model-agnostic gateway abstraction | Additional latency from gateway routing and guardrails |
| Multi-provider cost optimization   | Complexity of managing multiple LLM endpoints          |
| Versioned prompts with rollback    | Requires prompt registry infrastructure                |
| Automated evaluation pipelines     | LLM-as-judge evaluations have their own biases         |
| Multi-layer guardrails             | False positives can degrade user experience            |

## When to Use

✅ **Good fit for:**

- Production AI applications serving end users
- Enterprise systems requiring content safety, PII handling, and audit trails
- Multi-model deployments needing intelligent routing and cost management
- Applications requiring prompt versioning, A/B testing, and evaluation
- Teams building shared AI infrastructure across multiple products

❌ **Not ideal for:**

- Simple prototypes or one-off experiments
- Single-model applications with no cost or safety concerns
- Offline analysis where latency and guardrails are irrelevant
- Applications where prompts are trivial and unlikely to change

## References

- [Designing and Implementing a GenAI Gateway — Microsoft AI Playbook](https://learn.microsoft.com/en-us/ai/playbook/technology-guidance/generative-ai/dev-starters/genai-gateway)
- [Baseline Foundry Chat Reference Architecture — Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat)
- [Building Effective Agents — Anthropic (2024)](https://www.anthropic.com/engineering/building-effective-agents)
- [Prompt Engineering Guide — DAIR.AI](https://www.promptingguide.ai/)
- [LLM Evaluation Best Practices — Azure AI](https://learn.microsoft.com/en-us/azure/ai-studio/concepts/evaluation-approach-gen-ai)
- [Responsible AI Practices — Microsoft](https://www.microsoft.com/en-us/ai/responsible-ai)
