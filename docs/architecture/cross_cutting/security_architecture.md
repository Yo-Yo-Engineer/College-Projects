# Security Architecture

## Overview

**Security Architecture** defines the principles, patterns, and layered controls that protect a system's data, services, and users from unauthorized access, disclosure, modification, and disruption. Rather than treating security as an afterthought bolted onto a finished application, security architecture embeds protection into every layer of the system from the start.

Modern security follows the **Zero Trust** model: never trust, always verify — regardless of whether the request originates inside or outside the network perimeter.

Core principles:

- **Defense in Depth** — Multiple overlapping security layers so that a breach in one does not compromise the system
- **Least Privilege** — Grant only the minimum permissions required for each identity to perform its function
- **Zero Trust** — Verify explicitly on every request based on all available signals (identity, device, location, behavior)
- **Assume Breach** — Design systems so that when a component is compromised, the blast radius is contained
- **Secure by Default** — Ship with the most restrictive safe configuration; require explicit opt-in for risky features
- **Fail Closed** — When a security check encounters an error, deny access rather than allowing it

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Security Architecture                                │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                       Perimeter Layer                                 │ │
│   │   WAF, DDoS Protection, CDN TLS Termination                          │ │
│   └────────────────────────────┬──────────────────────────────────────────┘ │
│                                │                                            │
│   ┌────────────────────────────▼──────────────────────────────────────────┐ │
│   │                       Gateway Layer                                   │ │
│   │   Authentication, Rate Limiting, Request Validation, API Versioning   │ │
│   └────────────────────────────┬──────────────────────────────────────────┘ │
│                                │                                            │
│   ┌────────────────────────────▼──────────────────────────────────────────┐ │
│   │                       Application Layer                               │ │
│   │   Authorization (RBAC/ABAC), Input Validation, Business Rule Checks   │ │
│   └────────────────────────────┬──────────────────────────────────────────┘ │
│                                │                                            │
│   ┌────────────────────────────▼──────────────────────────────────────────┐ │
│   │                       Data Layer                                      │ │
│   │   Encryption at Rest, Field-Level Encryption, Access Controls, Audit  │ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│   Cross-Cutting: ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐  │
│                   │ Secrets      │ │ Audit        │ │ Security           │  │
│                   │ Management   │ │ Logging      │ │ Monitoring         │  │
│                   └──────────────┘ └──────────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Authentication

Authentication verifies **who** the caller is. Modern systems delegate authentication to an identity provider using standard protocols.

#### OAuth 2.0 and OpenID Connect

OAuth 2.0 handles **authorization** (what can a token do?) while OpenID Connect (OIDC) adds an **identity layer** (who is the user?):

```
// OAuth 2.0 Authorization Code Flow (with PKCE for public clients)
//
// Best practice for web apps, mobile apps, and SPAs
// PKCE (Proof Key for Code Exchange) prevents authorization code interception

FLOW AuthorizationCodeWithPKCE

    // Step 1: Client generates a cryptographic code verifier and challenge
    codeVerifier  = generateRandomString(128)
    codeChallenge = BASE64URL(SHA256(codeVerifier))

    // Step 2: Client redirects user to the authorization server
    REDIRECT TO authServer.authorizeUrl
        response_type  = "code"
        client_id      = CLIENT_ID
        redirect_uri   = "https://app.example.com/callback"
        scope          = "openid profile email"
        state          = generateRandomString(32)     // CSRF protection
        code_challenge  = codeChallenge
        code_challenge_method = "S256"

    // Step 3: User authenticates and consents at the auth server
    // Auth server redirects back with an authorization code

    // Step 4: Client exchanges the code for tokens
    POST authServer.tokenUrl
        grant_type     = "authorization_code"
        code           = receivedAuthorizationCode
        redirect_uri   = "https://app.example.com/callback"
        client_id      = CLIENT_ID
        code_verifier  = codeVerifier     // Proves possession

    // Step 5: Auth server returns tokens
    RESPONSE
        access_token   : "eyJhbGci..."    // Short-lived (minutes to hours)
        refresh_token  : "dGhpcyBp..."    // Long-lived, stored securely
        id_token       : "eyJhbGci..."    // OIDC identity claims (who the user is)
        token_type     : "Bearer"
        expires_in     : 3600

END FLOW
```

#### Token Validation

Every service must validate tokens independently — never trust a token without verification:

```
// Token validation middleware (runs on every request)
FUNCTION authenticateRequest(request)
    token = extractBearerToken(request.headers["Authorization"])

    IF token IS NULL THEN
        RETURN RESPONSE 401 Unauthorized
            body: { error: "Missing authentication token" }
    END IF

    TRY
        // 1. Verify the token signature using the issuer's public key
        claims = verifyTokenSignature(token, issuer.publicKeys)

        // 2. Validate standard claims
        IF claims.exp < NOW() THEN
            THROW TokenExpiredError
        END IF
        IF claims.iss != EXPECTED_ISSUER THEN
            THROW InvalidIssuerError
        END IF
        IF claims.aud != THIS_SERVICE_IDENTIFIER THEN
            THROW InvalidAudienceError
        END IF

        // 3. Attach verified identity to request context
        request.identity = NEW Identity(
            subject = claims.sub,
            roles   = claims.roles,
            scopes  = claims.scope,
            email   = claims.email
        )

    CATCH error
        RETURN RESPONSE 401 Unauthorized
            body: { error: "Invalid or expired token" }
    END TRY
END FUNCTION
```

#### Service-to-Service Authentication

Internal services authenticate using machine credentials — never embed shared secrets in code:

```
// Client Credentials Flow — machine-to-machine (no user context)
FUNCTION getServiceToken(targetServiceScope)
    // Credential retrieved from secrets manager at startup, NOT hardcoded
    credential = secretsManager.get("order-service-credential")

    POST authServer.tokenUrl
        grant_type    = "client_credentials"
        client_id     = credential.clientId
        client_secret = credential.clientSecret
        scope         = targetServiceScope

    RETURN response.access_token
END FUNCTION

// Managed identity (preferred in cloud environments)
// The platform provides credentials automatically — no secrets to manage
FUNCTION getServiceTokenViaIdentity(targetServiceScope)
    RETURN platformIdentityProvider.getToken(targetServiceScope)
    // The runtime environment supplies the proof of identity
    // No client_secret needed
END FUNCTION
```

### Authorization

Authorization determines **what** an authenticated identity is allowed to do.

#### Role-Based Access Control (RBAC)

Permissions are assigned to roles, and roles are assigned to users:

```
// RBAC — roles aggregate permissions
DEFINE ROLE admin
    PERMISSIONS: [create, read, update, delete, manage_users]

DEFINE ROLE editor
    PERMISSIONS: [create, read, update]

DEFINE ROLE viewer
    PERMISSIONS: [read]

// Authorization check
FUNCTION authorize(identity, requiredPermission, resource)
    role = identity.roles.find(r -> r.scope == resource.type)

    IF role IS NULL THEN
        THROW ForbiddenError("No role assigned for " + resource.type)
    END IF

    IF requiredPermission NOT IN role.permissions THEN
        THROW ForbiddenError("Role '" + role.name + "' lacks permission '" + requiredPermission + "'")
    END IF

    // Authorized — proceed
END FUNCTION

// Usage in a handler
FUNCTION deleteOrderHandler(request)
    authenticate(request)                                  // Who are you?
    authorize(request.identity, "delete", OrderResource)   // Can you do this?

    orderService.delete(request.params.orderId)            // Proceed
    RETURN RESPONSE 204 No Content
END FUNCTION
```

#### Attribute-Based Access Control (ABAC)

Authorization decisions based on attributes of the user, resource, action, and environment:

```
// ABAC — fine-grained, policy-driven authorization
INTERFACE PolicyEngine
    FUNCTION evaluate(context : AuthorizationContext) -> Decision

STRUCTURE AuthorizationContext
    subject     : SubjectAttributes     // user role, department, clearance
    resource    : ResourceAttributes    // resource type, owner, classification
    action      : String                // read, write, delete, approve
    environment : EnvironmentAttributes // time, location, device trust level

// Policy example: "Managers can approve expenses under 10,000 in their own department"
POLICY ExpenseApproval
    CONDITION:
        subject.role == "manager"
        AND action == "approve"
        AND resource.type == "expense"
        AND resource.amount < 10000
        AND resource.department == subject.department
    DECISION: ALLOW

// Policy example: "No access outside business hours for non-admin users"
POLICY BusinessHoursOnly
    CONDITION:
        subject.role != "admin"
        AND (environment.hour < 8 OR environment.hour > 18)
    DECISION: DENY

// Policy evaluation
FUNCTION authorizeWithPolicies(context)
    decision = policyEngine.evaluate(context)

    IF decision == DENY THEN
        auditLog.record("ACCESS_DENIED", context)
        THROW ForbiddenError("Policy denied access")
    END IF

    auditLog.record("ACCESS_GRANTED", context)
END FUNCTION
```

### Input Validation and Injection Prevention

All data from external sources is untrusted. Validate and sanitize at every system boundary:

```
// OWASP Top 10 — Injection Prevention

// 1. SQL Injection — NEVER concatenate user input into queries
BAD:
    query = "SELECT * FROM users WHERE email = '" + userInput + "'"
    // Attacker sends: ' OR '1'='1' --
    // Resulting query: SELECT * FROM users WHERE email = '' OR '1'='1' --'

GOOD:
    query = preparedStatement("SELECT * FROM users WHERE email = ?", userInput)
    // Input is treated as a data parameter, never as SQL

// 2. Cross-Site Scripting (XSS) — encode output for the target context
BAD:
    html = "<div>Welcome, " + userName + "</div>"
    // Attacker sets name to: <script>stealCookies()</script>

GOOD:
    html = "<div>Welcome, " + htmlEncode(userName) + "</div>"
    // Output: <div>Welcome, &lt;script&gt;stealCookies()&lt;/script&gt;</div>

// 3. Command Injection — avoid shell commands with user input
BAD:
    execute("convert " + uploadedFilename + " output.pdf")

GOOD:
    // Use language-native libraries instead of shell commands
    // If shell is unavoidable, use explicit argument arrays (no shell interpolation)
    executeWithArgs(["convert", sanitizeFilename(uploadedFilename), "output.pdf"])

// 4. Path Traversal — validate and canonicalize file paths
FUNCTION serveFile(requestedPath)
    canonicalPath = resolvePath(BASE_DIRECTORY, requestedPath)

    IF NOT canonicalPath.startsWith(BASE_DIRECTORY) THEN
        THROW ForbiddenError("Path traversal attempt detected")
    END IF

    RETURN readFile(canonicalPath)
END FUNCTION

// General input validation pattern
FUNCTION validateInput(input, rules)
    // Allowlist approach — define what IS valid, reject everything else
    FOR EACH rule IN rules
        IF NOT rule.validate(input[rule.field]) THEN
            THROW ValidationError(rule.field, rule.errorMessage)
        END IF
    END FOR
END FUNCTION

VALIDATION RULES FOR CreateUser
    email    : MATCHES pattern EMAIL_REGEX, MAX_LENGTH 254
    name     : MATCHES pattern ALPHANUMERIC_AND_SPACES, MAX_LENGTH 100, MIN_LENGTH 1
    age      : INTEGER, MIN 0, MAX 150
    // Use allowlists, not blocklists
```

### Encryption

Protect data confidentiality both at rest and in transit:

```
// Data in Transit — TLS everywhere
CONFIGURATION TransportSecurity
    ENFORCE TLS 1.2 or higher for ALL connections
    DISABLE TLS 1.0 and TLS 1.1
    USE strong cipher suites only
    ENABLE HSTS (HTTP Strict Transport Security) for web endpoints
    REDIRECT all HTTP to HTTPS
    VERIFY server certificates in service-to-service calls (no skipping)

// Data at Rest — encrypt stored data
CONFIGURATION StorageEncryption
    ENCRYPT database storage with platform-managed or customer-managed keys
    ENCRYPT file storage and backups
    ENCRYPT message queue data at rest

// Field-Level Encryption — for highly sensitive fields
FUNCTION encryptSensitiveField(plaintext, keyId)
    key = keyManagementService.getKey(keyId)
    ciphertext = encrypt(plaintext, key, algorithm = "AES-256-GCM")
    RETURN ciphertext
END FUNCTION

FUNCTION decryptSensitiveField(ciphertext, keyId)
    key = keyManagementService.getKey(keyId)
    plaintext = decrypt(ciphertext, key, algorithm = "AES-256-GCM")
    RETURN plaintext
END FUNCTION

// Example: storing payment information
ORDER RECORD
    orderId         : UUID              // Not encrypted
    customerName    : String            // Not encrypted
    cardLastFour    : String            // Stored as last-4 only (data minimization)
    cardTokenRef    : String            // Tokenized reference (not actual card data)
    shippingAddress : ENCRYPTED String  // Encrypted at field level
```

### Secrets Management

Application secrets (API keys, connection strings, certificates) must never be hardcoded or stored in source control:

```
// Anti-pattern — secrets in code or config files
BAD:
    databasePassword = "P@ssw0rd123"                     // Hardcoded
    apiKey = config.file["secrets"]["api_key"]            // In repo

// Correct — secrets from a dedicated secrets manager
FUNCTION initializeService()
    // Secrets loaded at startup from a secure vault
    dbCredential = secretsManager.getSecret("database-connection-string")
    apiKey       = secretsManager.getSecret("payment-provider-api-key")
    signingKey   = secretsManager.getSecret("jwt-signing-key")

    // Secrets are rotated automatically by the secrets manager
    // Application listens for rotation events and refreshes
    secretsManager.onRotation("database-connection-string", FUNCTION(newSecret)
        connectionPool.updateCredential(newSecret)
    END FUNCTION)
END FUNCTION

// Environment-based secret resolution (from environment variables in production)
// Environment variables are injected by the deployment platform, NOT committed to repos
dbHost     = ENV("DB_HOST")
dbPassword = ENV("DB_PASSWORD")     // Injected by orchestrator from secrets store
```

### Security Headers

HTTP response headers that mitigate common web attacks:

```
// Security headers for web applications
FUNCTION addSecurityHeaders(response)
    // Content Security Policy — restrict where content can load from
    response.setHeader("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:")

    // Prevent MIME type sniffing
    response.setHeader("X-Content-Type-Options", "nosniff")

    // Clickjacking protection
    response.setHeader("X-Frame-Options", "DENY")

    // Enable HSTS — force HTTPS for all future requests
    response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

    // Referrer policy — limit information leakage
    response.setHeader("Referrer-Policy", "strict-origin-when-cross-origin")

    // Permissions policy — disable unused browser features
    response.setHeader("Permissions-Policy", "camera=(), microphone=(), geolocation=()")

    RETURN response
END FUNCTION
```

### Audit Logging

Record security-relevant events for forensics and compliance:

```
// Audit log — immutable record of security events
INTERFACE AuditLogger
    FUNCTION log(event : AuditEvent) -> Void

STRUCTURE AuditEvent
    timestamp  : DateTime
    eventType  : String        // LOGIN, LOGOUT, ACCESS_GRANTED, ACCESS_DENIED,
                               // DATA_MODIFIED, CONFIG_CHANGED, SECRET_ACCESSED
    actor      : String        // Who performed the action (user ID or service ID)
    action     : String        // What was attempted
    resource   : String        // What was acted upon
    outcome    : String        // SUCCESS or FAILURE
    sourceIp   : String        // Where the request originated
    details    : Map           // Additional context (non-sensitive)

// Security events that MUST be logged
AUDIT REQUIREMENTS
    1. Authentication: all login attempts (success and failure)
    2. Authorization: all access denials
    3. Data access: reads of sensitive data (PII, financial)
    4. Data modification: creates, updates, deletes of business-critical data
    5. Configuration changes: permission changes, role assignments
    6. Secret access: when secrets are retrieved or rotated
    7. Administrative actions: user management, system configuration

// Audit logs must be:
// - Immutable (append-only, no edits or deletes)
// - Stored separately from application data
// - Retained per compliance requirements
// - Monitored for anomalies (unusual access patterns, failed logins)

FUNCTION logSecurityEvent(actor, action, resource, outcome, details)
    event = NEW AuditEvent(
        timestamp = NOW(),
        eventType = classifyEvent(action),
        actor     = actor,
        action    = action,
        resource  = resource,
        outcome   = outcome,
        sourceIp  = currentRequest.sourceIp,
        details   = redactSensitiveFields(details)  // Never log passwords or tokens
    )
    auditLogger.log(event)
END FUNCTION
```

## Key Design Considerations

### OWASP Top 10 Mitigation Summary

```
┌─────────────────────────────────────┬──────────────────────────────────────────────┐
│ OWASP Risk                          │ Mitigation Strategy                           │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A01: Broken Access Control          │ RBAC/ABAC enforcement at every endpoint,     │
│                                     │ deny by default, server-side checks          │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A02: Cryptographic Failures         │ TLS everywhere, encrypt at rest, no custom   │
│                                     │ crypto, use proven algorithms (AES-256, RSA) │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A03: Injection                      │ Parameterized queries, output encoding,      │
│                                     │ allowlist validation, no shell interpolation  │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A04: Insecure Design                │ Threat modeling, abuse case analysis,         │
│                                     │ defense in depth, security requirements       │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A05: Security Misconfiguration      │ Secure defaults, remove unused features,     │
│                                     │ automated config scanning, security headers   │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A06: Vulnerable Components          │ Dependency scanning, automated updates,       │
│                                     │ SBOMs (Software Bill of Materials)            │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A07: Auth Failures                  │ MFA, strong password policies, account        │
│                                     │ lockout, credential stuffing protection       │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A08: Software/Data Integrity        │ Signed artifacts, CI/CD pipeline security,    │
│                                     │ dependency integrity verification             │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A09: Logging & Monitoring Failures  │ Centralized audit logging, alerting on        │
│                                     │ anomalies, immutable log storage              │
├─────────────────────────────────────┼──────────────────────────────────────────────┤
│ A10: Server-Side Request Forgery    │ Allowlist outbound destinations, validate     │
│                                     │ URLs, block internal IP ranges                │
└─────────────────────────────────────┴──────────────────────────────────────────────┘
```

### Zero Trust Implementation

Zero Trust replaces the traditional "trusted network perimeter" model with continuous verification:

```
// Zero Trust decision flow — every request, every time
FUNCTION zeroTrustEvaluate(request)

    // 1. VERIFY IDENTITY — strong authentication
    identity = authenticateRequest(request)    // Token validation
    IF NOT identity.mfaVerified THEN
        RETURN CHALLENGE("Multi-factor authentication required")
    END IF

    // 2. VERIFY DEVICE — is the device compliant?
    device = deviceTrustService.evaluate(request.deviceFingerprint)
    IF device.complianceStatus != "compliant" THEN
        RETURN DENY("Device does not meet security requirements")
    END IF

    // 3. VERIFY CONTEXT — is this request normal?
    riskScore = riskEngine.assess(
        identity    = identity,
        device      = device,
        location    = request.sourceGeo,
        time        = NOW(),
        behavior    = behaviorAnalytics.getProfile(identity.subject)
    )
    IF riskScore > HIGH_RISK_THRESHOLD THEN
        RETURN CHALLENGE("Step-up authentication required")
    END IF

    // 4. ENFORCE LEAST PRIVILEGE — minimum necessary access
    authorize(identity, request.action, request.resource)

    // 5. SEGMENT ACCESS — limit blast radius
    // Identity can only reach the specific service and data it needs
    // No lateral movement to other services

    // 6. LOG EVERYTHING — for continuous monitoring
    auditLog.record(identity, request, "ACCESS_GRANTED")

    RETURN ALLOW
END FUNCTION
```

## Project Structure

```
src/
│
├── security/
│   ├── authentication/
│   │   ├── token-validator/            # JWT/token verification
│   │   ├── oauth/                      # OAuth 2.0 / OIDC integration
│   │   ├── middleware/                 # Auth middleware for request pipeline
│   │   └── service-identity/           # Service-to-service auth (client credentials)
│   │
│   ├── authorization/
│   │   ├── rbac/                       # Role-based access control
│   │   ├── abac/                       # Attribute-based policies
│   │   ├── policies/                   # Policy definitions
│   │   └── middleware/                 # Authorization middleware
│   │
│   ├── input-validation/
│   │   ├── sanitizers/                 # Input sanitization (XSS, injection)
│   │   ├── validators/                 # Schema and format validators
│   │   └── middleware/                 # Validation middleware
│   │
│   ├── encryption/
│   │   ├── at-rest/                    # Storage encryption utilities
│   │   ├── in-transit/                 # TLS configuration
│   │   └── field-level/               # Sensitive field encryption
│   │
│   ├── secrets/
│   │   ├── manager/                    # Secrets manager integration
│   │   └── rotation/                   # Secret rotation handlers
│   │
│   ├── audit/
│   │   ├── logger/                     # Audit event logging
│   │   ├── events/                     # Audit event type definitions
│   │   └── monitoring/                 # Anomaly detection on audit stream
│   │
│   └── headers/                        # Security header middleware
│
├── config/
│   ├── security-policies/              # RBAC role definitions, ABAC policies
│   ├── cors/                           # CORS configuration
│   └── tls/                            # TLS certificate configuration
│
└── tests/
    ├── security/
    │   ├── auth/                       # Authentication flow tests
    │   ├── authorization/              # Permission and policy tests
    │   ├── injection/                  # SQL injection, XSS test cases
    │   └── penetration/                # Automated security scan configs
    └── compliance/                     # Compliance validation tests
```

## Benefits

1. **Risk Reduction** — Defense in depth ensures no single vulnerability compromises the entire system
2. **Compliance** — Structured authentication, authorization, encryption, and audit logging align with regulatory requirements (GDPR, SOC 2, HIPAA, PCI-DSS)
3. **Trust** — Externally verifiable security posture builds confidence with customers, partners, and auditors
4. **Blast Radius Containment** — Zero Trust segmentation limits the impact of any single compromised component
5. **Operational Visibility** — Audit logging and monitoring provide real-time awareness of security posture and incidents
6. **Evolvability** — Standard protocols (OAuth 2.0, OIDC) and modular design allow security infrastructure to evolve without rewriting applications

## Trade-offs

| Advantage                                       | Consideration                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------- |
| Defense in depth prevents single-point failures | Multiple security layers add latency and operational complexity     |
| Least privilege minimizes blast radius          | Overly granular permissions become difficult to manage              |
| Zero Trust enables remote/cloud access          | Continuous verification requires identity infrastructure investment |
| Encryption protects data confidentiality        | Key management and rotation add operational burden                  |
| Audit logging enables forensics                 | High-volume logging requires storage and analysis tooling           |
| Standardized auth (OAuth/OIDC) is portable      | Identity provider becomes a critical dependency                     |

## When to Use

✅ **Good fit for:**

- Any production system handling user data or business-critical operations
- Systems subject to regulatory compliance (financial, healthcare, government)
- Microservices architectures where services communicate over a network
- APIs exposed to external consumers or third-party integrations
- Organizations adopting cloud or hybrid infrastructure

❌ **Not ideal for:**

- Isolated offline tools with no network access and no sensitive data
- Throwaway prototypes that will never handle real user data
- Internal development utilities running in fully trusted, air-gapped environments (even then, apply baseline hygiene)

## References

- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [NIST SP 800-207 — Zero Trust Architecture (2020)](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [Microsoft Zero Trust Guidance Center](https://learn.microsoft.com/en-us/security/zero-trust/)
- [Azure Architecture Center — Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/best-practices-and-patterns)
- [OAuth 2.0 Authorization Framework — RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [OpenID Connect Core 1.0 Specification](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP Application Security Verification Standard (ASVS)](https://owasp.org/www-project-application-security-verification-standard/)
- [CWE/SANS Top 25 Most Dangerous Software Weaknesses](https://cwe.mitre.org/top25/archive/2023/2023_top25_list.html)
