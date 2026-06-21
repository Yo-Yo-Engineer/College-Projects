# Security Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Perform a security-focused review of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Prioritize vulnerabilities with the highest risk and exploitability
- Keep changes within the security domain

## Focus Areas

### Input Validation and Injection Prevention

- Validate, sanitize, and encode all external input at system boundaries
- Prevent SQL injection, XSS, command injection, path traversal, and template injection
- Use parameterized queries and prepared statements for all database access
- Apply allowlists over denylists where practical

### Authentication and Authorization

- Verify authentication is enforced on all protected endpoints and operations
- Ensure authorization checks follow the principle of least privilege
- Check for broken access control — IDOR, privilege escalation, missing function-level access control
- Validate session management: secure cookie flags, token expiry, rotation, and invalidation

### Secrets and Configuration

- Ensure no hardcoded secrets, API keys, tokens, or credentials exist in source code
- Verify secrets are loaded from environment variables, vaults, or secret managers
- Check that sensitive configuration is not logged, exposed in error responses, or committed to version control
- Validate .gitignore and .env handling

### Cryptography and Data Protection

- Use current, recognized algorithms and libraries — avoid deprecated or custom cryptography
- Verify proper key management, rotation, and storage
- Ensure sensitive data is encrypted at rest and in transit (TLS)
- Check for insecure randomness in security-sensitive contexts

### Dependency and Supply Chain Security

- Identify known vulnerabilities in dependencies (CVEs)
- Verify dependency pinning and lockfile integrity
- Check for excessive or unnecessary dependency surface area
- Ensure dependencies are sourced from trusted registries

### Error Handling and Information Disclosure

- Ensure error responses do not leak internal details, stack traces, or sensitive data
- Verify consistent error handling that fails securely (deny by default)
- Check that logging does not capture sensitive user data or secrets

### Security Headers and Transport

- Verify appropriate security headers: Content-Security-Policy, X-Content-Type-Options, Strict-Transport-Security, X-Frame-Options
- Check CORS configuration for overly permissive origins
- Ensure HTTPS is enforced and HTTP is redirected or blocked

## Reference Standards

- OWASP Top 10 (2021)
- CWE/SANS Top 25 Most Dangerous Software Weaknesses
- OWASP Application Security Verification Standard (ASVS)
- Principle of Least Privilege
- Defense in Depth

## Constraints

- Preserve existing functionality unless a change is required to fix a vulnerability
- Prefer targeted, minimal-risk fixes over broad rewrites
- Retain security controls unless removal is explicitly justified
- Use existing dependencies; introduce new ones only when required to fix a vulnerability

## Output

1. Vulnerabilities found, ranked by severity (critical / high / medium / low)
2. Changes made or proposed with justification
3. Residual risks and accepted trade-offs
4. Recommendations for follow-up (penetration testing, dependency audit, etc.)
5. Any assumptions about the deployment environment or threat model
