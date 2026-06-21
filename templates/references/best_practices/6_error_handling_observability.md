# Error Handling and Observability Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve error handling, logging, and observability for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Improve resilience, diagnosability, and operational visibility
- Justify any new observability framework additions before introducing them

## Focus Areas

### Error Handling Patterns

- Verify errors are handled explicitly — not silently swallowed or caught with empty handlers
- Ensure error handling follows a consistent pattern across the codebase
- Check that errors fail fast on invalid state rather than propagating corrupt data
- Verify proper use of typed/structured errors with meaningful error codes and messages
- Ensure error boundaries exist at appropriate layers (controller, service, infrastructure)
- Check that cleanup and resource release happen correctly on error paths

### Resilience and Fault Tolerance

- Verify retry logic uses exponential backoff with jitter — not fixed intervals or infinite retries
- Check for circuit breaker patterns on external service calls
- Ensure timeouts are configured for all network calls, database queries, and external service interactions
- Verify graceful degradation — partial failures should not crash the entire system
- Check for bulkhead patterns to isolate failures in critical subsystems
- Ensure idempotency is maintained through retries

### Structured Logging

- Verify logs use structured format (JSON or key-value) — not unstructured string concatenation
- Ensure consistent log levels: DEBUG, INFO, WARN, ERROR, FATAL — used appropriately
- Check that log messages include relevant context: operation, entity ID, user context, timing
- Verify sensitive data is never logged: passwords, tokens, PII, secrets, full request bodies with sensitive fields
- Ensure log volume is manageable — avoid excessive debug logging in production paths
- Check for consistent log formatting and field naming across services

### Correlation and Tracing

- Verify request/correlation IDs are generated at entry points and propagated through all layers
- Ensure correlation IDs appear in logs, error responses, and downstream service calls
- Check for distributed tracing instrumentation (OpenTelemetry or equivalent) where applicable
- Verify trace context propagation across service boundaries

### Health Checks and Monitoring

- Verify health check endpoints exist: liveness (is the process running) and readiness (can it serve traffic)
- Ensure health checks validate critical dependencies (database, cache, external services)
- Check for key metrics: request rate, error rate, latency (RED method) and utilization, saturation, errors (USE method)
- Verify alerting thresholds and SLI/SLO definitions exist or are recommended
- Ensure dashboard or monitoring configuration reflects critical business and operational metrics

### Error Reporting and Alerting

- Verify critical errors trigger alerts — not just log entries
- Ensure error aggregation and deduplication to prevent alert fatigue
- Check that error context is sufficient for debugging without reproducing the issue
- Verify errors are categorized: transient vs. permanent, client vs. server, expected vs. unexpected

## Reference Standards

- OpenTelemetry specification (traces, metrics, logs)
- RED method (Request rate, Error rate, Duration)
- USE method (Utilization, Saturation, Errors)
- The Twelve-Factor App (XI: Logs, XII: Admin processes)
- Site Reliability Engineering (SLI/SLO/SLA concepts)

## Constraints

- Preserve existing error handling behavior unless it is clearly incorrect
- Ensure instrumentation additions maintain acceptable performance levels
- Prefer existing logging and monitoring frameworks used in the project
- Ensure observability does not leak sensitive data

## Output

1. Error handling gaps and observability issues identified
2. Changes made or proposed with justification
3. Logging improvements: structure, levels, context, sensitive data removal
4. Resilience patterns added or recommended
5. Monitoring and alerting recommendations
6. Follow-up actions: dashboards, runbooks, SLO definitions
