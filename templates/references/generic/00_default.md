# Default Review Template

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Keep changes within scope unless required for correctness
- Prefer focused fixes over broad rewrites
- Preserve file locations and names unless clearly justified

Requirements:

- Identify and fix issues
- Refactor where it improves maintainability
- Improve formatting, naming, and consistency
- Remove dead code, duplication, and unnecessary files
- Preserve intended behavior unless a change is justified
- Prefer backward-compatible changes where practical
- Use only commands, files, tools, workflows, and architecture supported by the repository

Validation:

- Run relevant lint, format, type-check, test, and build commands where available
- If validation cannot be run, clearly state why
- Only claim changes are verified when validation was actually performed
- Add or update tests when behavior changes or gaps are found

Security and reliability:

- Check for hardcoded secrets, unsafe configuration, and excessive permissions
- Verify input validation, error handling, and safe defaults where relevant
- Preserve secure patterns and avoid unnecessary dependencies or performance regressions

Documentation and consistency:

- Update related documentation, comments, examples, and configuration when needed
- Align changes with existing architecture and repository conventions
- Keep CI, workflows, and developer experience consistent with the changes

Working style:

- Prefer clear, minimal, maintainable changes
- Use existing project patterns before introducing new abstractions
- Explain important changes, assumptions, and risks
- Summarize findings, changed files, and recommended follow-up validation
