# Serverless Architecture

## Overview

**Serverless Architecture** is a cloud-native design approach where the cloud provider dynamically manages the allocation and provisioning of servers. Despite the name, servers still exist — but the developer does not provision, manage, or think about them. The two core building blocks are **Backend as a Service (BaaS)** — using third-party services for authentication, databases, and storage — and **Functions as a Service (FaaS)** — running application code in ephemeral, event-triggered containers fully managed by the provider.

The concept gained momentum after AWS Lambda launched in 2014 and was articulated in detail by Mike Roberts in his 2018 article on martinfowler.com. Today, every major cloud provider offers a Serverless platform (AWS Lambda, Azure Functions, Google Cloud Functions).

Key principles:

- **No Server Management** — The provider handles provisioning, patching, scaling, and availability of compute infrastructure
- **Event-Driven Execution** — Functions are triggered by events (HTTP requests, message queue messages, file uploads, schedules, database changes)
- **Pay-per-Use** — You pay only for the compute time consumed (often down to 100ms granularity), not for idle capacity
- **Automatic Scaling** — The platform scales from zero to thousands of concurrent executions without configuration
- **Stateless Functions** — Function instances are ephemeral; any persistent state must be stored externally
- **Managed Services Composition** — Applications are composed by connecting managed services (queues, databases, API gateways) rather than building everything from scratch

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Serverless Architecture                                   │
│                                                                             │
│   Event Sources                                                             │
│       │                                                                     │
│       ├─── HTTP Request ──────────────┐                                     │
│       ├─── Message Queue ─────────────┤                                     │
│       ├─── File Upload ───────────────┤                                     │
│       ├─── Schedule (Cron) ───────────┤                                     │
│       └─── Database Change ───────────┤                                     │
│                                       ▼                                     │
│                             ┌──────────────────┐                            │
│                             │   API Gateway /   │                            │
│                             │   Event Router    │                            │
│                             └────────┬─────────┘                            │
│                                      │                                      │
│                    ┌─────────────────┼─────────────────┐                    │
│                    ▼                 ▼                  ▼                    │
│           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│           │  Function A   │  │  Function B   │  │  Function C   │            │
│           │  (e.g. API    │  │  (e.g. Event  │  │  (e.g. Batch  │            │
│           │   handler)    │  │   processor)  │  │   job)        │            │
│           └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│                  │                 │                  │                      │
│                  ▼                 ▼                  ▼                      │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    Managed Backend Services (BaaS)               │       │
│   │                                                                  │       │
│   │   ┌─────────┐  ┌──────────┐  ┌────────────┐  ┌─────────────┐  │       │
│   │   │ Database │  │  Object  │  │  Message   │  │  Auth / IAM │  │       │
│   │   │ (NoSQL / │  │  Storage │  │  Queue     │  │  Service    │  │       │
│   │   │  SQL)    │  │          │  │            │  │             │  │       │
│   │   └─────────┘  └──────────┘  └────────────┘  └─────────────┘  │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│   Cross-Cutting: ┌─────────────┐  ┌──────────────┐  ┌────────────────┐     │
│                   │ Distributed │  │  Cost         │  │  Security /    │     │
│                   │ Tracing     │  │  Monitoring   │  │  IAM Policies  │     │
│                   └─────────────┘  └──────────────┘  └────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Functions as a Service (FaaS)

The defining execution model of Serverless — stateless functions triggered by events:

```
// FaaS Function — Stateless, event-triggered, ephemeral
FUNCTION handleOrderCreated(event : OrderCreatedEvent, context : FunctionContext) -> Response

    // 1. Parse event payload
    orderId    = event.payload.orderId
    customerId = event.payload.customerId
    items      = event.payload.items

    // 2. Execute business logic
    invoice = generateInvoice(orderId, customerId, items)

    // 3. Interact with managed services (BaaS)
    database.put("invoices", invoice)
    emailService.send(
        to      = invoice.customerEmail,
        subject = "Invoice for Order " + orderId,
        body    = formatInvoice(invoice)
    )

    // 4. Return result
    RETURN Response(status = 200, body = { invoiceId: invoice.id })

// Key constraints:
// - Function must complete within execution timeout (typically 5–15 minutes)
// - No guarantee of local state between invocations
// - Horizontal scaling is automatic and managed by the platform
END FUNCTION
```

### Backend as a Service (BaaS)

Third-party managed services that replace server-side components:

```
// BaaS Composition — Application wires together managed services
CONFIGURATION ServerlessApplication
    authentication = ManagedAuthService(
        provider = "cloud-auth",
        features = [social_login, MFA, JWT_tokens]
    )

    database = ManagedDatabase(
        type          = "document-store",
        scalingMode   = "on-demand",
        backupPolicy  = "continuous"
    )

    storage = ManagedObjectStorage(
        bucket  = "uploads",
        access  = "private",
        events  = [ON_CREATE -> triggerFunction("process-upload")]
    )

    messaging = ManagedQueue(
        type       = "FIFO",
        deadLetter = TRUE,
        retention  = "7 days"
    )

    // Client applications can access some BaaS services directly
    // (with appropriate security rules), reducing need for custom backend code
END CONFIGURATION
```

### Event Sources and Triggers

Functions are invoked in response to events from many different sources:

```
// Event Source Mapping — Connecting triggers to functions

// 1. HTTP API — Synchronous request/response
TRIGGER HttpTrigger
    source   = API_GATEWAY
    method   = "POST"
    path     = "/api/orders"
    function = "createOrder"
END TRIGGER

// 2. Message Queue — Asynchronous event processing
TRIGGER QueueTrigger
    source    = MESSAGE_QUEUE
    queueName = "order-events"
    batchSize = 10
    function  = "processOrderBatch"
END TRIGGER

// 3. Storage Event — File upload processing
TRIGGER StorageTrigger
    source   = OBJECT_STORAGE
    bucket   = "uploads"
    event    = "object:created"
    prefix   = "images/"
    function = "generateThumbnail"
END TRIGGER

// 4. Schedule — Time-based execution
TRIGGER ScheduleTrigger
    source   = SCHEDULER
    schedule = "CRON(0 */6 * * *)"   // Every 6 hours
    function = "generateReport"
END TRIGGER

// 5. Database Stream — Change data capture
TRIGGER DatabaseTrigger
    source     = DATABASE_STREAM
    table      = "orders"
    events     = [INSERT, MODIFY]
    function   = "syncOrderToAnalytics"
END TRIGGER
```

## Implementation

### API-Driven Pattern

The most common Serverless pattern — replacing a traditional web server with API Gateway + functions:

```
// API Gateway routes HTTP requests to individual functions
CLASS OrderApi

    ENDPOINT POST "/orders"
    FUNCTION createOrder(request : HTTPRequest) -> HTTPResponse
        // Validate input
        body = parseAndValidate(request.body, OrderCreateSchema)
        IF body.errors IS NOT EMPTY THEN
            RETURN HTTPResponse(status = 400, body = body.errors)
        END IF

        // Create order
        order = NEW Order(
            id         = generateUUID(),
            customerId = request.auth.userId,
            items      = body.items,
            status     = "PENDING",
            createdAt  = NOW()
        )

        // Persist to managed database
        database.put("orders", order)

        // Emit event to trigger downstream processing
        eventBus.publish("order.created", {
            orderId    : order.id,
            customerId : order.customerId,
            items      : order.items
        })

        RETURN HTTPResponse(status = 201, body = order)

    ENDPOINT GET "/orders/{id}"
    FUNCTION getOrder(request : HTTPRequest) -> HTTPResponse
        order = database.get("orders", request.pathParams.id)
        IF order IS NULL THEN
            RETURN HTTPResponse(status = 404)
        END IF
        RETURN HTTPResponse(status = 200, body = order)
END CLASS
```

### Event Processing Pattern

Asynchronous processing pipelines triggered by events:

```
// Event Processing Pipeline — Chain of functions connected via events

// Step 1: Order placed → validate inventory
FUNCTION validateInventory(event : OrderCreatedEvent) -> Void
    FOR EACH item IN event.items
        stock = inventoryDB.get(item.productId)
        IF stock.available < item.quantity THEN
            eventBus.publish("order.validation.failed", {
                orderId : event.orderId,
                reason  : "Insufficient stock for " + item.productId
            })
            RETURN
        END IF
    END FOR

    // Reserve inventory
    FOR EACH item IN event.items
        inventoryDB.decrement(item.productId, "available", item.quantity)
        inventoryDB.increment(item.productId, "reserved", item.quantity)
    END FOR

    eventBus.publish("inventory.reserved", {
        orderId : event.orderId,
        items   : event.items
    })
END FUNCTION

// Step 2: Inventory reserved → process payment
FUNCTION processPayment(event : InventoryReservedEvent) -> Void
    order = database.get("orders", event.orderId)
    paymentResult = paymentGateway.charge(order.customerId, order.total)

    IF paymentResult.success THEN
        eventBus.publish("payment.completed", {
            orderId   : event.orderId,
            paymentId : paymentResult.id
        })
    ELSE
        // Compensate — release reserved inventory
        eventBus.publish("payment.failed", {
            orderId : event.orderId,
            reason  : paymentResult.errorMessage
        })
    END IF
END FUNCTION

// Step 3: Payment completed → confirm order
FUNCTION confirmOrder(event : PaymentCompletedEvent) -> Void
    database.update("orders", event.orderId, { status: "CONFIRMED" })

    emailService.send(
        to      = lookupCustomerEmail(event.orderId),
        subject = "Order Confirmed",
        body    = buildConfirmationEmail(event.orderId)
    )
END FUNCTION
```

### Scheduled Task Pattern

```
// Scheduled Function — Runs on a cron schedule

FUNCTION dailyReportGenerator(event : ScheduleEvent) -> Void
    yesterday = TODAY().subtract(1, "day")

    // Query analytics from managed data store
    metrics = analyticsDB.query(
        "SELECT COUNT(*) as orders, SUM(total) as revenue " +
        "FROM orders WHERE date = ?", [yesterday]
    )

    // Generate and store report
    report = formatDailyReport(yesterday, metrics)
    storage.put("reports/" + yesterday + ".pdf", report)

    // Notify stakeholders
    notificationService.send(
        channel = "daily-metrics",
        message = "Daily report ready: " + metrics.orders + " orders, " +
                  formatCurrency(metrics.revenue) + " revenue"
    )
END FUNCTION
```

### Cold Start Mitigation

Cold starts — the latency when a new function instance is initialized — are a key concern:

```
// Cold Start Strategies

// 1. Keep functions lightweight
//    - Minimize dependencies and package size
//    - Use lazy initialization for expensive resources

CLASS FunctionHandler
    PROPERTIES
        dbConnection : DatabaseConnection OR NULL = NULL  // Lazy init

    FUNCTION getConnection() -> DatabaseConnection
        // Reuse connection across warm invocations
        IF dbConnection IS NULL OR NOT dbConnection.isAlive() THEN
            dbConnection = createDatabaseConnection()
        END IF
        RETURN dbConnection

// 2. Provisioned concurrency (platform feature)
//    Pre-warm a minimum number of function instances
CONFIGURATION CriticalApiFunction
    runtime              = "managed"
    memory               = 512
    timeout              = 30
    provisionedConcurrency = 5   // Always keep 5 warm instances
END CONFIGURATION

// 3. Architecture-level mitigation
//    - Use asynchronous patterns where latency tolerance is higher
//    - Reserve synchronous HTTP-triggered functions for user-facing APIs
//    - Use lighter runtimes for latency-sensitive functions
```

## Project Structure

```
serverless-app/
│
├── functions/
│   ├── api/                        # HTTP-triggered functions
│   │   ├── create-order/
│   │   ├── get-order/
│   │   └── list-orders/
│   │
│   ├── events/                     # Event-triggered functions
│   │   ├── process-order/
│   │   ├── send-notification/
│   │   └── sync-analytics/
│   │
│   ├── scheduled/                  # Cron-triggered functions
│   │   ├── daily-report/
│   │   └── cleanup-expired/
│   │
│   └── shared/                     # Shared utilities across functions
│       ├── validation/
│       ├── database/
│       └── auth/
│
├── infrastructure/                 # Infrastructure as Code
│   ├── api-gateway/
│   ├── database/
│   ├── queues/
│   ├── storage/
│   └── iam/                        # Permissions and policies
│
├── config/
│   ├── serverless-config           # Function definitions and triggers
│   └── environment/
│
└── tests/
    ├── unit/                       # Function logic tests
    ├── integration/                # Tests against cloud services
    └── e2e/                        # End-to-end API tests
```

## Key Design Considerations

### Security

- **Principle of Least Privilege** — Each function should have its own IAM role with the minimum permissions required
- **Environment Secrets** — Use managed secret stores, not environment variables, for sensitive credentials
- **Input Validation** — Validate all event payloads; do not trust event sources implicitly
- **API Authentication** — Use API Gateway authorizers (JWT, OAuth, API keys) rather than custom auth logic in functions

### Cost Optimization

- **Right-size memory allocation** — Memory also controls CPU proportion; profile to find the optimal setting
- **Minimize execution duration** — Pay-per-100ms means faster code directly reduces cost
- **Use asynchronous patterns** — Queue-based processing avoids blocking functions waiting for downstream services
- **Reserved concurrency** — Set limits to prevent runaway costs from accidental infinite loops or traffic spikes
- **Caching** — Use API Gateway response caching for read-heavy endpoints to avoid function invocations

### Observability

- **Structured logging** — Log in JSON format with correlation IDs for end-to-end tracing across function chains
- **Distributed tracing** — Use cloud-native tracing (e.g., X-Ray) to visualize request flows across functions and services
- **Custom metrics** — Publish business metrics (orders processed, errors by type) alongside platform metrics
- **Alerting** — Set alarms on error rate, duration percentiles, throttling, and dead-letter queue depth

## Benefits

1. **Zero Server Management** — No operating systems to patch, no capacity to plan, no servers to provision
2. **Automatic Scaling** — Handles zero to peak traffic transparently, including scaling back to zero when idle
3. **Pay-per-Use** — Costs directly track actual usage; no charge for idle time
4. **Faster Time to Market** — Less infrastructure work means faster deployment of new features and experiments
5. **Built-in High Availability** — Cloud providers run functions across multiple availability zones by default
6. **Reduced Operational Burden** — Deployment is uploading code; no deployment pipelines for infrastructure

## Trade-offs

| Advantage                              | Consideration                                                         |
| -------------------------------------- | --------------------------------------------------------------------- |
| No server provisioning or management   | Vendor lock-in to a specific cloud platform                           |
| Automatic, transparent scaling         | Cold starts add latency for infrequent or latency-sensitive workloads |
| Pay only for actual compute consumed   | Cost can be unpredictable under sustained high throughput             |
| Rapid deployment and experimentation   | Debugging distributed function chains is harder than monoliths        |
| Managed high availability and patching | Execution duration limits (typically 5–15 min max)                    |
| Composable with managed services       | Local state is ephemeral; requires external persistence               |

## When to Use

✅ **Good fit for:**

- Event-driven workloads — file processing, message handling, webhook receivers
- APIs with variable or unpredictable traffic patterns (including long idle periods)
- Scheduled tasks and batch jobs that run periodically
- Rapid prototyping and experimentation where infrastructure speed matters
- Microservice backends where individual functions map to individual operations
- Cost-sensitive applications with low or spiky traffic

❌ **Not ideal for:**

- Long-running processes that exceed function timeout limits
- Workloads requiring persistent in-memory state (e.g., WebSocket servers, game servers)
- Latency-critical applications that cannot tolerate cold start overhead
- High-throughput, steady-load applications where reserved servers are cheaper
- Applications requiring fine-grained control over the runtime environment
- Teams with compliance requirements restricting third-party cloud infrastructure

## References

- [Serverless Architectures — Mike Roberts (martinfowler.com, 2018)](https://martinfowler.com/articles/serverless.html)
- [Serverless Computing — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/serverless/)
- [Azure Functions Overview — Microsoft](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Serverless Application Lens — AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html)
- [The Twelve-Factor App](https://12factor.net/)
