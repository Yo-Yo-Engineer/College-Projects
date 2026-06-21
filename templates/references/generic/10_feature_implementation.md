# Feature Implementation Template

#todo #agent [OPTIONAL_TAGS]

Task:
Implement [FEATURE_DESCRIPTION] in [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[ARCHITECTURE_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Deliver a complete, working implementation that meets the stated requirements
- Follow existing project patterns and conventions

## Implementation Workflow

### 1. Requirements Analysis

- Clarify the feature's purpose, expected behavior, and acceptance criteria
- Identify affected components, modules, and integration points
- Determine data model changes, API surface changes, and UI changes required
- Note any dependencies on external services, libraries, or infrastructure

### 2. Design Decisions

- Choose the approach that best fits the existing architecture and patterns
- Identify trade-offs between simplicity, extensibility, and performance
- Document any significant design decisions with rationale
- Consider edge cases, error scenarios, and failure modes upfront

### 3. Implementation Plan

- Break the feature into incremental, verifiable steps
- Order steps to maintain a working codebase at each stage
- Identify shared code, utilities, or abstractions to reuse
- Plan for feature flags or gradual rollout if applicable

### 4. Implementation

- Follow existing project structure, naming conventions, and patterns
- Keep each change focused and reviewable
- Handle errors explicitly and provide meaningful feedback
- Validate inputs at system boundaries
- Ensure the feature integrates cleanly with existing functionality

### 5. Testing

- Write unit tests for core logic, edge cases, and error paths
- Add integration tests for component interactions and data flow
- Verify end-to-end behavior matches acceptance criteria
- Test failure scenarios and degraded states
- Run the existing test suite to confirm no regressions

### 6. Documentation

- Update API documentation, README, or usage guides as needed
- Add inline comments only where intent or rationale is not self-evident
- Document configuration changes, environment variables, or migration steps
- Update changelog or release notes if applicable

## Constraints

- Prefer existing patterns and abstractions over introducing new ones
- Keep the implementation as simple as possible while meeting requirements
- Preserve backward compatibility unless breaking changes are explicitly accepted
- Introduce new dependencies only when clearly justified

## Output

1. Implementation summary with key design decisions
2. Files created or modified with descriptions of changes
3. Tests added with coverage of core paths, edge cases, and error scenarios
4. Documentation updated or created
5. Known limitations and recommended follow-up work
