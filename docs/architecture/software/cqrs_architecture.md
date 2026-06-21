# CQRS (Command Query Responsibility Segregation)

## Overview

**CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates the read (query) and write (command) operations of a system into distinct models. Instead of using a single data model for both reading and writing — as in traditional CRUD — CQRS uses one model optimized for updates and another optimized for reads.

This separation originates from Bertrand Meyer's **Command-Query Separation (CQS)** principle at the method level, elevated to an architectural pattern by Greg Young:

- **Command** — An operation that changes state but returns no data (a write/mutation)
- **Query** — An operation that returns data but does not change state (a read)

Key principles:

- **Separate Models** — The write model (command side) and read model (query side) are independently designed and optimized
- **Task-Based Commands** — Commands represent user intent (`PlaceOrder`, `ApproveRefund`) rather than CRUD operations (`UpdateOrder`)
- **Optimized Read Models** — Query models are denormalized views tailored to specific UI or API needs
- **Eventual Consistency** — The read model may lag behind the write model; the system embraces eventual consistency
- **Independent Scalability** — Read and write sides can be scaled, deployed, and optimized independently

### CQRS vs. CRUD

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         CRUD vs. CQRS                                    │
├──────────────────┬────────────────────────────────────────────────────────┤
│  CRUD            │  Single model for reads and writes.                   │
│                  │  Same data structure is used to query and update.     │
│                  │  Simplest approach — appropriate for most apps.       │
├──────────────────┼────────────────────────────────────────────────────────┤
│  CQRS            │  Separate models for reads and writes.                │
│                  │  Write model enforces invariants and business rules.  │
│                  │  Read model is denormalized for fast, specific queries│
│                  │  Suited for complex domains with asymmetric R/W loads.│
└──────────────────┴────────────────────────────────────────────────────────┘
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           CQRS Architecture                                      │
│                                                                                  │
│     Clients / UI                                                                 │
│     ┌──────────────┐                           ┌──────────────┐                  │
│     │  Commands     │                           │  Queries      │                  │
│     │  (writes)     │                           │  (reads)      │                  │
│     └──────┬───────┘                           └──────┬───────┘                  │
│            │                                          │                           │
│            ▼                                          ▼                           │
│  ┌──────────────────┐                      ┌──────────────────┐                  │
│  │  Command Handler  │                      │  Query Handler    │                  │
│  │  (validates,      │                      │  (reads from      │                  │
│  │   enforces rules) │                      │   denormalized    │                  │
│  └────────┬─────────┘                      │   read store)     │                  │
│           │                                 └────────┬─────────┘                  │
│           ▼                                          ▲                            │
│  ┌──────────────────┐                                │                            │
│  │  Write Model      │        Projection /           │                            │
│  │  (Domain Model)   │        Synchronization        │                            │
│  └────────┬─────────┘     ┌──────────────────┐      │                            │
│           │               │  Read Model       │      │                            │
│           ▼               │  Updater          │──────┘                            │
│  ┌──────────────────┐     │  (denormalizes    │                                   │
│  │  Write Store      │────▶│   and projects)   │                                   │
│  │  (normalized,     │     └──────────────────┘                                   │
│  │   transactional)  │               │                                            │
│  └──────────────────┘               ▼                                            │
│                            ┌──────────────────┐                                   │
│                            │  Read Store       │                                   │
│                            │  (denormalized,   │                                   │
│                            │   query-optimized) │                                   │
│                            └──────────────────┘                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### CQRS with Event Sourcing

CQRS is frequently paired with Event Sourcing, where the write store is an append-only event log:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      CQRS + Event Sourcing                                       │
│                                                                                  │
│  ┌──────────────┐                                ┌──────────────┐                │
│  │  Command      │                                │  Query        │                │
│  └──────┬───────┘                                └──────┬───────┘                │
│         │                                               │                        │
│         ▼                                               ▼                        │
│  ┌──────────────────┐                         ┌──────────────────┐               │
│  │  Command Handler  │                         │  Query Handler    │               │
│  └────────┬─────────┘                         └────────┬─────────┘               │
│           │                                            ▲                          │
│           ▼                                            │                          │
│  ┌──────────────────┐    Events     ┌───────────────────┐                        │
│  │  Aggregate        │─────────────▶│  Event Projector   │                        │
│  │  (Domain Model)   │              │  (builds read      │                        │
│  └────────┬─────────┘              │   models from      │                        │
│           │                         │   event stream)    │                        │
│           ▼                         └────────┬──────────┘                        │
│  ┌──────────────────┐                        │                                   │
│  │  Event Store      │                        ▼                                   │
│  │  (append-only     │              ┌──────────────────┐                          │
│  │   event log)      │              │  Read Store(s)    │                          │
│  └──────────────────┘              │  (SQL, document,  │                          │
│                                     │   search index,   │                          │
│                                     │   cache)          │                          │
│                                     └──────────────────┘                          │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### 1. Commands & Command Handlers

Commands express user intent and are processed by handlers that enforce business rules:

```
// Command — a request to change state (imperative, task-based)
IMMUTABLE DATA PlaceOrderCommand
    orderId     : UUID
    customerId  : UUID
    items       : List<OrderItemData>
    shippingAddress : Address
END DATA

// Command Handler — validates and executes the command
CLASS PlaceOrderCommandHandler
    CONSTRUCTOR(
        orderRepository  : OrderRepository,
        customerRepository : CustomerRepository,
        eventPublisher   : EventPublisher
    )

    FUNCTION handle(command : PlaceOrderCommand) -> CommandResult
        // Validate business rules
        customer = customerRepository.findById(command.customerId)
        IF customer IS NULL THEN
            RETURN CommandResult.FAILURE("Customer not found")
        END IF

        IF NOT customer.isActive() THEN
            RETURN CommandResult.FAILURE("Customer account is inactive")
        END IF

        // Create aggregate and apply business logic
        order = NEW Order(
            id         = command.orderId,
            customerId = command.customerId
        )

        FOR EACH item IN command.items
            order.addItem(item.productId, item.quantity, item.price)
        END FOR

        order.setShippingAddress(command.shippingAddress)
        order.submit()

        // Persist write model
        orderRepository.save(order)

        // Publish domain events for read model projection
        FOR EACH event IN order.pendingEvents()
            eventPublisher.publish(event)
        END FOR

        RETURN CommandResult.SUCCESS(orderId = order.id)
END CLASS
```

### 2. Queries & Query Handlers

Queries read from denormalized, pre-computed read models:

```
// Query — a request for data (no side effects)
IMMUTABLE DATA GetOrderSummaryQuery
    orderId : UUID
END DATA

IMMUTABLE DATA GetCustomerOrdersQuery
    customerId : UUID
    page       : Integer
    pageSize   : Integer
    status     : String OR NULL    // Optional filter
END DATA

// Query Handler — reads from an optimized read store
CLASS GetOrderSummaryQueryHandler
    CONSTRUCTOR(readStore : OrderReadStore)

    FUNCTION handle(query : GetOrderSummaryQuery) -> OrderSummaryView
        // Direct read from denormalized view — no joins, no computation
        view = readStore.getOrderSummary(query.orderId)

        IF view IS NULL THEN
            RAISE OrderNotFoundError(query.orderId)
        END IF

        RETURN view
END CLASS

// Read model — pre-computed, denormalized for the specific query
IMMUTABLE DATA OrderSummaryView
    orderId           : UUID
    customerName      : String      // Denormalized from customer
    customerEmail     : String      // Denormalized from customer
    items             : List<OrderItemView>
    totalAmount       : Decimal     // Pre-computed
    status            : String
    statusDisplayName : String      // Pre-formatted
    createdAt         : DateTime
    lastUpdatedAt     : DateTime
END DATA

CLASS GetCustomerOrdersQueryHandler
    CONSTRUCTOR(readStore : OrderReadStore)

    FUNCTION handle(query : GetCustomerOrdersQuery) -> PagedResult<OrderListView>
        // Pre-built index for customer orders — fast lookup, no joins
        RETURN readStore.getCustomerOrders(
            customerId = query.customerId,
            status     = query.status,
            page       = query.page,
            pageSize   = query.pageSize
        )
END CLASS
```

### 3. Read Model Projections

Projections subscribe to domain events and maintain denormalized read models:

```
// Projector — builds read models from domain events
CLASS OrderReadModelProjector
    CONSTRUCTOR(readStore : OrderReadStore)

    FUNCTION onOrderPlaced(event : OrderPlacedEvent) -> Void
        readStore.upsert(
            id = event.orderId,
            view = NEW OrderSummaryView(
                orderId       = event.orderId,
                customerName  = event.customerName,
                customerEmail = event.customerEmail,
                items         = mapToItemViews(event.items),
                totalAmount   = event.totalAmount,
                status        = "placed",
                statusDisplayName = "Order Placed",
                createdAt     = event.occurredAt,
                lastUpdatedAt = event.occurredAt
            )
        )

        // Also update the customer orders list view
        readStore.addToCustomerOrders(
            customerId = event.customerId,
            orderListView = NEW OrderListView(
                orderId     = event.orderId,
                totalAmount = event.totalAmount,
                status      = "placed",
                itemCount   = event.items.size(),
                createdAt   = event.occurredAt
            )
        )

    FUNCTION onOrderShipped(event : OrderShippedEvent) -> Void
        readStore.updateFields(
            id = event.orderId,
            updates = {
                status            = "shipped",
                statusDisplayName = "Shipped",
                trackingNumber    = event.trackingNumber,
                shippedAt         = event.occurredAt,
                lastUpdatedAt     = event.occurredAt
            }
        )

    FUNCTION onOrderCancelled(event : OrderCancelledEvent) -> Void
        readStore.updateFields(
            id = event.orderId,
            updates = {
                status            = "cancelled",
                statusDisplayName = "Cancelled — " + event.reason,
                cancelledAt       = event.occurredAt,
                lastUpdatedAt     = event.occurredAt
            }
        )
END CLASS
```

### 4. Event Sourcing Integration

When paired with event sourcing, the write store is an append-only event stream:

```
// Event-sourced aggregate — state is derived from events
CLASS OrderAggregate
    PROPERTIES
        id        : UUID
        status    : String
        items     : List<OrderItem>
        events    : List<DomainEvent>    // Uncommitted events
        version   : Integer

    // Reconstruct state by replaying events
    STATIC FUNCTION fromHistory(eventStream : List<DomainEvent>) -> OrderAggregate
        order = NEW OrderAggregate()
        FOR EACH event IN eventStream
            order.apply(event, isNew = FALSE)
        END FOR
        RETURN order

    FUNCTION submit() -> Void
        IF items.isEmpty() THEN
            RAISE ValidationError("Cannot submit empty order")
        END IF
        IF status != "draft" THEN
            RAISE InvalidStateError("Order already submitted")
        END IF

        raiseEvent(NEW OrderSubmittedEvent(
            orderId    = id,
            totalAmount = computeTotal(),
            occurredAt  = NOW()
        ))

    // Apply event to internal state (used for replay AND new events)
    FUNCTION apply(event : DomainEvent, isNew : Boolean) -> Void
        MATCH event.type
            CASE "OrderCreated":
                id = event.orderId
                status = "draft"
                items = EMPTY LIST
            CASE "ItemAdded":
                items.ADD(NEW OrderItem(event.productId, event.quantity, event.price))
            CASE "OrderSubmitted":
                status = "submitted"
            CASE "OrderShipped":
                status = "shipped"
        END MATCH

        IF isNew THEN
            events.ADD(event)    // Track for persistence
        END IF
        version = version + 1

    FUNCTION raiseEvent(event : DomainEvent) -> Void
        apply(event, isNew = TRUE)
END CLASS

// Event Store — append-only persistence
CLASS EventStore
    CONSTRUCTOR(storage : EventStorage)

    FUNCTION save(aggregateId : UUID, events : List<DomainEvent>, expectedVersion : Integer) -> Void
        // Optimistic concurrency — prevent conflicting writes
        currentVersion = storage.getLatestVersion(aggregateId)

        IF currentVersion != expectedVersion THEN
            RAISE ConcurrencyConflictError(aggregateId, expectedVersion, currentVersion)
        END IF

        storage.append(aggregateId, events, startVersion = expectedVersion + 1)

    FUNCTION load(aggregateId : UUID) -> List<DomainEvent>
        RETURN storage.readStream(aggregateId)

    FUNCTION loadFromVersion(aggregateId : UUID, fromVersion : Integer) -> List<DomainEvent>
        RETURN storage.readStream(aggregateId, fromVersion = fromVersion)
END CLASS
```

### 5. Handling Eventual Consistency

Strategies for dealing with the read model lagging behind the write model:

```
// Consistency strategies for CQRS
CLASS ConsistencyManager
    CONSTRUCTOR(
        writeStore    : WriteStore,
        readStore     : ReadStore,
        projectionLog : ProjectionLog
    )

    // Strategy 1: Return command result with version for client-side polling
    FUNCTION executeWithVersion(command : Command) -> CommandResultWithVersion
        result = commandHandler.handle(command)
        RETURN NEW CommandResultWithVersion(
            result  = result,
            version = result.aggregateVersion    // Client can poll until read model catches up
        )

    // Strategy 2: Read-your-writes consistency
    FUNCTION queryWithMinVersion(query : Query, minVersion : Integer) -> QueryResult
        // Wait briefly for projection to catch up
        deadline = NOW() + MAX_CONSISTENCY_WAIT

        WHILE NOW() < deadline
            currentVersion = projectionLog.getProjectedVersion(query.aggregateId)
            IF currentVersion >= minVersion THEN
                RETURN queryHandler.handle(query)
            END IF
            WAIT(Duration.millis(50))
        END WHILE

        // Fallback: read directly from write model (slower but consistent)
        RETURN readFromWriteModel(query)

    // Strategy 3: Synchronous projection for critical paths
    FUNCTION executeWithSyncProjection(command : Command) -> QueryResult
        result = commandHandler.handle(command)

        // Synchronously project the events before returning
        FOR EACH event IN result.events
            projector.project(event)
        END FOR

        RETURN queryHandler.handle(query = queryFromResult(result))
END CLASS
```

## Project Structure

```
src/
├── commands/                       # Command side (write model)
│   ├── handlers/
│   ├── validators/
│   └── commands/                   # Command DTOs
│
├── queries/                        # Query side (read model)
│   ├── handlers/
│   ├── queries/                    # Query DTOs
│   └── views/                     # Read model view DTOs
│
├── domain/                         # Domain model (aggregates, entities, events)
│   ├── aggregates/
│   ├── entities/
│   ├── events/
│   └── value_objects/
│
├── projections/                    # Event-to-read-model projectors
│   ├── order_projector/
│   ├── customer_projector/
│   └── reporting_projector/
│
├── infrastructure/                 # Persistence and messaging
│   ├── write_store/                # Event store or normalized DB
│   ├── read_store/                 # Denormalized read DB(s)
│   ├── event_bus/
│   └── config/
│
└── tests/
    ├── command_tests/
    ├── query_tests/
    ├── projection_tests/
    └── integration/
```

## Benefits

1. **Optimized Read Performance** — Read models are denormalized and tailored to specific queries, eliminating complex joins
2. **Independent Scalability** — Scale the read side and write side independently based on actual workload ratios
3. **Simplified Commands** — Task-based commands clearly express user intent and enforce business rules
4. **Flexible Read Models** — Build multiple read models (SQL, search index, cache) from the same source of truth
5. **Auditability** — When combined with event sourcing, provides a complete audit trail of every state change
6. **Team Autonomy** — Read and write models can be developed and deployed independently

## Trade-offs

| Advantage                      | Consideration                                        |
| ------------------------------ | ---------------------------------------------------- |
| Optimized read performance     | Eventual consistency requires careful UX design      |
| Independent read/write scaling | Increased system complexity (two models to maintain) |
| Clear command/query separation | More code and infrastructure than simple CRUD        |
| Multiple optimized read stores | Projection logic must handle events idempotently     |
| Full audit trail (with ES)     | Event schema versioning is non-trivial               |
| Team parallelism               | Debugging across command→event→projection is harder  |

## When to Use

✅ **Good fit for:**

- Systems with asymmetric read/write loads (reads >> writes or vice versa)
- Complex domains where the write model and read model have fundamentally different shapes
- Applications requiring multiple read representations (API, reports, search, dashboards)
- Event-driven systems where events are already the primary communication mechanism
- Domains where audit trails and temporal queries are important

❌ **Not ideal for:**

- Simple CRUD applications with one-to-one mapping between reads and writes
- Systems requiring strong immediate consistency across all views
- Small applications where the complexity overhead is not justified
- Domains with low data volumes and simple query patterns
- Teams unfamiliar with eventual consistency patterns

## References

- [CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [CQRS Documents — Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [Command Query Responsibility Segregation Pattern — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Sourcing Pattern — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Implementing Domain-Driven Design — Vaughn Vernon](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577)
- [Versioning in an Event Sourced System — Greg Young](https://leanpub.com/esversioning)
