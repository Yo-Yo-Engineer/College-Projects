# Documentation Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve documentation for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Improve documentation quality, accuracy, and discoverability
- Focus on documentation that helps developers understand, use, and maintain the system

## Focus Areas

### README and Project Documentation

- Verify the project README includes: purpose, getting started, prerequisites, installation, usage, and contributing
- Ensure the README is current and matches the actual state of the project
- Check for a clear project structure overview or pointer to architecture docs
- Verify environment setup instructions work for a new contributor
- Ensure license, security policy, and code of conduct are present where appropriate

### Architecture and Design Documentation

- Verify high-level architecture is documented (system context, component diagrams)
- Check for Architecture Decision Records (ADRs) for significant technical decisions
- Ensure diagrams use a standard notation (C4, Mermaid, PlantUML) and are version-controlled
- Verify domain model or data model documentation exists for complex systems
- Check that integration points, protocols, and data flows are documented

### Code-Level Documentation

- Verify public APIs, interfaces, and exported functions have clear docstrings or comments
- Ensure comments explain why, not what — the code should be self-explanatory for the what
- Check for misleading, outdated, or redundant comments that should be removed
- Verify complex algorithms, business rules, and non-obvious decisions are documented inline
- Ensure type definitions and interfaces serve as documentation through clear naming

### API Documentation

- Verify API documentation exists (OpenAPI/Swagger, GraphQL schema, or equivalent)
- Ensure request/response examples are provided for all endpoints
- Check that error codes and their meanings are documented
- Verify authentication and authorization requirements are clearly stated
- Ensure API documentation is auto-generated from source where possible to prevent drift

### Changelog and Versioning

- Verify a changelog exists and follows a consistent format (Keep a Changelog or equivalent)
- Ensure breaking changes, new features, and bug fixes are documented per release
- Check that versioning follows Semantic Versioning (SemVer) or the project's stated convention

### Operational Documentation

- Verify runbooks or operational guides exist for common tasks (deployment, rollback, incident response)
- Check that environment variables and configuration are documented with purpose, type, and defaults
- Ensure monitoring, alerting, and debugging guidance is available
- Verify disaster recovery and backup procedures are documented where applicable

### Documentation Maintenance

- Identify stale, inaccurate, or orphaned documentation
- Verify documentation is close to the code it describes (co-location principle)
- Check for broken links, missing references, and incorrect paths
- Ensure documentation build/generation is part of CI where applicable

## Principles

- Documentation should reduce the time for a new developer to become productive
- Prefer living documentation that stays in sync with code over external wikis that drift
- Document decisions, constraints, and trade-offs — not just the final implementation
- Keep documentation minimal but sufficient — over-documentation creates maintenance burden

## Constraints

- Focus documentation on non-trivial, non-self-explanatory code and decisions
- Prefer updating existing documentation over creating new files
- Follow the project's existing documentation conventions and tools
- Ensure documentation changes are consistent with code changes in the same review

## Output

1. Documentation gaps and issues identified
2. Documentation added or updated with justification
3. Stale or inaccurate documentation corrected or removed
4. Recommendations for documentation tooling or process improvements
5. Suggested documentation structure for undocumented areas
