# DevOps and CI/CD Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve DevOps practices and CI/CD pipelines for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Improve build, test, deploy reliability and developer experience
- Confirm with the user before modifying production infrastructure

## Focus Areas

### CI Pipeline Design

- Verify the pipeline includes stages: lint, format, type-check, unit test, build, integration test, security scan
- Ensure pipeline stages fail fast — cheapest checks run first
- Check for parallelization of independent stages to reduce total pipeline duration
- Verify caching is used for dependencies, build artifacts, and Docker layers
- Ensure pipeline configuration is version-controlled alongside application code
- Check that pipeline triggers are appropriate: push, PR, merge, tag, schedule

### Build and Artifact Management

- Verify builds are reproducible — deterministic dependency resolution, lockfiles, pinned versions
- Ensure build artifacts are tagged with version, commit SHA, and build metadata
- Check for multi-stage builds to minimize artifact size and attack surface
- Verify build outputs are stored in a registry or artifact store, not rebuilt for each environment
- Ensure build configuration is environment-agnostic — environment-specific values injected at runtime

### Testing in CI

- Verify all test types run in CI and failures block merging
- Ensure test environments are isolated and ephemeral — no shared mutable state across runs
- Check for flaky test detection, reporting, and quarantine
- Verify test result reporting with clear failure attribution
- Ensure integration and E2E tests use realistic fixtures or contracts, not mocked everything

### Security Scanning

- Verify static analysis (SAST) runs on every PR or push
- Check for dependency vulnerability scanning (SCA) with automated alerts
- Ensure secret scanning is enabled — no credentials in source code or CI logs
- Verify container image scanning for known vulnerabilities
- Check that security scan failures block deployment for critical findings

### Deployment Strategy

- Verify deployments are automated and repeatable — no manual steps in the deployment path
- Ensure a deployment strategy exists: rolling, blue/green, canary, or equivalent
- Check for zero-downtime deployment capability with database migration safety
- Verify rollback procedures are documented and tested
- Ensure environment promotion follows a consistent path: dev → staging → production
- Check for feature flags to decouple deployments from feature releases

### Configuration and Environment Management

- Verify configuration follows the Twelve-Factor App: config in environment, not in code
- Ensure secrets are managed through secret managers — not environment files committed to source
- Check for configuration validation at startup — fail fast on missing or invalid config
- Verify environment parity — dev, staging, and production are structurally identical
- Ensure infrastructure is defined as code (Terraform, Pulumi, CloudFormation, or equivalent)

### Container Best Practices (if applicable)

- Verify Dockerfiles use specific base image tags — not :latest
- Ensure multi-stage builds separate build dependencies from runtime
- Check that containers run as non-root users
- Verify .dockerignore excludes unnecessary files (node_modules, .git, tests, docs)
- Ensure container images are minimal — no build tools, dev dependencies, or debug utilities in production
- Check for health check instructions in container definitions

### Developer Experience

- Verify local development setup is documented and works from a clean checkout
- Ensure development containers or environment scripts provide consistent environments
- Check that CI feedback is fast — aim for PR checks completing within a reasonable timeframe
- Verify PR templates guide contributors through quality checks
- Ensure branch protection rules enforce required checks before merging

### Monitoring and Feedback Loops

- Verify deployment events are tracked and correlated with metrics
- Check for automated canary analysis or smoke tests post-deployment
- Ensure CI/CD metrics are tracked: build time, failure rate, deployment frequency, lead time
- Verify incident response procedures reference deployment history and rollback capability

## Reference Standards

- The Twelve-Factor App
- DORA metrics (Deployment Frequency, Lead Time, Change Failure Rate, Time to Restore)
- CIS Benchmarks for container and cloud security (where applicable)
- GitOps principles (declarative, versioned, automated, observable)

## Constraints

- Obtain explicit approval before modifying production infrastructure or pipelines
- Preserve existing deployment processes unless a specific improvement is justified
- Prefer incremental pipeline improvements over complete rewrites
- Ensure CI/CD changes are backward compatible with existing workflows

## Output

1. CI/CD issues and improvement opportunities identified
2. Pipeline changes made or proposed with justification
3. Security scanning gaps and recommendations
4. Deployment reliability improvements
5. Developer experience enhancements
6. Recommended DORA metrics baseline and targets
7. Follow-up actions for infrastructure, monitoring, or process changes
