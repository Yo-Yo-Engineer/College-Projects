# Guardrails & Content Filtering Architecture

## Overview

**Guardrails & Content Filtering Architecture** defines the structural patterns for building runtime safety enforcement layers around AI systems. Unlike AI governance (which addresses organizational policy, auditing, and compliance at the program level), guardrails are the **runtime enforcement mechanisms** that actively validate, filter, transform, and block inputs and outputs in real-time during every AI interaction.

This document covers input validation pipelines, output safety filtering, prompt injection defense, PII protection, topic restriction, hallucination detection, and configurable policy engines for production AI systems.

Key principles:

- **Defense in Depth** — Apply guardrails at multiple layers: input, tool use, and output — never rely on a single check
- **Configurable Policies** — Safety rules are externalized as configuration, not hardcoded, enabling per-application and per-tenant customization
- **Fail Closed** — When guardrail evaluation fails or times out, block the request rather than letting it through
- **Minimize False Positives** — Overly aggressive filtering degrades user experience; balance safety with utility
- **Observable** — Every guardrail decision is logged with the rule that triggered it, enabling tuning and audit

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Guardrails & Content Filtering Architecture                    │
│                                                                             │
│   User Input                                                                │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    INPUT GUARDRAILS                                │     │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │     │
│   │  │  Prompt     │ │  Content   │ │  PII        │ │  Topic        │  │    │
│   │  │  Injection  │ │  Safety    │ │  Detection  │ │  Restriction  │  │    │
│   │  │  Detection  │ │  Filter    │ │  & Masking  │ │  Filter       │  │    │
│   │  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │     │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    LLM / AI MODEL                                 │     │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    OUTPUT GUARDRAILS                               │     │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │     │
│   │  │  Content    │ │  Grounded-  │ │  PII        │ │  Format &     │  │    │
│   │  │  Safety     │ │  ness       │ │  Scrubbing  │ │  Schema       │  │    │
│   │  │  Filter     │ │  Check      │ │  (output)   │ │  Validation   │  │    │
│   │  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │     │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    TOOL-USE GUARDRAILS                             │     │
│   │  ┌────────────┐ ┌──────────────┐ ┌────────────────────────────┐  │     │
│   │  │  Permission │ │  Argument     │ │  Result Validation          │  │    │
│   │  │  Check      │ │  Sanitization │ │  (before returning to LLM) │  │    │
│   │  └────────────┘ └──────────────┘ └────────────────────────────┘  │     │
│   └──────────────────────────┬───────────────────────────────────────┘      │
│                              │                                              │
│                              ▼                                              │
│                     Safe Response to User                                   │
│                                                                             │
│   Cross-Cutting:                                                            │
│   ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐  │
│   │ Policy      │  │  Audit        │  │  Metrics    │  │  Override &       │  │
│   │ Engine      │  │  Logger       │  │  Dashboard  │  │  Escalation       │  │
│   └────────────┘  └──────────────┘  └────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Guardrail Categories

```
┌───────────────────────────────────────────────────────────────────────┐
│                   Guardrail Categories                                │
├──────────────────┬────────────────────────────────────────────────────┤
│  Category        │  Description                                      │
├──────────────────┼────────────────────────────────────────────────────┤
│  Prompt          │  Detect and block attempts to override system     │
│  Injection       │  instructions, extract prompts, or manipulate    │
│  Defense         │  model behavior through crafted inputs.           │
├──────────────────┼────────────────────────────────────────────────────┤
│  Content         │  Filter harmful content across categories:        │
│  Safety          │  violence, hate, sexual, self-harm, harassment.  │
│                  │  Applied to both input and output.                │
├──────────────────┼────────────────────────────────────────────────────┤
│  PII             │  Detect and mask/redact personally identifiable  │
│  Protection      │  information: names, emails, phone numbers,      │
│                  │  SSNs, credit cards, addresses.                   │
├──────────────────┼────────────────────────────────────────────────────┤
│  Topic           │  Restrict the AI to approved topics. Block        │
│  Restriction     │  off-topic requests (e.g., a customer service    │
│                  │  bot refusing to discuss competitors).            │
├──────────────────┼────────────────────────────────────────────────────┤
│  Groundedness    │  Verify that model outputs are supported by       │
│  Verification    │  provided context. Detect hallucinated claims    │
│                  │  or fabricated references.                        │
├──────────────────┼────────────────────────────────────────────────────┤
│  Output Format   │  Validate that output conforms to expected        │
│  Validation      │  schema (JSON, structured data) before returning │
│                  │  to the application.                              │
├──────────────────┼────────────────────────────────────────────────────┤
│  Tool-Use        │  Validate that tool calls are permitted, arguments│
│  Guardrails      │  are safe, and results are appropriate before    │
│                  │  the LLM acts on them.                            │
├──────────────────┼────────────────────────────────────────────────────┤
│  Rate & Cost     │  Enforce per-user or per-session token limits,   │
│  Guardrails      │  request frequency caps, and cost ceilings.      │
└──────────────────┴────────────────────────────────────────────────────┘
```

### Guardrail Execution Pipeline

Guardrails execute as an ordered pipeline where each check can pass, transform, or block:

```
┌──────────────────────────────────────────────────────────────────────┐
│              Guardrail Execution Pipeline                             │
│                                                                      │
│  Input                                                               │
│    │                                                                 │
│    ▼                                                                 │
│  ┌──────────┐   PASS    ┌──────────┐   PASS    ┌──────────┐        │
│  │ Guard 1   │─────────▶│ Guard 2   │─────────▶│ Guard 3   │───▶ OK │
│  └──────────┘           └──────────┘           └──────────┘        │
│    │                      │                      │                  │
│    │ BLOCK                │ TRANSFORM             │ BLOCK            │
│    ▼                      ▼                      ▼                  │
│  Rejected              Modified Input          Rejected             │
│  (return error)        (continues with         (return error)       │
│                         modified data)                              │
│                                                                      │
│  Actions per guard:                                                  │
│    PASS      — Input is safe, continue to next guard                │
│    TRANSFORM — Modify input (e.g., mask PII) and continue           │
│    BLOCK     — Reject the request with a reason                     │
│    WARN      — Log warning but allow (for monitoring mode)          │
└──────────────────────────────────────────────────────────────────────┘
```

### Policy Configuration

Guardrails are driven by externalized, configurable policies:

```
┌──────────────────────────────────────────────────────────────────────┐
│              Policy Configuration Example                            │
│                                                                      │
│  policy:                                                             │
│    name: "customer_support_bot"                                      │
│    version: "2.1"                                                    │
│                                                                      │
│    input_guards:                                                     │
│      - type: prompt_injection                                        │
│        action: BLOCK                                                 │
│        threshold: 0.85                                               │
│                                                                      │
│      - type: content_safety                                          │
│        action: BLOCK                                                 │
│        categories: [violence, hate, sexual, self_harm]               │
│        threshold: MEDIUM                                             │
│                                                                      │
│      - type: pii_detection                                           │
│        action: TRANSFORM                 # mask, don't block         │
│        entities: [email, phone, ssn, credit_card]                    │
│                                                                      │
│      - type: topic_restriction                                       │
│        action: BLOCK                                                 │
│        allowed_topics: [product_support, billing, returns]           │
│        blocked_topics: [competitors, politics, medical_advice]       │
│                                                                      │
│    output_guards:                                                    │
│      - type: content_safety                                          │
│        action: BLOCK                                                 │
│        threshold: LOW                    # stricter on output        │
│                                                                      │
│      - type: groundedness                                            │
│        action: WARN                      # log but don't block yet   │
│        threshold: 0.7                                                │
│                                                                      │
│      - type: pii_detection                                           │
│        action: TRANSFORM                                             │
│        entities: [all]                                               │
│                                                                      │
│    tool_guards:                                                      │
│      - type: permission_check                                        │
│        allowed_tools: [search_kb, lookup_order, track_shipment]      │
│        blocked_tools: [delete_account, modify_billing]               │
│                                                                      │
│    rate_limits:                                                      │
│      max_requests_per_minute: 30                                     │
│      max_tokens_per_session: 50000                                   │
└──────────────────────────────────────────────────────────────────────┘
```

### Prompt Injection Defense Strategies

```
┌───────────────────────────────────────────────────────────────────────┐
│              Prompt Injection Defense Layers                          │
│                                                                      │
│  Layer 1: Input Classification                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  ML classifier trained on injection examples.                │    │
│  │  Scores input 0.0 (safe) → 1.0 (injection).                │    │
│  │  Block if score > threshold.                                 │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Layer 2: Delimiter-Based Isolation                                  │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Wrap user input in clear delimiters to separate it from     │    │
│  │  system instructions. E.g., <user_input>...</user_input>     │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Layer 3: Instruction Hierarchy                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  System prompt explicitly states that user input cannot       │    │
│  │  override system instructions. Models trained to respect     │    │
│  │  instruction hierarchy.                                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Layer 4: Output Verification                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  After LLM response, verify it doesn't contain system prompt │    │
│  │  content, instructions leakage, or policy violations.        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Layer 5: Canary Tokens                                              │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Embed unique tokens in system prompt. If they appear in     │    │
│  │  output, system prompt extraction was attempted.             │    │
│  └──────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────┘
```

## Implementation

### Guardrail Pipeline Engine

```
// Guardrail Pipeline — Executes guards in sequence
CLASS GuardrailPipeline
    CONSTRUCTOR(
        guards      : List<Guard>,
        policy      : GuardrailPolicy,
        auditLogger : AuditLogger,
        metrics     : GuardrailMetrics
    )

    FUNCTION evaluate(content : String,
                      context : GuardrailContext) -> GuardrailResult
        currentContent = content
        allChecks = EMPTY LIST

        FOR EACH guard IN guards
            // Skip guards not enabled in policy
            IF NOT policy.isEnabled(guard.type, context.direction) THEN
                CONTINUE
            END IF

            startTime = NOW()
            guardConfig = policy.getConfig(guard.type, context.direction)

            TRY
                checkResult = guard.check(currentContent, guardConfig, context)
            CATCH error
                // Fail closed: treat errors as blocks
                checkResult = NEW CheckResult(
                    action = Action.BLOCK,
                    reason = "Guard error: " + error.message,
                    guardType = guard.type
                )
            END TRY

            latency = NOW() - startTime
            metrics.recordCheck(guard.type, checkResult.action, latency)

            allChecks.ADD(checkResult)

            MATCH checkResult.action
                CASE Action.BLOCK:
                    auditLogger.log(NEW AuditEntry(
                        direction = context.direction,
                        guardType = guard.type,
                        action    = "BLOCKED",
                        reason    = checkResult.reason,
                        content   = redactForAudit(currentContent)
                    ))
                    RETURN NEW GuardrailResult(
                        passed  = FALSE,
                        action  = Action.BLOCK,
                        reason  = checkResult.reason,
                        checks  = allChecks
                    )

                CASE Action.TRANSFORM:
                    currentContent = checkResult.transformedContent
                    auditLogger.log(NEW AuditEntry(
                        direction = context.direction,
                        guardType = guard.type,
                        action    = "TRANSFORMED",
                        reason    = checkResult.reason
                    ))

                CASE Action.WARN:
                    auditLogger.log(NEW AuditEntry(
                        direction = context.direction,
                        guardType = guard.type,
                        action    = "WARNED",
                        reason    = checkResult.reason
                    ))

                CASE Action.PASS:
                    // Continue to next guard
            END MATCH
        END FOR

        RETURN NEW GuardrailResult(
            passed           = TRUE,
            action           = Action.PASS,
            processedContent = currentContent,
            checks           = allChecks
        )
END CLASS
```

### Prompt Injection Detector

```
// Prompt Injection Detector — Multi-layer injection defense
CLASS PromptInjectionDetector IMPLEMENTS Guard
    CONSTRUCTOR(
        classifierModel : InjectionClassifier,
        patternMatcher  : PatternMatcher,
        canaryTokens    : List<String>
    )

    PROPERTIES
        type = "prompt_injection"

    FUNCTION check(content : String,
                   config : GuardConfig,
                   context : GuardrailContext) -> CheckResult

        // Layer 1: Pattern matching (known injection patterns)
        patternMatch = patternMatcher.match(content, INJECTION_PATTERNS)
        IF patternMatch.found THEN
            RETURN NEW CheckResult(
                action = config.action,
                reason = "Known injection pattern detected: " + patternMatch.pattern,
                score  = 1.0
            )
        END IF

        // Layer 2: ML classifier
        score = classifierModel.predict(content)
        IF score >= config.threshold THEN
            RETURN NEW CheckResult(
                action = config.action,
                reason = "Injection classifier score: " + score,
                score  = score
            )
        END IF

        // Layer 3: Canary token check (for output direction)
        IF context.direction == "OUTPUT" THEN
            FOR EACH token IN canaryTokens
                IF content.CONTAINS(token) THEN
                    RETURN NEW CheckResult(
                        action = Action.BLOCK,
                        reason = "Canary token detected in output — prompt extraction attempted",
                        score  = 1.0
                    )
                END IF
            END FOR
        END IF

        RETURN NEW CheckResult(action = Action.PASS, score = score)
END CLASS
```

### Content Safety Filter

```
// Content Safety Filter — Detect harmful content across categories
CLASS ContentSafetyFilter IMPLEMENTS Guard
    CONSTRUCTOR(
        safetyModel : ContentSafetyModel
    )

    PROPERTIES
        type = "content_safety"

    FUNCTION check(content : String,
                   config : GuardConfig,
                   context : GuardrailContext) -> CheckResult

        // Analyze across all safety categories
        analysis = safetyModel.analyze(content)

        violations = EMPTY LIST
        FOR EACH category IN config.categories
            score = analysis.getCategoryScore(category)
            threshold = resolveThreshold(config.threshold, category)

            IF score >= threshold THEN
                violations.ADD(NEW Violation(
                    category  = category,
                    score     = score,
                    threshold = threshold
                ))
            END IF
        END FOR

        IF violations IS NOT EMPTY THEN
            RETURN NEW CheckResult(
                action = config.action,
                reason = "Content safety violation: " +
                         violations.MAP(v -> v.category).JOIN(", "),
                details = violations
            )
        END IF

        RETURN NEW CheckResult(action = Action.PASS)
END CLASS
```

### PII Detector and Masker

```
// PII Protection — Detect and mask personally identifiable information
CLASS PIIProtector IMPLEMENTS Guard
    CONSTRUCTOR(
        nerModel       : NERModel,
        regexDetector  : RegexPIIDetector,
        maskingStrategy : MaskingStrategy
    )

    PROPERTIES
        type = "pii_detection"

    FUNCTION check(content : String,
                   config : GuardConfig,
                   context : GuardrailContext) -> CheckResult

        // 1. Detect PII using NER model
        nerEntities = nerModel.detectEntities(content)

        // 2. Detect PII using regex patterns (SSN, credit card, etc.)
        regexEntities = regexDetector.detect(content)

        // 3. Merge and deduplicate
        allEntities = mergeEntities(nerEntities, regexEntities)

        // 4. Filter to configured entity types
        relevantEntities = allEntities.FILTER(e ->
            config.entities.CONTAINS("all") OR
            config.entities.CONTAINS(e.type)
        )

        IF relevantEntities IS EMPTY THEN
            RETURN NEW CheckResult(action = Action.PASS)
        END IF

        // 5. Apply masking strategy
        IF config.action == Action.TRANSFORM THEN
            maskedContent = maskingStrategy.mask(content, relevantEntities)
            RETURN NEW CheckResult(
                action             = Action.TRANSFORM,
                transformedContent = maskedContent,
                reason             = "PII masked: " + relevantEntities.SIZE() + " entities",
                details            = summarizeEntities(relevantEntities)
            )
        END IF

        // Block mode
        RETURN NEW CheckResult(
            action = Action.BLOCK,
            reason = "PII detected: " +
                     relevantEntities.MAP(e -> e.type).DISTINCT().JOIN(", ")
        )
END CLASS

// Masking strategies
INTERFACE MaskingStrategy
    FUNCTION mask(content : String, entities : List<PIIEntity>) -> String
END INTERFACE

CLASS RedactMaskingStrategy IMPLEMENTS MaskingStrategy
    // Replace PII with [REDACTED]
    FUNCTION mask(content : String, entities : List<PIIEntity>) -> String
        result = content
        // Process in reverse order to preserve offsets
        FOR EACH entity IN entities.SORT_BY(e -> e.endOffset).REVERSE()
            result = result.REPLACE_RANGE(
                entity.startOffset, entity.endOffset, "[REDACTED]"
            )
        END FOR
        RETURN result
END CLASS

CLASS TypedMaskingStrategy IMPLEMENTS MaskingStrategy
    // Replace PII with type label: [EMAIL], [PHONE], [SSN]
    FUNCTION mask(content : String, entities : List<PIIEntity>) -> String
        result = content
        FOR EACH entity IN entities.SORT_BY(e -> e.endOffset).REVERSE()
            result = result.REPLACE_RANGE(
                entity.startOffset, entity.endOffset,
                "[" + entity.type.UPPER() + "]"
            )
        END FOR
        RETURN result
END CLASS
```

### Topic Restriction Filter

```
// Topic Restriction — Enforce allowed/blocked topic boundaries
CLASS TopicRestrictionFilter IMPLEMENTS Guard
    CONSTRUCTOR(
        classifier : TopicClassifier
    )

    PROPERTIES
        type = "topic_restriction"

    FUNCTION check(content : String,
                   config : GuardConfig,
                   context : GuardrailContext) -> CheckResult

        // Classify input topic
        topics = classifier.classify(content)

        // Check against blocked topics
        FOR EACH topic IN topics
            IF config.blockedTopics.CONTAINS(topic.label) AND
               topic.confidence >= 0.7 THEN
                RETURN NEW CheckResult(
                    action = config.action,
                    reason = "Blocked topic detected: " + topic.label
                )
            END IF
        END FOR

        // Check against allowed topics (if allowlist is configured)
        IF config.allowedTopics IS NOT EMPTY THEN
            matchesAllowed = topics.ANY(t ->
                config.allowedTopics.CONTAINS(t.label) AND t.confidence >= 0.5
            )
            IF NOT matchesAllowed THEN
                RETURN NEW CheckResult(
                    action = config.action,
                    reason = "Input does not match any allowed topic"
                )
            END IF
        END IF

        RETURN NEW CheckResult(action = Action.PASS)
END CLASS
```

### Groundedness Checker

```
// Groundedness — Verify outputs are supported by provided context
CLASS GroundednessChecker IMPLEMENTS Guard
    CONSTRUCTOR(
        nliModel    : NLIModel,
        llmJudge    : LanguageModel OR NULL
    )

    PROPERTIES
        type = "groundedness"

    FUNCTION check(content : String,
                   config : GuardConfig,
                   context : GuardrailContext) -> CheckResult

        // Groundedness only applies to output when context was provided
        IF context.direction != "OUTPUT" OR
           context.retrievedDocuments IS EMPTY THEN
            RETURN NEW CheckResult(action = Action.PASS)
        END IF

        // 1. Split output into individual claims
        claims = extractClaims(content)

        // 2. Check each claim against source documents
        ungrounded = EMPTY LIST
        FOR EACH claim IN claims
            supported = FALSE
            FOR EACH doc IN context.retrievedDocuments
                entailmentScore = nliModel.checkEntailment(
                    premise    = doc.content,
                    hypothesis = claim
                )
                IF entailmentScore >= config.threshold THEN
                    supported = TRUE
                    BREAK
                END IF
            END FOR

            IF NOT supported THEN
                ungrounded.ADD(claim)
            END IF
        END FOR

        IF ungrounded IS NOT EMPTY THEN
            groundednessScore = 1.0 - (ungrounded.SIZE() / claims.SIZE())
            RETURN NEW CheckResult(
                action = config.action,
                reason = "Ungrounded claims detected: " + ungrounded.SIZE() +
                         " of " + claims.SIZE() + " claims",
                score  = groundednessScore,
                details = ungrounded
            )
        END IF

        RETURN NEW CheckResult(action = Action.PASS, score = 1.0)
END CLASS
```

### Tool-Use Guardrails

```
// Tool-Use Guardrails — Validate agent tool calls
CLASS ToolUseGuardrails
    CONSTRUCTOR(
        policy     : GuardrailPolicy,
        sanitizer  : ArgumentSanitizer,
        auditLogger : AuditLogger
    )

    FUNCTION validateToolCall(toolCall : ToolCall,
                               agentContext : AgentContext) -> ToolGuardResult

        toolConfig = policy.getToolConfig(toolCall.name)

        // 1. Permission check
        IF toolConfig IS NULL OR NOT toolConfig.allowed THEN
            auditLogger.log("TOOL_BLOCKED", toolCall.name, "Not in allowed list")
            RETURN NEW ToolGuardResult(allowed = FALSE,
                                       reason = "Tool not permitted: " + toolCall.name)
        END IF

        // 2. Argument sanitization
        FOR EACH argName, argValue IN toolCall.arguments
            sanitizationResult = sanitizer.sanitize(argName, argValue, toolConfig)
            IF sanitizationResult.blocked THEN
                RETURN NEW ToolGuardResult(allowed = FALSE,
                                           reason = "Unsafe argument: " + argName)
            END IF
            IF sanitizationResult.modified THEN
                toolCall.arguments.SET(argName, sanitizationResult.sanitizedValue)
            END IF
        END FOR

        // 3. Require human approval for sensitive tools
        IF toolConfig.requiresApproval THEN
            RETURN NEW ToolGuardResult(
                allowed        = FALSE,
                requiresApproval = TRUE,
                toolCall       = toolCall
            )
        END IF

        RETURN NEW ToolGuardResult(allowed = TRUE)

    FUNCTION validateToolResult(result : ToolResult,
                                 toolCall : ToolCall) -> ToolGuardResult
        // Validate tool results before returning to LLM
        // Prevents data exfiltration via tool results
        IF result.containsSensitiveData() THEN
            result = redactSensitiveFields(result)
        END IF

        RETURN NEW ToolGuardResult(allowed = TRUE, result = result)
END CLASS
```

### Policy Engine

```
// Policy Engine — Load and manage guardrail policies
CLASS PolicyEngine
    CONSTRUCTOR(
        policyStore : PolicyStore,
        cache       : PolicyCache
    )

    FUNCTION getPolicy(applicationId : String,
                       tenantId : String OR NULL) -> GuardrailPolicy
        // Check cache first
        cacheKey = applicationId + ":" + (tenantId OR "default")
        cached = cache.get(cacheKey)
        IF cached IS NOT NULL THEN
            RETURN cached
        END IF

        // Load policy hierarchy: tenant overrides app defaults
        appPolicy = policyStore.load(applicationId)
        IF tenantId IS NOT NULL THEN
            tenantOverrides = policyStore.loadTenantOverrides(applicationId, tenantId)
            policy = mergePolicy(appPolicy, tenantOverrides)
        ELSE
            policy = appPolicy
        END IF

        cache.put(cacheKey, policy, ttl = 300)  // 5-minute cache
        RETURN policy

    FUNCTION updatePolicy(applicationId : String,
                          updatedPolicy : GuardrailPolicy) -> Void
        // Validate policy before saving
        validation = validatePolicy(updatedPolicy)
        IF NOT validation.isValid THEN
            THROW InvalidPolicyError(validation.errors)
        END IF

        policyStore.save(applicationId, updatedPolicy)
        cache.invalidate(applicationId + ":*")
END CLASS
```

## Project Structure

```
src/
├── guardrails/                        # Core Guardrail Engine
│   ├── pipeline/                      # Pipeline execution engine
│   ├── guards/                        # Individual guard implementations
│   │   ├── prompt_injection/          # Injection detection (classifier + patterns)
│   │   ├── content_safety/            # Harmful content filtering
│   │   ├── pii/                       # PII detection, masking, redaction
│   │   ├── topic_restriction/         # Topic classification and filtering
│   │   ├── groundedness/              # Hallucination / groundedness checking
│   │   ├── format_validation/         # Schema and format validation
│   │   └── rate_limiting/             # Token and request rate guards
│   └── tool_guards/                   # Tool-use validation
│       ├── permissions/               # Tool permission checking
│       ├── sanitization/              # Argument sanitization
│       └── result_validation/         # Tool result validation
│
├── policy/                            # Policy Management
│   ├── engine/                        # Policy loading and resolution
│   ├── store/                         # Policy storage (YAML/JSON/DB)
│   ├── schemas/                       # Policy validation schemas
│   └── templates/                     # Pre-built policy templates
│
├── models/                            # ML Models for Guards
│   ├── injection_classifier/          # Prompt injection detection model
│   ├── content_safety_model/          # Multi-category safety classifier
│   ├── ner_model/                     # Named entity recognition for PII
│   ├── topic_classifier/              # Topic classification model
│   └── nli_model/                     # Natural language inference for groundedness
│
├── masking/                           # PII Masking Strategies
│   ├── strategies/                    # Redact, type-label, hash, encrypt
│   └── reversible/                    # Reversible masking for authorized access
│
├── audit/                             # Audit and Compliance
│   ├── logger/                        # Structured audit logging
│   ├── reports/                       # Compliance reports
│   └── exports/                       # Audit data exports
│
├── observability/                     # Monitoring
│   ├── metrics/                       # Guard hit rates, latency, false positives
│   ├── dashboards/                    # Safety monitoring dashboards
│   └── alerts/                        # Anomaly alerts (spike in blocks, etc.)
│
├── config/
│   ├── policies/                      # Default policy definitions
│   └── patterns/                      # Injection pattern libraries
│
└── tests/
    ├── unit/
    ├── integration/
    ├── adversarial/                   # Red-team test cases
    └── fixtures/                      # Test data for each guard type
```

## Benefits

1. **Runtime Safety** — Active enforcement of content policies during every AI interaction, not just at review time
2. **Defense in Depth** — Multiple independent checks catch threats that any single guard might miss
3. **Configurability** — Policies are externalized, enabling per-application and per-tenant customization without code changes
4. **Auditability** — Complete audit trail of every guardrail decision supports compliance and incident investigation
5. **PII Protection** — Automatic detection and masking prevents sensitive data from reaching the LLM or appearing in outputs
6. **Composability** — Guards are modular and independently testable; new guards can be added without modifying existing ones

## Trade-offs

| Advantage                           | Consideration                                               |
| ----------------------------------- | ----------------------------------------------------------- |
| Multi-layer safety enforcement      | Additional latency from sequential guard evaluation         |
| Configurable per application/tenant | Policy complexity grows with the number of use cases        |
| ML-based injection detection        | Classifier requires training data and periodic retraining   |
| Groundedness verification           | NLI checks add LLM calls and latency                        |
| Fail-closed behavior                | Overly strict policies increase false positive rate         |
| Complete audit logging              | High-volume logging requires storage and retention planning |

## When to Use

✅ **Good fit for:**

- Production AI applications serving end users
- Enterprise systems handling sensitive or regulated data (healthcare, finance)
- Customer-facing chatbots and virtual assistants
- AI agents with tool access that can perform real-world actions
- Multi-tenant AI platforms requiring per-tenant safety policies
- Applications requiring compliance with content safety regulations

❌ **Not ideal for:**

- Internal development tools used only by trusted engineers
- Offline batch analysis with no user-facing output
- Simple rule-based systems with no LLM component
- Research environments where safety constraints hinder experimentation

## References

- [Azure AI Content Safety — Microsoft](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/)
- [NeMo Guardrails — NVIDIA](https://github.com/NVIDIA/NeMo-Guardrails)
- [Guardrails AI — Open Source Framework](https://github.com/guardrails-ai/guardrails)
- [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/)
- [Prompt Injection Mitigation — Microsoft AI Red Team](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/prompt-injection)
- [Responsible AI Practices — Microsoft](https://www.microsoft.com/en-us/ai/responsible-ai)
- [Lakera Guard — Prompt Injection Protection](https://www.lakera.ai/)
- [Presidio: Data Protection and PII Anonymization — Microsoft](https://github.com/microsoft/presidio)
