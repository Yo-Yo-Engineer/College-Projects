# Code Quality Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Perform a code quality review of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Prioritize clarity, maintainability, and correctness over style preferences
- Avoid cosmetic-only changes unless they improve consistency

## Focus Areas

### SOLID Principles (where applicable to the language and paradigm)

- Single Responsibility: each module, class, and function has one clear purpose
- Open/Closed: code is open for extension, closed for modification where appropriate
- Liskov Substitution: subtypes are substitutable for their base types without surprises
- Interface Segregation: interfaces are focused and cohesive, not bloated
- Dependency Inversion: high-level modules depend on abstractions, not concrete implementations

### Simplicity and Clarity

- KISS: prefer the simplest solution that correctly solves the problem
- YAGNI: remove speculative features, unused abstractions, and dead configuration
- Avoid cleverness — code should be readable by someone unfamiliar with the codebase
- Functions and methods should operate at a single level of abstraction
- Control flow should be predictable and easy to follow

### Duplication and Abstraction

- DRY: eliminate duplication of knowledge and logic where it improves maintainability
- AHA (Avoid Hasty Abstractions): do not abstract prematurely — wait for stable patterns
- Prefer duplication over the wrong abstraction when the correct pattern is not yet clear
- Remove abstractions that add indirection without reducing complexity

### Naming and Readability

- Variables, functions, classes, and modules have clear, descriptive, intention-revealing names
- Naming is consistent with project conventions and domain language
- Abbreviations are avoided unless universally understood in context
- Boolean variables and functions clearly express their meaning (e.g., isValid, hasPermission)

### Complexity and Structure

- Reduce cyclomatic complexity — simplify deep nesting, long conditionals, and switch/case chains
- Keep functions small and focused — extract when a function does more than one logical thing
- Ensure high cohesion within modules and low coupling between modules
- Apply separation of concerns across layers, services, and responsibilities
- Prefer composition over inheritance unless inheritance is clearly justified

### Error Handling

- Error handling is explicit, consistent, and appropriate for the context
- Errors are not silently swallowed — handle, propagate, or document the decision
- Fail fast on invalid state rather than propagating corrupt data
- Use typed or structured errors where the language supports them

### Code Smells

- Identify and address: long methods, large classes, feature envy, data clumps, primitive obsession, shotgun surgery, divergent change, inappropriate intimacy, and message chains
- Remove dead code, commented-out code, and TODO/FIXME items that are no longer relevant
- Consolidate inconsistent patterns within the same codebase

## Decision Rules

When trade-offs arise, prioritize in this order:

1. Correctness and reliability
2. Simplicity and clarity
3. Maintainability and testability
4. Reuse and abstraction
5. Performance (unless on a proven hot path)

## Constraints

- Preserve existing behavior unless a change fixes a defect or is clearly justified
- Prefer minimal, targeted changes over broad rewrites
- Follow existing project conventions unless they conflict with quality principles
- Introduce new patterns, frameworks, or abstractions only when solving a demonstrated problem

## Output

1. Code quality issues identified, categorized by severity
2. Changes made or proposed with justification
3. Patterns consolidated or simplified
4. Remaining technical debt and recommended follow-up
5. Any assumptions or decisions that require team alignment
