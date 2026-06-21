# Environment and Configuration Management Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve environment and configuration management for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review configuration patterns, secret handling, and environment parity
- Keep credentials and secret values out of review outputs

## Focus Areas

### Configuration Externalization

- Verify all environment-specific configuration is externalized — not hardcoded in source code
- Ensure configuration follows the Twelve-Factor App: strict separation of config from code
- Check for environment variables, configuration files, or configuration services as the source of truth
- Verify default values are safe and sensible — not production credentials or open permissions
- Ensure feature flags and toggles are managed through a configuration system, not code branches

### Secret Management

- Verify secrets (passwords, API keys, connection strings, tokens, certificates) are NEVER in source code, config files, prompts, or CI scripts
- Ensure secrets are loaded from a dedicated secret manager (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault, or equivalent)
- Check that secret references (names/paths) are in config, but secret values are resolved at runtime
- Verify secrets are not logged, printed to console, or included in error messages or stack traces
- Ensure secret rotation is possible without application redeployment
- Check for secrets accidentally committed to version control history (even in past commits)

### Connection String and Endpoint Management

- Verify database connection strings, service endpoints, and API URLs are environment-specific and externalized
- Ensure connection strings use managed identities or key vault references where the platform supports them
- Check that TLS/SSL is enforced for all external connections (databases, APIs, message queues)
- Verify connection timeout, retry, and pooling settings are configured per environment
- Ensure service discovery or DNS-based resolution is used rather than hardcoded IP addresses and ports

### Environment Parity

- Verify dev, UAT, staging, and production environments are structurally identical — differing only in configuration values and scale
- Ensure infrastructure-as-code defines all environments from the same templates with environment-specific variables
- Check that environment-specific overrides are minimal and clearly documented
- Verify shared infrastructure between environments has clear boundaries, access controls, and capacity allocation
- Ensure environment naming conventions are consistent across infrastructure, configuration, and monitoring

### Configuration Validation

- Verify the application validates all required configuration at startup — fail fast on missing or invalid values
- Ensure configuration schema is defined and enforced (types, required fields, allowed values)
- Check for configuration drift detection between environments
- Verify configuration changes are versioned and auditable
- Ensure configuration reload capability exists for settings that can change without restart

### Feature Flags and Toggles

- Verify feature flags are used to decouple deployments from feature releases
- Ensure feature flags have clear ownership, expiry dates, and cleanup procedures
- Check that feature flag evaluation does not introduce performance overhead on hot paths
- Verify feature flags are testable — both enabled and disabled paths are covered
- Ensure flag management is centralized and accessible to the team

### Environment-Specific Concerns

- Verify logging levels are appropriate per environment (verbose in dev, minimal in production)
- Ensure debug endpoints, profiling tools, and admin panels are disabled in production
- Check that CORS, CSP, and security headers are configured correctly per environment
- Verify monitoring, alerting, and tracing are configured for each environment
- Ensure development environments do not accidentally connect to production resources

### Infrastructure as Code

- Verify all infrastructure is defined as code (Terraform, Bicep/ARM, Pulumi, CloudFormation)
- Ensure IaC follows the DRY principle — shared modules with environment-specific variable files
- Check that infrastructure changes go through the same review and CI process as application code
- Verify state management for IaC is secure and backed up (remote state with locking)
- Ensure infrastructure outputs (endpoints, resource IDs) feed into application configuration automatically

## Reference Standards

- The Twelve-Factor App (III: Config, X: Dev/prod parity)
- NIST SP 800-53 (Configuration Management controls)
- CIS Benchmarks for cloud configuration hardening
- GitOps principles for declarative configuration management

## Constraints

- Keep secret values out of the review process and all outputs
- Preserve existing configuration patterns unless a change is clearly justified
- Ensure configuration changes are backward-compatible with running instances
- Prefer platform-native secret management over custom solutions

## Output

1. Configuration and secret management issues identified, ranked by risk
2. Hardcoded credentials or exposed secrets found and remediated
3. Environment parity gaps and isolation concerns
4. Configuration validation improvements applied or recommended
5. Infrastructure-as-code improvements made or proposed
6. Recommendations for secret rotation, feature flag management, and monitoring
