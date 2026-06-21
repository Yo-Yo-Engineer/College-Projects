# Bug Investigation Template

#todo #agent [OPTIONAL_TAGS]

Task:
Investigate, diagnose, and fix [BUG_DESCRIPTION] in [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Prioritize root cause identification over surface-level fixes
- Keep the fix minimal and targeted — avoid unrelated changes

## Investigation Workflow

### 1. Reproduce

- Confirm the bug is reproducible with a clear set of steps
- Document the expected behavior versus the actual behavior
- Identify the environment, inputs, and conditions that trigger the issue
- Note whether the bug is deterministic or intermittent

### 2. Isolate

- Narrow the scope to the specific module, function, or component responsible
- Trace the data flow and control flow through the affected path
- Identify the earliest point where behavior diverges from expectation
- Use logs, breakpoints, or instrumentation to pinpoint the source

### 3. Root Cause Analysis

- Identify the underlying cause — distinguish root cause from symptoms
- Determine when the bug was introduced (regression analysis if possible)
- Check whether the same root cause affects other code paths or features
- Document any contributing factors (race conditions, edge cases, incorrect assumptions)

### 4. Fix

- Apply a targeted fix that addresses the root cause, not just the symptom
- Ensure the fix preserves existing behavior for all unaffected paths
- Verify the fix handles related edge cases and boundary conditions
- Keep the change minimal — avoid refactoring or cleanup in the same change

### 5. Regression Testing

- Add or update tests that reproduce the original bug and verify the fix
- Cover the specific input, state, or condition that triggered the issue
- Verify related functionality is not broken by the change
- Run the existing test suite to confirm no regressions

### 6. Document

- Summarize the root cause and the fix clearly
- Note any assumptions, workarounds, or limitations of the fix
- Identify follow-up work if a broader fix or refactor is warranted

## Constraints

- Prefer the smallest possible change that correctly fixes the issue
- Preserve existing functionality and behavior outside the bug scope
- Avoid bundling unrelated improvements with the bug fix
- Use existing patterns and conventions in the codebase

## Output

1. Root cause identified with clear explanation
2. Steps to reproduce documented
3. Fix applied with justification
4. Tests added or updated to prevent recurrence
5. Related areas at risk and recommended follow-up
