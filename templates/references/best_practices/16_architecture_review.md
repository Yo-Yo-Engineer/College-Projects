# Architecture Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Perform an architecture-focused review of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[ARCHITECTURE_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Evaluate alignment with the project's chosen architectural pattern(s)
- Keep changes within the architectural domain — defer implementation-level fixes to other reviews

## Focus Areas

### Architectural Pattern Compliance

- Verify the codebase adheres to its declared architecture (clean, hexagonal, onion, DDD, microservices, serverless, event-driven, etc.)
- Check that layer boundaries are respected — no leaking of concerns across layers
- Ensure dependency direction follows the architecture's rules (e.g., inward-only in clean/onion)
- Identify violations where infrastructure or framework details bleed into domain or business logic

### Component Boundaries and Cohesion

- Verify each module, service, or bounded context has a single, well-defined responsibility
- Ensure high cohesion within components — related logic belongs together
- Identify components that have grown beyond their original scope and need splitting
- Check that public interfaces are minimal and intentional

### Coupling and Dependency Management

- Identify tight coupling between components that should be independent
- Verify communication between components uses appropriate abstractions (interfaces, events, contracts)
- Check for circular dependencies between modules, packages, or services
- Ensure external dependencies are isolated behind adapters or anti-corruption layers

### Cross-Cutting Concerns

- Verify consistent handling of logging, monitoring, authentication, authorization, and error handling across boundaries
- Ensure cross-cutting concerns are implemented through shared infrastructure, not duplicated per component
- Check that middleware, interceptors, or decorators are used appropriately for shared behavior

### Data Flow and Integration

- Verify data flows through the architecture in a predictable, traceable path
- Check that integration points (APIs, message queues, event buses) have clear contracts
- Ensure data transformations happen at appropriate boundaries (DTOs, mappers, serializers)
- Identify unintended tight coupling through shared database schemas or direct data access

### Scalability and Extensibility

- Verify the architecture supports expected growth patterns without structural changes
- Check that new features can be added without modifying existing core components (open/closed)
- Ensure the architecture allows independent deployment, testing, and scaling of components where required
- Identify architectural bottlenecks that limit horizontal or vertical scaling

### Architecture Decision Records

- Check that significant architectural decisions are documented with context, rationale, and trade-offs
- Verify ADRs exist for technology choices, pattern selections, and deviation from conventions
- Ensure superseded decisions are marked and linked to their replacements

## Reference Standards

- Domain-Driven Design (bounded contexts, ubiquitous language, aggregates)
- Clean Architecture (dependency rule, use cases, entities)
- SOLID Principles (applied at the architectural level)
- Twelve-Factor App methodology
- Architecture fitness functions and evolutionary architecture

## Constraints

- Preserve existing architectural boundaries unless a change is clearly justified
- Prefer incremental structural improvements over full architectural rewrites
- Introduce new patterns or layers only when solving a demonstrated problem
- Maintain backward compatibility with existing integrations and APIs

## Output

1. Architectural issues identified, categorized by impact (structural / design / convention)
2. Pattern violations found with specific locations and recommended corrections
3. Dependency and coupling analysis with visual or descriptive mapping where helpful
4. Changes made or proposed with justification
5. Recommendations for follow-up (ADR creation, decomposition, migration planning)
