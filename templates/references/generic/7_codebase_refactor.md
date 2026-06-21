# Codebase Refactor and Validation Template

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the codebase in [SCOPE].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

## Primary Objective

Improve correctness, reliability, readability, maintainability, and testability while preserving intended behavior unless a change is required to fix a defect, reduce risk, or improve operational robustness.

## Goals

- Fix bugs, fragile logic, and reliability issues
- Refactor for readability, maintainability, and testability
- Standardize structure, naming, formatting, and conventions
- Remove dead code, duplication, unused dependencies/imports, and obsolete files
- Preserve intended behavior unless change is justified and documented

## Engineering Principles

Apply the following principles where they improve clarity, correctness, and maintainability:

- KISS: Prefer the simplest solution that correctly solves the problem
- YAGNI: Remove speculative features, abstractions, hooks, or configuration that are not currently needed
- DRY: Remove duplication of knowledge and logic where doing so improves maintainability
- AHA: Avoid hasty abstractions; wait for stable patterns before generalizing
- Prefer duplication over the wrong abstraction when the correct abstraction is not yet clear
- Separation of Concerns: Keep responsibilities well-bounded across modules, functions, classes, and layers
- Minimize coupling and unnecessary knowledge of internals; apply the Law of Demeter where relevant
- Apply SOLID principles when appropriate to the language, architecture, and paradigm, especially in object-oriented or interface-driven codebases
- Prefer composition over inheritance unless inheritance is clearly justified
- Favor explicitness, predictable control flow, and safe defaults over hidden behavior and cleverness

## Decision Rules

When principles conflict, prioritize in this order:

1. Correctness and reliability
2. Simplicity and clarity
3. Preservation of existing behavior and compatibility
4. Maintainability and testability
5. Reuse and abstraction

Use abstraction only when it is clearly justified by repeated, stable patterns.

## Standards

- Use small, cohesive functions/modules with clear responsibilities
- Keep code easy to read, reason about, test, and modify
- Follow existing project conventions when they are reasonable and internally consistent
- Standardize inconsistent patterns where this reduces maintenance cost
- Write comments/docstrings that explain intent, constraints, assumptions, tradeoffs, or why a decision exists
- Use explicit error handling, input validation, and safe failure behavior
- Keep tests maintainable and aligned with real behavior
- Update related documentation when behavior, setup, configuration, or usage changes

## Required Work

1. Audit for bugs, code smells, inconsistent patterns, security risks, unnecessary complexity, and maintainability issues
2. Refactor safely and incrementally without unnecessary rewrites
3. Standardize conventions across the project
4. Remove unused, redundant, obsolete, or speculative code and files
5. Simplify overly abstract or over-engineered code where a clearer design is available
6. Add or update tests for critical paths, bug fixes, regressions, and edge cases
7. Validate major workflows, runtime behavior, and failure paths
8. Ensure the project builds, runs, and passes checks/tests without unresolved errors

## Constraints

- Preserve business logic unless a change fixes a defect or is clearly justified
- Prefer targeted, minimal-risk changes over broad rewrites
- Use existing frameworks and architectural patterns; introduce new ones only to solve a demonstrated problem
- Retain useful abstractions; remove only those that add indirection without reducing complexity

## Output

1. Issues found
2. Key changes made
3. Files/modules significantly updated
4. Tests added or updated
5. Validation performed
6. Remaining risks, tradeoffs, or follow-up recommendations
7. Any intentionally deferred improvements and why they were deferred
