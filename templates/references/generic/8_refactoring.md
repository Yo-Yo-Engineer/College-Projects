# Codebase Review and Refactoring Template

#todo #agent [OPTIONAL_TAGS]

Task:
Review, refactor, and validate [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Prioritize correctness, maintainability, and readability
- Preserve existing functionality unless a change fixes a defect or improves stability

## Review

- Identify and resolve defects, fragile logic, and reliability issues
- Improve code structure, readability, and consistency
- Standardize formatting, naming, and conventions
- Remove dead code, redundant logic, unused assets, and unnecessary files
- Ensure the codebase follows clean architecture principles where practical
- Use consistent patterns and avoid overly complex or duplicated implementations

## Validation

- Test the application to confirm all features and workflows function as expected
- Verify edge cases are handled correctly and no regressions are introduced
- Run relevant lint, format, type-check, test, and build commands where available
- If validation cannot be run, clearly state why

## Constraints

- Prefer minimal, targeted changes over broad rewrites
- Use existing project patterns before introducing new abstractions
- Keep changes backward-compatible where practical

## Output

1. Issues found and changes made
2. Files and modules significantly updated
3. Tests added, updated, or identified as missing
4. Validation results and any remaining risks
5. Recommended follow-up work
