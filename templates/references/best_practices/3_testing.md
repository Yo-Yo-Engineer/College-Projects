# Testing Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the testing practices for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Improve test quality, coverage, and maintainability
- Focus tests on meaningful behavior, edge cases, and critical paths

## Focus Areas

### Test Pyramid and Coverage Strategy

- Verify a healthy distribution: many unit tests, fewer integration tests, minimal end-to-end tests
- Identify untested critical paths, business logic, and error handling
- Check for coverage gaps in boundary conditions, edge cases, and failure scenarios
- Ensure coverage metrics are meaningful — high coverage with weak assertions is not valuable

### Test Quality and Structure

- Verify tests follow the Arrange-Act-Assert (AAA) or Given-When-Then pattern
- Ensure each test has a single clear purpose and tests one behavior
- Check for meaningful, descriptive test names that document expected behavior
- Remove or refactor tests that are brittle, flaky, slow, or provide no value
- Verify assertions are specific and meaningful — not just "no error was thrown"

### Test Isolation and Determinism

- Ensure tests are independent and can run in any order
- Check for shared mutable state, test coupling, and order dependencies
- Verify proper setup and teardown — no leaked state between tests
- Ensure tests are deterministic — no reliance on timing, network, or non-deterministic data
- Check that test data is self-contained and does not depend on external state

### Mocking and Test Doubles

- Verify mocks, stubs, and fakes are used appropriately — not excessively
- Check that mocks match the real interface and behavior
- Ensure integration points are tested with realistic doubles or real implementations
- Avoid mocking what you don't own without an adapter/wrapper layer

### Error and Edge Case Testing

- Verify negative cases, invalid inputs, and boundary conditions are tested
- Check for error handling tests: exceptions, timeouts, retries, partial failures
- Ensure empty states, null/undefined values, and extreme values are covered
- Test concurrent and race-condition-prone code paths where applicable

### Test Maintainability

- Identify test code duplication and extract shared fixtures or helpers
- Verify test fixtures are minimal and focused — avoid overly broad shared setup
- Check for hard-coded values that should be constants or builders/factories
- Ensure test utilities and helpers are well-organized and discoverable

### CI and Automation

- Verify tests run in CI and failures block merging
- Check for test execution speed — identify slow tests that could be parallelized or optimized
- Ensure test configuration is consistent across local and CI environments
- Verify flaky test detection and quarantine processes exist or are recommended

## Principles

- Tests are documentation — they should clearly express expected behavior
- Prefer testing behavior over testing implementation details
- A failing test should clearly indicate what went wrong and where
- Test code deserves the same quality standards as production code
- Delete tests that do not add confidence or document behavior

## Constraints

- Preserve existing test coverage — replace removed tests with improved alternatives
- Ensure new tests add unique coverage or value beyond existing tests
- Prefer existing test frameworks and conventions used in the project
- Maintain backward compatibility with existing test infrastructure

## Output

1. Coverage gaps and testing issues identified
2. Tests added, updated, or removed with justification
3. Test quality improvements (naming, structure, assertions)
4. Recommendations for test infrastructure or process improvements
5. Any areas where tests could not be added and why
