# API Design Architecture

## Overview

**API Design Architecture** defines the principles, patterns, and conventions for building application programming interfaces that are consistent, evolvable, and developer-friendly. APIs are the contracts between system components — whether between frontend and backend, between microservices, or between your system and external consumers.

Good API design is independent of any specific framework or language. The principles apply whether you are building a REST API over HTTP, a gRPC service, a GraphQL endpoint, or an asynchronous message-based interface.

Core principles:

- **Contract-First** — Design the API surface before implementing it; the interface is the product
- **Consistency** — Use uniform naming, error formats, and conventions across all endpoints
- **Evolvability** — APIs must change without breaking existing consumers; versioning is a first-class concern
- **Discoverability** — A well-designed API is intuitive; developers can predict behavior from conventions alone
- **Least Surprise** — Operations behave as their names suggest; side effects are explicit
- **Separation of Concerns** — Public APIs (for external consumers) and internal APIs (for service-to-service communication) have different requirements

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API Architecture                                    │
│                                                                             │
│   External Consumers          Internal Services                             │
│   ┌──────────────────┐        ┌──────────────────┐                          │
│   │ Web / Mobile     │        │ Service A        │                           │
│   │ Third-Party Apps │        │ Service B        │                           │
│   └──────┬───────────┘        └──────┬───────────┘                          │
│          │                           │                                      │
│          ▼                           ▼                                      │
│   ┌──────────────┐            ┌──────────────┐                              │
│   │ Public API   │            │ Internal API │                               │
│   │ (REST/GraphQL│            │ (gRPC / async│                               │
│   │  versioned)  │            │  messaging)  │                               │
│   └──────┬───────┘            └──────┬───────┘                              │
│          │                           │                                      │
│          ▼                           ▼                                      │
│   ┌─────────────────────────────────────────────────────┐                   │
│   │                   API Gateway                        │                   │
│   │   (routing, auth, rate limiting, versioning)         │                   │
│   └──────────────────────┬──────────────────────────────┘                   │
│                          │                                                  │
│          ┌───────────────┼───────────────┐                                  │
│          ▼               ▼               ▼                                  │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐                           │
│   │ Service X  │  │ Service Y  │  │ Service Z  │                            │
│   └────────────┘  └────────────┘  └────────────┘                           │
│                                                                             │
│   Cross-Cutting: ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐  │
│                   │ API Docs    │ │ Schema       │ │ Contract Testing  │  │
│                   │ (OpenAPI /  │ │ Validation   │ │                    │  │
│                   │  Protobuf)  │ │              │ │                    │  │
│                   └──────────────┘ └──────────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### REST (Representational State Transfer)

REST models the system as a collection of **resources** identified by URIs, manipulated through standard HTTP methods:

```
// REST resource design — resources are nouns, methods are verbs
//
// Collection resource:  /orders
// Instance resource:    /orders/{orderId}
// Sub-resource:         /orders/{orderId}/items

// HTTP Method Semantics
// ┌──────────┬───────────────────────┬──────────────┬─────────────┐
// │ Method   │ Purpose               │ Idempotent?  │ Safe?       │
// ├──────────┼───────────────────────┼──────────────┼─────────────┤
// │ GET      │ Retrieve resource(s)  │ Yes          │ Yes         │
// │ POST     │ Create new resource   │ No           │ No          │
// │ PUT      │ Replace entire entity │ Yes          │ No          │
// │ PATCH    │ Partial update        │ No*          │ No          │
// │ DELETE   │ Remove resource       │ Yes          │ No          │
// └──────────┴───────────────────────┴──────────────┴─────────────┘
// * PATCH can be made idempotent with conditional headers

// Resource naming conventions
GOOD:
    GET    /customers                    // List customers
    GET    /customers/{id}               // Get one customer
    POST   /customers                    // Create customer
    PUT    /customers/{id}               // Replace customer
    PATCH  /customers/{id}               // Update customer fields
    DELETE /customers/{id}               // Remove customer
    GET    /customers/{id}/orders        // List customer's orders

BAD:
    GET    /getCustomers                 // Verb in URL
    POST   /createCustomer              // Verb in URL
    GET    /customer                     // Inconsistent pluralization
    POST   /customers/{id}/delete       // Using POST for deletion
```

### GraphQL

GraphQL provides a single endpoint where clients request exactly the data they need:

```
// GraphQL — client-driven queries through a typed schema

// Schema definition (server-side)
TYPE Customer
    id        : ID
    name      : String
    email     : String
    orders    : List<Order>

TYPE Order
    id        : ID
    status    : OrderStatus
    total     : Decimal
    items     : List<OrderItem>
    createdAt : DateTime

TYPE Query
    customer(id : ID) : Customer
    orders(customerId : ID, status : OrderStatus) : List<Order>

TYPE Mutation
    createOrder(input : CreateOrderInput) : Order
    cancelOrder(id : ID) : Order

// Client query — fetches exactly what it needs (no over/under-fetching)
QUERY
    customer(id: "abc-123")
        name
        email
        orders(status: SHIPPED)
            id
            total
            items
                productName
                quantity

// Advantages over REST for this case:
// - Single round trip instead of GET /customers/abc-123 + GET /customers/abc-123/orders
// - No over-fetching — only requested fields returned
// - Strongly typed schema serves as documentation
```

### gRPC (Remote Procedure Call)

gRPC uses Protocol Buffers for strongly typed, binary-serialized service definitions. Optimized for service-to-service communication:

```
// gRPC service definition (Protocol Buffer style)

SERVICE OrderService
    RPC CreateOrder(CreateOrderRequest) RETURNS OrderResponse
    RPC GetOrder(GetOrderRequest) RETURNS OrderResponse
    RPC ListOrders(ListOrdersRequest) RETURNS ListOrdersResponse
    RPC StreamOrderUpdates(StreamRequest) RETURNS STREAM OrderUpdate  // Server streaming

MESSAGE CreateOrderRequest
    customerId  : String
    items       : REPEATED OrderItemInput

MESSAGE OrderResponse
    orderId     : String
    status      : OrderStatus
    total       : Double
    createdAt   : Timestamp

ENUM OrderStatus
    PENDING   = 0
    CONFIRMED = 1
    SHIPPED   = 2
    DELIVERED = 3

// Trade-offs vs REST:
// ✅ Binary serialization — smaller payloads, faster parsing
// ✅ Strongly typed contracts — code generation for client/server
// ✅ Streaming support — bidirectional real-time communication
// ❌ Not browser-native — requires a proxy or gRPC-Web
// ❌ Less human-readable — binary format, harder to debug manually
// ❌ Tighter coupling — client and server share compiled schema
```

### Async / Message-Based APIs

For operations that don't require an immediate response:

```
// Asynchronous API — accept request, process later

// Client sends a command
POST /orders
    Body: { customerId: "abc", items: [...] }
    Response: 202 Accepted
              Location: /orders/status/job-456
              { jobId: "job-456", status: "processing" }

// Client polls for result (or subscribes to webhook/event)
GET /orders/status/job-456
    Response: 200 OK
              { jobId: "job-456", status: "completed", orderId: "order-789" }

// Alternative: Webhook callback
POST /orders
    Body: { customerId: "abc", items: [...], callbackUrl: "https://client.example/hooks" }
    Response: 202 Accepted

// Server calls back when done:
POST https://client.example/hooks
    Body: { event: "order.created", orderId: "order-789", status: "confirmed" }
```

### Core Design Patterns

#### Error Handling

Use a consistent, machine-readable error format across all endpoints:

```
// Standardized error response (RFC 9457 — Problem Details)
STRUCTURE ProblemDetail
    type     : URI           // Error type identifier (stable, documentable)
    title    : String        // Human-readable summary
    status   : Integer       // HTTP status code
    detail   : String        // Specific explanation for this occurrence
    instance : URI           // URI identifying this specific occurrence
    errors   : List<FieldError> OR NULL   // For validation errors

STRUCTURE FieldError
    field   : String         // Which field failed
    message : String         // Why it failed

// Examples

// 400 — Validation error
RESPONSE 400 Bad Request
    type:     "/errors/validation-error"
    title:    "Validation Failed"
    status:   400
    detail:   "One or more fields failed validation"
    errors:
        - field: "email",    message: "Must be a valid email address"
        - field: "quantity", message: "Must be greater than zero"

// 404 — Not found
RESPONSE 404 Not Found
    type:     "/errors/resource-not-found"
    title:    "Resource Not Found"
    status:   404
    detail:   "Order with ID 'ord-999' does not exist"

// 409 — Conflict
RESPONSE 409 Conflict
    type:     "/errors/order-already-submitted"
    title:    "Order Already Submitted"
    status:   409
    detail:   "Order 'ord-123' has already been submitted and cannot be modified"

// 429 — Rate limited
RESPONSE 429 Too Many Requests
    type:     "/errors/rate-limit-exceeded"
    title:    "Rate Limit Exceeded"
    status:   429
    detail:   "You have exceeded 100 requests per minute"
    Retry-After: 30   // Header indicating when to retry
```

#### Pagination

Never return unbounded collections. Provide consistent pagination across all list endpoints:

```
// Offset-based pagination (simple, supports random access)
GET /orders?limit=25&offset=50

RESPONSE 200 OK
    data:  [...]
    pagination:
        total:   243
        limit:   25
        offset:  50
        hasMore: true

// Cursor-based pagination (stable under concurrent writes)
GET /orders?limit=25&cursor=eyJpZCI6MTAwfQ==

RESPONSE 200 OK
    data:  [...]
    pagination:
        limit:      25
        nextCursor: "eyJpZCI6MTI1fQ=="
        hasMore:    true

// Use cursor-based when:
// - Data changes frequently (inserts/deletes shift offsets)
// - Working with large datasets (offset-based degrades at high offsets)
// - Results must be stable across pages

// Use offset-based when:
// - Clients need "jump to page N" functionality
// - Data is relatively static
// - Simplicity is preferred
```

#### Filtering, Sorting, and Field Selection

Allow clients to request only the data they need:

```
// Filtering — query parameters for common filters
GET /orders?status=shipped&minTotal=50.00&createdAfter=2025-01-01

// Sorting — consistent sort parameter
GET /orders?sort=createdAt:desc
GET /orders?sort=status:asc,total:desc    // Multi-field sort

// Field selection — reduce payload size
GET /customers/abc-123?fields=id,name,email

// Combining all three
GET /orders?status=pending&sort=total:desc&fields=id,status,total&limit=10
```

#### Idempotency

Ensure that retrying a request produces the same result as the original:

```
// Idempotency key — client provides a unique key with each mutation
POST /payments
    Idempotency-Key: "pay-req-abc-123"
    Body: { orderId: "ord-456", amount: 99.99, currency: "USD" }

// Server-side handling
FUNCTION handlePayment(request)
    idempotencyKey = request.header("Idempotency-Key")

    // Check if this request was already processed
    existing = idempotencyStore.find(idempotencyKey)
    IF existing IS NOT NULL THEN
        RETURN existing.response   // Return the original response
    END IF

    // Process the payment
    result = paymentProcessor.charge(request.body)

    // Store the result keyed by idempotency key (with TTL)
    idempotencyStore.save(idempotencyKey, result, ttl = 24 HOURS)

    RETURN result
END FUNCTION

// Natural idempotency:
// GET    — always idempotent (safe, read-only)
// PUT    — idempotent by definition (full replacement)
// DELETE — idempotent (deleting twice = same result)
// POST   — NOT naturally idempotent → use idempotency keys
// PATCH  — context-dependent → use conditional requests (If-Match)
```

#### Versioning

Support API evolution without breaking existing consumers:

```
// Strategy 1: URI path versioning (most common, most explicit)
GET /v1/orders/{id}
GET /v2/orders/{id}

// Advantages: Clear, cache-friendly, easy to route
// Disadvantages: URL pollution, harder to deprecate

// Strategy 2: Header versioning
GET /orders/{id}
    Accept: application/vnd.myapi.v2+json

// Advantages: Clean URLs, supports content negotiation
// Disadvantages: Harder to test (can't just paste URL in browser)

// Strategy 3: Query parameter versioning
GET /orders/{id}?version=2

// Advantages: Simple to implement and test
// Disadvantages: Optional parameter can be forgotten

// Versioning rules
GUIDELINES
    1. Use semantic versioning — MAJOR.MINOR.PATCH
       Clients select by MAJOR version only

    2. Additive changes are NOT breaking:
       - Adding new fields to responses
       - Adding new optional query parameters
       - Adding new endpoints

    3. Breaking changes REQUIRE a new major version:
       - Removing or renaming fields
       - Changing field types
       - Changing URL structure
       - Altering error response formats

    4. Support at least N-1 versions concurrently
       Deprecate with sunset headers and migration guides

    5. Communicate deprecation clearly:
       Sunset: Sat, 01 Mar 2026 00:00:00 GMT
       Deprecation: true
       Link: <https://api.example.com/docs/migration/v1-to-v2>; rel="successor-version"
```

#### Rate Limiting

Protect APIs from abuse and ensure fair access:

```
// Rate limit response headers (standard practice)
RESPONSE HEADERS
    X-RateLimit-Limit:     100        // Max requests per window
    X-RateLimit-Remaining: 42         // Requests left in current window
    X-RateLimit-Reset:     1710288000 // Unix timestamp when window resets

// Rate limiting strategies
// Token Bucket — allows burst, replenishes at fixed rate
// Sliding Window — smooth limiting over rolling time window
// Fixed Window — simple counter reset at interval boundaries

FUNCTION rateLimitMiddleware(request)
    clientId = identifyClient(request)   // API key, IP, or user ID
    bucket   = rateLimiter.getBucket(clientId)

    IF bucket.tokens <= 0 THEN
        RETURN RESPONSE 429 Too Many Requests
            Retry-After: bucket.resetInSeconds
    END IF

    bucket.consume(1)
    setRateLimitHeaders(response, bucket)
    RETURN next(request)
END FUNCTION
```

#### HATEOAS (Hypermedia as the Engine of Application State)

Responses include links to related actions, making the API self-describing:

```
// Response with hypermedia links
GET /orders/ord-123

RESPONSE 200 OK
    data:
        id:     "ord-123"
        status: "confirmed"
        total:  149.99
    links:
        self:    { href: "/orders/ord-123", method: "GET" }
        cancel:  { href: "/orders/ord-123/cancel", method: "POST" }
        items:   { href: "/orders/ord-123/items", method: "GET" }
        payment: { href: "/orders/ord-123/payment", method: "GET" }
        customer:{ href: "/customers/cust-456", method: "GET" }

// If the order were already shipped, the "cancel" link would be absent
// — the API itself communicates what actions are currently valid
```

## Implementation

### API Definition and Documentation

Define the API contract before writing implementation code:

```
// OpenAPI / Swagger specification (pseudocode representation)
API SPECIFICATION OrderAPI
    VERSION: "2.0.0"
    BASE_PATH: "/v2"

    ENDPOINT GET /orders
        SUMMARY: "List orders with optional filters"
        PARAMETERS:
            status   : String OPTIONAL  [query] ENUM(pending, confirmed, shipped)
            limit    : Integer OPTIONAL [query] DEFAULT=25, MAX=100
            cursor   : String OPTIONAL  [query]
            sort     : String OPTIONAL  [query] DEFAULT="createdAt:desc"
        RESPONSES:
            200: PagedResult<OrderSummary>
            400: ProblemDetail     // Invalid parameters
            401: ProblemDetail     // Not authenticated
            429: ProblemDetail     // Rate limited

    ENDPOINT POST /orders
        SUMMARY: "Create a new order"
        HEADERS:
            Idempotency-Key : String REQUIRED
        BODY: CreateOrderRequest
        RESPONSES:
            201: OrderResponse     // Created successfully
            400: ProblemDetail     // Validation error
            409: ProblemDetail     // Duplicate idempotency key
            422: ProblemDetail     // Business rule violation

    ENDPOINT GET /orders/{orderId}
        SUMMARY: "Get order details"
        PARAMETERS:
            orderId : UUID REQUIRED [path]
            fields  : String OPTIONAL [query]
        RESPONSES:
            200: OrderResponse
            404: ProblemDetail

// Generate from this spec:
// - Client SDKs (any language)
// - Server stubs
// - Interactive documentation
// - Contract tests
```

### Request Validation

Validate all inbound data at the API boundary:

```
// Validation middleware — runs before business logic
FUNCTION validateRequest(schema, request)
    errors = []

    FOR EACH field IN schema.requiredFields
        IF request.body[field.name] IS NULL OR MISSING THEN
            errors.ADD(NEW FieldError(field.name, "is required"))
        END IF
    END FOR

    FOR EACH field IN schema.allFields
        value = request.body[field.name]
        IF value IS NOT NULL THEN
            IF NOT field.type.isValid(value) THEN
                errors.ADD(NEW FieldError(field.name, "must be " + field.type.description))
            END IF
            IF field.maxLength AND LENGTH(value) > field.maxLength THEN
                errors.ADD(NEW FieldError(field.name, "exceeds max length " + field.maxLength))
            END IF
            IF field.pattern AND NOT MATCHES(value, field.pattern) THEN
                errors.ADD(NEW FieldError(field.name, "does not match expected format"))
            END IF
        END IF
    END FOR

    IF errors IS NOT EMPTY THEN
        THROW ValidationError(errors)
    END IF
END FUNCTION

// Controller using validation
FUNCTION createOrderHandler(request)
    validateRequest(CreateOrderSchema, request)

    // If we reach here, the input is safe and well-formed
    command = NEW CreateOrderCommand(
        customerId = request.body.customerId,
        items      = request.body.items
    )
    result = orderService.createOrder(command)

    RETURN RESPONSE 201 Created
        Location: "/orders/" + result.id
        Body: mapToResponse(result)
END FUNCTION
```

### API Gateway Pattern

Centralize cross-cutting concerns for all APIs:

```
// API Gateway — single entry point for all clients
CLASS ApiGateway
    CONSTRUCTOR(config)
        this.router      = NEW Router(config.routes)
        this.authHandler  = NEW AuthenticationHandler(config.auth)
        this.rateLimiter  = NEW RateLimiter(config.limits)
        this.validator    = NEW RequestValidator()

    FUNCTION handleRequest(request)
        // 1. Rate limiting
        this.rateLimiter.check(request)

        // 2. Authentication
        identity = this.authHandler.authenticate(request)

        // 3. Authorization
        this.authHandler.authorize(identity, request.path, request.method)

        // 4. Request validation
        this.validator.validate(request)

        // 5. Route to appropriate service
        service = this.router.resolve(request.path)
        response = service.forward(request)

        // 6. Response transformation (if needed)
        RETURN transformResponse(response, request.apiVersion)
    END FUNCTION
END CLASS
```

## Project Structure

```
api/
│
├── specs/                              # API definitions (contract-first)
│   ├── openapi/                        # REST API specifications
│   │   ├── orders-v1.yaml
│   │   └── orders-v2.yaml
│   ├── protobuf/                       # gRPC service definitions
│   │   └── order_service.proto
│   └── graphql/                        # GraphQL schemas
│       └── schema.graphql
│
├── gateway/                            # API Gateway configuration
│   ├── routes/                         # Route definitions and mappings
│   ├── middleware/                      # Auth, rate limiting, logging
│   └── config/                         # Gateway configuration
│
├── endpoints/                          # API endpoint handlers
│   ├── rest/
│   │   ├── controllers/                # HTTP request handlers
│   │   ├── validators/                 # Request validation schemas
│   │   ├── mappers/                    # Domain ↔ API response mappers
│   │   └── middleware/                 # Endpoint-specific middleware
│   ├── grpc/
│   │   ├── services/                   # gRPC service implementations
│   │   └── interceptors/               # gRPC interceptors (auth, logging)
│   └── graphql/
│       ├── resolvers/                  # Query and mutation resolvers
│       └── dataloaders/                # Batch loading for N+1 prevention
│
├── shared/
│   ├── errors/                         # Standard error types and formatters
│   ├── pagination/                     # Pagination helpers
│   └── versioning/                     # Version negotiation utilities
│
├── docs/                               # Generated and hand-written API docs
│
└── tests/
    ├── contract/                       # Consumer-driven contract tests
    ├── integration/                    # Full API integration tests
    └── load/                           # Performance and load tests
```

## Benefits

1. **Developer Experience** — Consistent conventions, clear documentation, and predictable behavior reduce integration time for API consumers
2. **Evolvability** — Versioning strategies and additive-change policies allow APIs to evolve without breaking existing clients
3. **Performance** — Pagination, field selection, and appropriate protocol choices (gRPC for internal, REST for external) optimize for each context
4. **Reliability** — Idempotency keys, rate limiting, and standardized error handling make APIs resilient under real-world conditions
5. **Discoverability** — Contract-first design with generated documentation ensures APIs are self-describing and testable
6. **Interoperability** — Standard protocols and formats (HTTP, JSON, Protocol Buffers) enable integration across any language or platform

## Trade-offs

| Advantage                               | Consideration                                                          |
| --------------------------------------- | ---------------------------------------------------------------------- |
| Contract-first ensures early consensus  | Requires up-front design investment before implementation              |
| REST is universally understood          | Can lead to over-fetching / under-fetching compared to GraphQL         |
| GraphQL gives clients query flexibility | Shifts complexity to the server (performance, authorization per field) |
| gRPC is fast and type-safe              | Not browser-native; requires proxying for web clients                  |
| Versioning enables evolution            | Multiple live versions increase maintenance and testing burden         |
| Rate limiting protects services         | Legitimate high-volume clients may require special accommodation       |

## When to Use

✅ **Good fit for:**

- Any system exposing functionality to external or internal consumers
- Microservices architectures where clear service contracts are critical
- Public APIs where backward compatibility is a business requirement
- Systems where multiple client types (web, mobile, CLI) consume the same backend
- Teams adopting contract-first development workflows

❌ **Not ideal for:**

- Purely internal monolithic applications with tightly coupled modules
- Early prototypes where the domain model is still highly unstable (design the API as the model stabilizes)
- Batch file-transfer integrations with no interactive request/response pattern

## References

- [Azure Architecture Center — API Design Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Azure Architecture Center — API Design for Microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/api-design)
- [RFC 9457 — Problem Details for HTTP APIs (2023)](https://www.rfc-editor.org/rfc/rfc9457)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Roy Fielding — Architectural Styles and the Design of Network-based Software Architectures (2000)](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)
- [GraphQL Specification](https://spec.graphql.org/)
- [gRPC Documentation](https://grpc.io/docs/)
