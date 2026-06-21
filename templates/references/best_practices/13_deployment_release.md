# Deployment and Release Management Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the deployment and release process for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review deployment pipelines, environment promotion, and release procedures
- Confirm with the user before modifying production infrastructure

## Focus Areas

### Environment Promotion Strategy

- Verify a clear promotion path exists: dev → UAT/QA → staging → production
- Ensure each environment has a defined purpose and gate criteria before promotion
- Check that promotion is automated where possible — manual gates only where human approval is required
- Verify artifacts are built once and promoted across environments — not rebuilt per environment
- Ensure environment-specific configuration is injected at deployment time, not baked into artifacts

### Shared Infrastructure Management

- Verify shared resources (message buses, caches, databases) are properly isolated between environments or have clear sharing boundaries
- Ensure shared infrastructure has appropriate access controls per environment
- Check that shared resources have monitoring and capacity planning for multi-environment usage
- Verify naming conventions clearly identify which environments share infrastructure
- Ensure shared infrastructure failures have defined blast radius and fallback procedures

### Deployment Automation

- Verify deployments are fully automated and repeatable — no manual SSH, copy-paste, or ad-hoc scripts
- Ensure deployment scripts are version-controlled alongside application code
- Check for infrastructure-as-code for environment provisioning (Terraform, Pulumi, ARM/Bicep, CloudFormation)
- Verify deployment commands are idempotent — running twice produces the same result
- Ensure deployment logs capture what was deployed, when, by whom, and from which commit/artifact

### Database Migration Safety

- Verify database migrations run before or during deployment with proper ordering
- Ensure migrations are backward-compatible — the previous application version can still run during migration
- Check for safe migration patterns: additive-only changes, multi-phase migrations for breaking changes
- Verify migration rollback procedures exist and have been tested
- Ensure data migrations for large tables use batched operations — not full-table locks

### Deployment Verification

- Verify smoke tests run automatically after each deployment
- Ensure health check endpoints are validated post-deployment
- Check for canary or progressive rollout capability for production deployments
- Verify deployment monitoring detects regressions in error rate, latency, and availability
- Ensure automated rollback triggers exist for critical metric degradation

### Rollback Procedures

- Verify rollback procedures are documented, tested, and can be executed quickly
- Ensure rollback includes both application and database components
- Check that forward-fix is preferred when safe, with rollback as the safety net
- Verify rollback does not cause data loss or corruption
- Ensure rollback procedures account for message queue consumer compatibility

### Release Management

- Verify releases follow semantic versioning or the project's stated convention
- Ensure changelogs are maintained and include migration/upgrade notes for breaking changes
- Check for feature flags to decouple deployment from feature release
- Verify release tagging and artifact labeling are consistent and traceable
- Ensure release documentation includes deployment prerequisites and verification steps

### Secrets and Credentials in Deployment

- Verify no secrets, connection strings, or credentials appear in deployment scripts, CI logs, or prompts
- Ensure secrets are injected at runtime from secret managers (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault)
- Check that deployment service accounts use least-privilege permissions
- Verify secrets rotation does not require redeployment
- Ensure credential exposure in error messages, logs, or monitoring dashboards is prevented

### Post-Deployment Validation

- Verify end-to-end tests run against the deployed environment after deployment
- Ensure data integrity checks validate that the deployment did not corrupt existing data
- Check for integration validation with dependent services (message queues, external APIs, databases)
- Verify performance baselines are compared post-deployment
- Ensure user-facing functionality is validated through automated or manual acceptance tests

## Reference Standards

- The Twelve-Factor App (V: Build/release/run, X: Dev/prod parity)
- DORA metrics (Deployment Frequency, Lead Time, Change Failure Rate, Time to Restore)
- GitOps principles (declarative, versioned, automated, self-healing)
- Infrastructure as Code best practices

## Constraints

- Ensure all gate criteria pass before promoting to production
- Keep secrets and credentials out of deployment configurations and prompts
- Preserve backward compatibility during deployment transitions
- Ensure zero-downtime deployment capability for production environments

## Output

1. Deployment process gaps and risks identified
2. Environment promotion improvements made or proposed
3. Shared infrastructure concerns and isolation recommendations
4. Credential and secret management issues resolved
5. Rollback procedure validation results
6. Post-deployment verification checklist and results
7. Recommendations for monitoring, alerting, and incident response
