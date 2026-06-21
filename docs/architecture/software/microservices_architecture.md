# Microservices Architecture

## Overview

**Microservices Architecture** is an approach to building a single application as a suite of small, independently deployable services, each running in its own process and communicating through lightweight mechanisms — typically HTTP APIs or asynchronous messaging. The term was popularized by James Lewis and Martin Fowler in 2014, though the ideas trace back to Unix design principles and early service-oriented approaches at Amazon and Netflix.

Each service is organized around a business capability, owns its data, and can be developed, deployed, and scaled independently by a small cross-functional team.

Key principles:

- **Single Responsibility** — Each service focuses on one business capability and does it well
- **Autonomous Deployment** — Services are independently deployable without coordinating releases across the system
- **Decentralized Data** — Each service owns its data store; no shared database across services
- **Smart Endpoints, Dumb Pipes** — Business logic lives in the services, not in the communication infrastructure
- **Design for Failure** — Services assume that any dependency can fail at any time and handle it gracefully
- **Evolutionary Design** — Services are designed to be replaceable, not just maintainable

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Microservices Architecture                               │
│                                                                             │
│   Client Requests                                                           │
│       │                                                                     │
│       ▼                                                                     │
│   ┌──────────────────────────────────────────────────────┐                  │
│   │                   API Gateway                         │                  │
│   │   (routing, auth, rate limiting, load balancing)      │                  │
│   └──────┬──────────────┬──────────────┬─────────────────┘                  │
│          │              │              │                                     │
│          ▼              ▼              ▼                                     │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐                             │
│   │  Order     │ │  Inventory │ │  Payment   │                              │
│   │  Service   │ │  Service   │ │  Service   │   ... more services          │
│   │            │ │            │ │            │                               │
│   │  ┌──────┐ │ │  ┌──────┐ │ │  ┌──────┐ │                               │
│   │  │ DB   │ │ │  │ DB   │ │ │  │ DB   │ │   ← Database per Service      │
│   │  └──────┘ │ │  └──────┘ │ │  └──────┘ │                               │
│   └─────┬──────┘ └─────┬──────┘ └─────┬──────┘                             │
│         │              │              │                                      │
│         └──────────────┼──────────────┘                                     │
│                        │                                                    │
│                        ▼                                                    │
│              ┌──────────────────┐                                           │
│              │   Message Broker  │  (async events between services)          │
│              │   (e.g. Kafka)    │                                           │
│              └──────────────────┘                                           │
│                                                                             │
│   Cross-Cutting: ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐  │
│                   │ Service     │ │ Distributed  │ │ Centralized         │  │
│                   │ Registry   │ │ Tracing      │ │ Logging & Metrics   │  │
│                   └─────────────┘ └──────────────┘ └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Service Decomposition

Deciding where to draw service boundaries is the most critical design decision. Two primary strategies:

**Decompose by Business Capability** — Align services with organizational capabilities (what the business does):

```
┌─────────────────────────────────────────────────────────┐
│              Business Capabilities                       │
├─────────────────┬─────────────────┬─────────────────────┤
│  Order Mgmt     │  Inventory      │  Customer Mgmt      │
│  ┌───────────┐  │  ┌───────────┐  │  ┌───────────────┐  │
│  │ Order     │  │  │ Inventory │  │  │  Customer     │  │
│  │ Service   │  │  │ Service   │  │  │  Service      │  │
│  └───────────┘  │  └───────────┘  │  └───────────────┘  │
├─────────────────┼─────────────────┼─────────────────────┤
│  Shipping       │  Billing        │  Notifications       │
│  ┌───────────┐  │  ┌───────────┐  │  ┌───────────────┐  │
│  │ Shipping  │  │  │ Billing   │  │  │ Notification  │  │
│  │ Service   │  │  │ Service   │  │  │ Service       │  │
│  └───────────┘  │  └───────────┘  │  └───────────────┘  │
└─────────────────┴─────────────────┴─────────────────────┘
```

**Decompose by Subdomain (DDD)** — Use bounded contexts from Domain-Driven Design to define service boundaries. See [Domain-Driven Design (DDD)](domain_driven_design.md).

### Database per Service

Each service owns its data store — this is non-negotiable for true independence:

```
// Anti-pattern: Shared database
// Multiple services reading/writing the same tables
// → Tight coupling, schema changes break multiple services

// Correct: Database per Service
// Each service has exclusive ownership of its data
// Other services access data only through the service's API

INTERFACE OrderRepository
    FUNCTION save(order : Order) -> Void
    FUNCTION findById(id : UUID) -> Order OR NULL
    FUNCTION findByCustomer(customerId : UUID) -> List<Order>
END INTERFACE

// Service A cannot directly query Service B's database
// Instead, Service A calls Service B's API or subscribes to its events
```

### Communication Patterns

Services communicate through two primary mechanisms:

#### Synchronous — Request/Response

```
// Service Gateway — Abstracts inter-service HTTP calls
CLASS ServiceGateway
    PROPERTIES
        registry    : ServiceRegistry
        circuitBreaker : CircuitBreaker
        retryPolicy : RetryPolicy

    FUNCTION call(serviceName : String,
                  method : HTTPMethod,
                  path : String,
                  body : Any OR NULL) -> ServiceResponse

        // 1. Discover service instance
        endpoint = registry.resolve(serviceName)

        // 2. Execute with circuit breaker and retry
        response = circuitBreaker.execute(serviceName,
            FUNCTION() ->
                retryPolicy.execute(
                    FUNCTION() ->
                        httpClient.request(
                            url    = endpoint + path,
                            method = method,
                            body   = body
                        )
                )
        )

        RETURN response
END CLASS
```

#### Asynchronous — Event-Driven

```
// Event Publisher — Publishes domain events to a message broker
CLASS EventPublisher
    PROPERTIES
        broker : MessageBroker

    FUNCTION publish(event : DomainEvent) -> Void
        message = {
            eventId   : generateUUID(),
            eventType : event.type(),
            timestamp : NOW(),
            source    : SERVICE_NAME,
            payload   : serialize(event)
        }
        broker.publish(topic = event.type(), message = message)
END CLASS

// Event Consumer — Subscribes to events from other services
CLASS EventConsumer
    PROPERTIES
        broker  : MessageBroker
        handler : EventHandler

    FUNCTION subscribe(eventType : String) -> Void
        broker.subscribe(
            topic   = eventType,
            group   = SERVICE_NAME,  // Consumer group for load balancing
            handler = FUNCTION(message) ->
                event = deserialize(message.payload)

                // Idempotency check — events may be delivered more than once
                IF alreadyProcessed(message.eventId) THEN
                    RETURN
                END IF

                handler.handle(event)
                markProcessed(message.eventId)
        )
END CLASS
```

### Resilience Patterns

#### Circuit Breaker

Prevents cascading failures when a downstream service is unavailable:

```
// Circuit Breaker — Stops calling a failing service to allow recovery
CLASS CircuitBreaker
    PROPERTIES
        state           : CircuitState = CLOSED   // CLOSED, OPEN, HALF_OPEN
        failureCount    : Integer = 0
        failureThreshold : Integer
        resetTimeout    : Duration
        lastFailureTime : DateTime OR NULL

    FUNCTION execute(serviceName : String, action : Function) -> Any
        MATCH state
            CASE OPEN:
                IF NOW() - lastFailureTime > resetTimeout THEN
                    state = HALF_OPEN
                    // Fall through to try one request
                ELSE
                    THROW CircuitOpenException(
                        "Circuit open for " + serviceName
                    )
                END IF

            CASE HALF_OPEN:
                TRY
                    result = action()
                    // Success — reset circuit
                    state = CLOSED
                    failureCount = 0
                    RETURN result
                CATCH error
                    // Still failing — reopen circuit
                    state = OPEN
                    lastFailureTime = NOW()
                    THROW error
                END TRY

            CASE CLOSED:
                TRY
                    result = action()
                    failureCount = 0
                    RETURN result
                CATCH error
                    failureCount = failureCount + 1
                    IF failureCount >= failureThreshold THEN
                        state = OPEN
                        lastFailureTime = NOW()
                    END IF
                    THROW error
                END TRY
        END MATCH
END CLASS
```

#### Saga Pattern

Manages distributed transactions across services without two-phase commit:

```
// Saga Orchestrator — Coordinates a multi-service business transaction
CLASS OrderSaga
    PROPERTIES
        steps : List<SagaStep>

    CONSTRUCTOR()
        steps = [
            NEW SagaStep(
                name       = "ReserveInventory",
                execute    = inventoryService.reserve,
                compensate = inventoryService.releaseReservation
            ),
            NEW SagaStep(
                name       = "ProcessPayment",
                execute    = paymentService.charge,
                compensate = paymentService.refund
            ),
            NEW SagaStep(
                name       = "ConfirmShipment",
                execute    = shippingService.schedule,
                compensate = shippingService.cancel
            )
        ]

    FUNCTION execute(context : OrderContext) -> SagaResult
        completedSteps = EMPTY LIST

        FOR EACH step IN steps
            TRY
                step.execute(context)
                completedSteps.ADD(step)
            CATCH error
                // Compensate all completed steps in reverse order
                FOR EACH completedStep IN completedSteps.REVERSED()
                    TRY
                        completedStep.compensate(context)
                    CATCH compensationError
                        LOG.error("Compensation failed for " +
                                  completedStep.name, compensationError)
                        // Record for manual intervention
                        alertOperations(completedStep, compensationError)
                    END TRY
                END FOR
                RETURN SagaResult.FAILED(step.name, error)
            END TRY
        END FOR

        RETURN SagaResult.COMPLETED()

DATA SagaStep
    name       : String
    execute    : Function<OrderContext, Void>
    compensate : Function<OrderContext, Void>
END DATA
```

### API Gateway

The API Gateway is the single entry point for all client requests:

```
// API Gateway — Routes, authenticates, and rate-limits client requests
CLASS APIGateway
    PROPERTIES
        routeTable    : Map<String, ServiceRoute>
        authenticator : Authenticator
        rateLimiter   : RateLimiter
        loadBalancer  : LoadBalancer

    FUNCTION handleRequest(request : HTTPRequest) -> HTTPResponse
        // 1. Authenticate
        identity = authenticator.authenticate(request)
        IF identity IS NULL THEN
            RETURN HTTPResponse(status = 401, body = "Unauthorized")
        END IF

        // 2. Rate limit
        IF NOT rateLimiter.allow(identity, request.path) THEN
            RETURN HTTPResponse(status = 429, body = "Too Many Requests")
        END IF

        // 3. Route to service
        route = routeTable.match(request.method, request.path)
        IF route IS NULL THEN
            RETURN HTTPResponse(status = 404, body = "Not Found")
        END IF

        // 4. Load balance across service instances
        instance = loadBalancer.selectInstance(route.serviceName)

        // 5. Forward request (with correlation ID for tracing)
        request.headers.SET("X-Correlation-ID", generateCorrelationId())
        request.headers.SET("X-Authenticated-User", identity.userId)

        RETURN instance.forward(request)

DATA ServiceRoute
    pathPattern : String       // e.g. "/api/orders/**"
    serviceName : String       // e.g. "order-service"
    methods     : List<HTTPMethod>
    policies    : List<Policy> // rate limits, CORS, etc.
END DATA
```

### Service Discovery

Services need to find each other without hardcoded addresses:

```
// Service Registry — Central directory of service instances
INTERFACE ServiceRegistry
    FUNCTION register(instance : ServiceInstance) -> Void
    FUNCTION deregister(instanceId : String) -> Void
    FUNCTION resolve(serviceName : String) -> ServiceEndpoint
    FUNCTION resolveAll(serviceName : String) -> List<ServiceEndpoint>
END INTERFACE

// Health-Check Registration — Services register and send heartbeats
CLASS ServiceInstance
    PROPERTIES
        instanceId  : String
        serviceName : String
        host        : String
        port        : Integer
        metadata    : Map<String, String>  // version, region, etc.

    FUNCTION startHeartbeat(registry : ServiceRegistry, interval : Duration)
        EVERY interval DO
            TRY
                registry.sendHeartbeat(instanceId)
            CATCH error
                LOG.warn("Heartbeat failed, re-registering")
                registry.register(THIS)
            END TRY
        END EVERY
END CLASS
```

### Observability

Observability is critical in distributed systems — you cannot debug what you cannot see:

#### Distributed Tracing

```
// Tracing Middleware — Propagates trace context across service boundaries
CLASS TracingMiddleware
    PROPERTIES
        tracer : DistributedTracer

    FUNCTION handle(request : HTTPRequest, next : Handler) -> HTTPResponse
        // Extract or create trace context
        traceId = request.headers.GET("X-Trace-ID") OR generateTraceId()
        spanId  = generateSpanId()
        parentSpanId = request.headers.GET("X-Span-ID")

        span = tracer.startSpan(
            name     = request.method + " " + request.path,
            traceId  = traceId,
            spanId   = spanId,
            parentId = parentSpanId
        )

        TRY
            // Inject trace context into downstream calls
            request.headers.SET("X-Trace-ID", traceId)
            request.headers.SET("X-Span-ID", spanId)

            response = next.handle(request)

            span.setStatus(response.status)
            RETURN response
        CATCH error
            span.setError(error)
            THROW error
        FINALLY
            span.finish()
        END TRY
END CLASS
```

#### Health Checks

```
// Health Check Endpoint — Exposes service and dependency health
CLASS HealthCheckEndpoint
    PROPERTIES
        checks : List<HealthCheck>

    ENDPOINT GET "/health"
        results = EMPTY MAP
        overallStatus = "healthy"

        FOR EACH check IN checks
            TRY
                check.execute()
                results.PUT(check.name, { status: "healthy" })
            CATCH error
                results.PUT(check.name, {
                    status: "unhealthy",
                    error: error.message
                })
                overallStatus = "unhealthy"
            END TRY
        END FOR

        RETURN {
            status      : overallStatus,
            serviceName : SERVICE_NAME,
            version     : SERVICE_VERSION,
            uptime      : getUptime(),
            checks      : results
        }

INTERFACE HealthCheck
    PROPERTIES
        name : String
    FUNCTION execute() -> Void  // Throws on failure
END INTERFACE
```

## Project Structure

```
system/
│
├── api-gateway/                    # API Gateway / BFF
│   ├── routes/
│   ├── middleware/
│   └── config/
│
├── services/
│   ├── order-service/              # Each service is a separate deployable
│   │   ├── api/                    # HTTP endpoints
│   │   ├── domain/                 # Business logic and entities
│   │   ├── events/                 # Event publishers and consumers
│   │   ├── repository/             # Data access
│   │   ├── config/
│   │   └── tests/
│   │
│   ├── inventory-service/
│   │   ├── api/
│   │   ├── domain/
│   │   ├── events/
│   │   ├── repository/
│   │   └── tests/
│   │
│   ├── payment-service/
│   │   └── ...
│   │
│   └── notification-service/
│       └── ...
│
├── shared/
│   ├── event-contracts/            # Shared event schemas / interfaces
│   └── observability/              # Common tracing, logging, metrics
│
├── infrastructure/
│   ├── service-registry/
│   ├── message-broker/
│   ├── monitoring/                 # Dashboards, alerting
│   └── deployment/                 # Container orchestration configs
│
└── tests/
    ├── contract/                   # Consumer-driven contract tests
    └── integration/                # End-to-end integration tests
```

## Key Design Considerations

### Data Consistency

- **Embrace eventual consistency** — Distributed transactions (two-phase commit) do not scale; use sagas and event-driven state synchronization instead
- **Idempotent operations** — Design all operations to be safely retryable since messages may be delivered more than once
- **Outbox pattern** — Atomically commit domain changes and outgoing events to the same database, then publish events asynchronously

### Service Granularity

- **Too fine-grained** — Excessive inter-service communication, operational overhead, distributed debugging complexity
- **Too coarse-grained** — Loses independent deployability, teams step on each other, harder to scale selectively
- **Right-sized** — A service should be ownable by one small team (Amazon's "two-pizza team"), aligned to a single bounded context

### Testing Strategy

| Level             | Scope                                   | Speed  |
| ----------------- | --------------------------------------- | ------ |
| Unit Tests        | Individual service logic                | Fast   |
| Integration Tests | Service + its database and dependencies | Medium |
| Contract Tests    | API contracts between services          | Medium |
| Component Tests   | Full service in isolation (mocked deps) | Medium |
| End-to-End Tests  | Multiple services in a staging env      | Slow   |

## Benefits

1. **Independent Deployment** — Deploy services without coordinating with other teams or redeploying the entire system
2. **Technology Heterogeneity** — Each service can use the best language, framework, and database for its problem domain
3. **Resilience** — A failure in one service does not cascade to bring down the whole system (with proper circuit breakers)
4. **Scalability** — Scale individual services horizontally based on their specific load, not the entire application
5. **Team Autonomy** — Small teams own services end-to-end ("you build it, you run it"), reducing coordination overhead
6. **Evolutionary Architecture** — Replace or rewrite individual services without affecting the rest of the system

## Trade-offs

| Advantage                          | Consideration                                                 |
| ---------------------------------- | ------------------------------------------------------------- |
| Independent deployment and scaling | Distributed systems complexity (network, latency, failures)   |
| Technology freedom per service     | Operational overhead of managing many services                |
| Team autonomy and ownership        | Data consistency is eventually consistent, not ACID           |
| Fault isolation and resilience     | Debugging distributed requests requires sophisticated tooling |
| Aligned with business capabilities | Service boundaries are hard to get right upfront              |
| Evolutionary, replaceable design   | Integration testing across services is fundamentally harder   |

## When to Use

✅ **Good fit for:**

- Large applications with multiple teams needing independent delivery cadences
- Systems requiring selective scaling of high-traffic components
- Organizations practicing DevOps where teams own services end-to-end
- Applications with distinct business domains that naturally separate
- Long-lived products that need evolutionary, incremental modernization

❌ **Not ideal for:**

- Small applications or teams — the operational overhead outweighs the benefits
- Greenfield projects where domain boundaries are not yet understood (start with a modular monolith)
- Applications requiring strong transactional consistency across many entities
- Teams without infrastructure automation (CI/CD, container orchestration, monitoring)
- Projects where rapid prototyping is the priority

## References

- [Microservices — James Lewis & Martin Fowler (2014)](https://martinfowler.com/articles/microservices.html)
- [Microservice Architecture Pattern — Chris Richardson (microservices.io)](https://microservices.io/patterns/microservices.html)
- [Design a Microservices Architecture — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/)
- [Using Domain Analysis to Model Microservices — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
- [Building Microservices — Sam Newman (O'Reilly, 2nd ed.)](https://samnewman.io/books/building_microservices_2nd_edition/)
- [Microservices Patterns — Chris Richardson (Manning)](https://microservices.io/book)
- [Saga Pattern — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)
