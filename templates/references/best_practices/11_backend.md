# Backend Development Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the backend implementation of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review service layer, data access, messaging, and API implementation
- Use architectural patterns already adopted by the project

## Focus Areas

### Architecture and Layering

- Verify clear separation of concerns across layers (API/controller, service/application, domain, infrastructure)
- Ensure domain logic is not leaking into controllers, data access, or infrastructure code
- Check that dependencies point inward — infrastructure depends on domain, not the reverse
- Verify the application follows its stated architecture (clean, hexagonal, onion, or layered)
- Ensure cross-cutting concerns (logging, auth, validation) are handled consistently via middleware or decorators

### Dependency Injection and Inversion of Control

- Verify services depend on abstractions (interfaces/contracts), not concrete implementations
- Check service lifetime management: transient, scoped, and singleton registrations are appropriate
- Ensure no service locator anti-pattern — dependencies are injected, not resolved at runtime
- Verify circular dependencies do not exist between services
- Check that DI container registration is organized and discoverable

### Service Layer and Business Logic

- Verify business rules are encapsulated in the service or domain layer — not scattered across controllers or repositories
- Ensure service methods have a single responsibility and clear input/output contracts
- Check for proper use of domain models vs. DTOs vs. persistence entities — no leaking of internal models to external APIs
- Verify validation happens at system boundaries (API input) and at the domain level (business rules)
- Ensure idempotency for operations that may be retried

### Data Access and Repository Patterns

- Verify data access is abstracted behind repositories or equivalent patterns
- Check that queries are efficient: no N+1, no unbounded result sets, proper use of projections
- Ensure transactions are scoped correctly — short, focused, and at the appropriate boundary
- Verify Unit of Work pattern (or equivalent) is used correctly for multi-entity operations
- Check that connection management follows pooling best practices

### Messaging and Event-Driven Patterns

- Verify message consumers are idempotent — processing the same message twice produces the same result
- Ensure dead-letter queue (DLQ) handling is configured for failed messages
- Check for poison message protection — messages that consistently fail are moved aside, not retried infinitely
- Verify message schemas are versioned and backward-compatible
- Ensure message ordering guarantees match business requirements
- Check for proper correlation ID propagation through message flows
- Verify publish/subscribe patterns use appropriate topic/queue separation

### Background Jobs and Scheduled Tasks

- Verify background jobs are idempotent and safe to retry on failure
- Ensure long-running jobs have progress tracking, cancellation support, and timeouts
- Check for proper error handling and alerting on job failures
- Verify job scheduling does not create overlapping executions unless intended
- Ensure background work does not block request processing

### Authentication and Authorization

- Verify authentication is enforced on all endpoints that require it
- Ensure authorization checks happen at the service layer — not only at the controller level
- Check for proper role-based or policy-based access control
- Verify token validation, expiry handling, and refresh flows are correct
- Ensure multi-tenancy isolation is enforced at the data access layer if applicable

### Error Handling and Resilience

- Verify a consistent error-handling strategy across all layers
- Ensure external service calls have timeouts, retries with backoff, and circuit breakers
- Check that domain exceptions are translated to appropriate HTTP responses at the API boundary
- Verify transient fault handling for database, cache, and messaging infrastructure
- Ensure partial failures in distributed operations are handled (compensating transactions, saga pattern)

### Configuration and Secrets

- Verify all configuration is externalized — not hardcoded in source code
- Ensure secrets (connection strings, API keys, passwords) are loaded from secret managers or environment variables
- Check that no credentials appear in source code, configuration files, logs, or error messages
- Verify configuration is validated at startup — fail fast on missing or invalid values

## Reference Standards

- Clean Architecture (Robert C. Martin)
- Domain-Driven Design tactical patterns (Evans)
- CQRS and Event Sourcing (where applicable)
- The Twelve-Factor App (III: Config, IV: Backing services, VIII: Concurrency, XI: Logs)
- OWASP API Security Top 10

## Constraints

- Preserve existing architecture and patterns unless a change is clearly justified
- Use existing frameworks and architectural patterns; introduce new ones only to solve a demonstrated problem
- Prefer incremental improvement over large-scale refactoring
- Ensure all changes are backward-compatible with existing clients and integrations

## Output

1. Architecture and design issues identified
2. Changes made or proposed with justification
3. Service layer and data access improvements
4. Messaging and resilience patterns applied or recommended
5. Security and configuration issues resolved
6. Recommendations for testing, monitoring, or follow-up work
