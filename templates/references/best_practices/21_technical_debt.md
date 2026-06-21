# Technical Debt Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Perform a technical debt assessment of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Identify, classify, and prioritize technical debt for remediation
- Distinguish actionable debt from acceptable trade-offs

## Focus Areas

### Code-Level Debt

- Identify duplicated logic, inconsistent patterns, and overly complex implementations
- Flag dead code, commented-out code, and obsolete TODO/FIXME/HACK markers
- Check for premature abstractions, over-engineering, and speculative generality
- Identify missing or inadequate error handling, input validation, and edge case coverage

### Architectural Debt

- Identify violations of the project's intended architecture or design patterns
- Flag inappropriate coupling between modules, layers, or services
- Check for responsibilities that have drifted into the wrong component over time
- Identify areas where the architecture has not scaled with the codebase's growth

### Test Debt

- Identify critical paths, business logic, and edge cases lacking test coverage
- Flag brittle, flaky, or slow tests that reduce confidence or developer velocity
- Check for missing integration tests at key boundaries
- Identify test code that is difficult to maintain or understand

### Dependency Debt

- Identify outdated, deprecated, or end-of-life dependencies
- Flag dependencies with known vulnerabilities or licensing concerns
- Check for unnecessary or redundant dependencies that increase surface area
- Identify version pinning issues or inconsistencies across the project

### Documentation Debt

- Identify outdated, inaccurate, or missing documentation for public APIs and key workflows
- Check for undocumented configuration, environment setup, or deployment steps
- Flag missing or stale architecture decision records
- Verify README and onboarding documentation reflects the current state

### Infrastructure and Tooling Debt

- Identify manual processes that should be automated (builds, deploys, checks)
- Check for outdated CI/CD configurations, scripts, or tooling
- Flag missing linting, formatting, or static analysis enforcement
- Identify inconsistencies between development, staging, and production environments

## Classification

Categorize each debt item by type and urgency:

| Type       | Description                                                                     |
| ---------- | ------------------------------------------------------------------------------- |
| Deliberate | Conscious trade-off made for speed or scope — document and schedule remediation |
| Accidental | Unintentional debt from evolving requirements or knowledge gaps                 |
| Bit Rot    | Degradation over time from dependency updates, platform changes, or neglect     |
| Prudent    | Acceptable short-term trade-off with a clear path to resolution                 |
| Reckless   | Debt that introduces active risk and requires immediate attention               |

## Prioritization Criteria

Rank debt items for remediation using these factors:

1. **Risk** — Does it cause bugs, security vulnerabilities, or data integrity issues?
2. **Velocity impact** — Does it slow down development, onboarding, or debugging?
3. **Blast radius** — How many features, teams, or users are affected?
4. **Remediation cost** — What is the effort to fix relative to the benefit gained?
5. **Trend** — Is the debt growing, stable, or naturally resolving?

## Constraints

- Assess and classify only — apply fixes only when they are low-risk and clearly justified
- Prefer incremental remediation plans over large-scale rewrites
- Respect existing project timelines and priorities when recommending remediation order
- Distinguish between debt that needs fixing and trade-offs that are acceptable

## Output

1. Technical debt inventory, categorized by area (code, architecture, test, dependency, documentation, infrastructure)
2. Each item classified by type (deliberate / accidental / bit rot / prudent / reckless)
3. Prioritized remediation plan with estimated effort (small / medium / large)
4. Quick wins — low-effort, high-impact items that can be addressed immediately
5. Strategic recommendations for systemic debt reduction
