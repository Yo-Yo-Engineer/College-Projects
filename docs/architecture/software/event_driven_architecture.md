# Event-Driven Architecture (EDA)

## Overview

**Event-Driven Architecture (EDA)** is an architectural pattern that promotes the production, detection, consumption, and reaction to events. Unlike request-driven architectures where services call each other directly, EDA enables loosely coupled systems that communicate through events.

An **event** represents a significant change in state or an occurrence that happened in the system. Events are facts about something that already happened — they are immutable and represent history.

## Core Concepts

### What is an Event?

An event is a record of something that happened:

- **Immutable** — Cannot be changed once created
- **Past tense** — Represents something that already occurred (e.g., `OrderPlaced`, not `PlaceOrder`)
- **Contains state** — Includes relevant data about what happened
- **Named descriptively** — `OrderPlaced`, `PaymentReceived`, `UserRegistered`

```
// Event — Immutable record of a domain occurrence
IMMUTABLE DATA OrderPlaced
    eventId     : UUID
    orderId     : UUID
    customerId  : UUID
    totalAmount : Decimal
    items       : List<ItemData>   // Immutable collection
    occurredAt  : DateTime

    FUNCTION eventType() -> String
        RETURN "OrderPlaced"
END DATA
```

### Event Types

#### 1. Domain Events

Business-meaningful occurrences within a bounded context:

- `OrderPlaced`
- `PaymentProcessed`
- `InventoryReserved`

#### 2. Integration Events

Events shared between bounded contexts or services:

- `OrderCompletedIntegrationEvent`
- `CustomerCreatedIntegrationEvent`

#### 3. Event Notifications

Thin events that notify something happened — consumers fetch details separately:

```
IMMUTABLE DATA OrderPlacedNotification
    orderId    : UUID
    occurredAt : DateTime
    // Consumer calls Order Service API for full details
END DATA
```

#### 4. Event-Carried State Transfer

Fat events containing all data consumers need — avoids callback queries:

```
IMMUTABLE DATA OrderPlacedFull
    orderId         : UUID
    customerId      : UUID
    customerEmail   : String
    shippingAddress : Address
    items           : List<OrderItem>
    total           : Decimal
    // Contains everything consumers need — no callbacks required
END DATA
```

### Architecture Patterns

#### 1. Event Notification (Pub/Sub)

The simplest EDA pattern — producers publish events, consumers subscribe independently:

```
┌─────────────┐     publishes     ┌─────────────┐     subscribes    ┌─────────────┐
│   Order      │ ────────────────▶│    Event     │ ───────────────▶ │  Inventory   │
│   Service    │                  │     Bus      │                  │   Service    │
└─────────────┘                   └─────────────┘                   └─────────────┘
                                        │
                                        │ subscribes
                                        ▼
                                  ┌─────────────┐
                                  │  Shipping    │
                                  │   Service    │
                                  └─────────────┘
```

#### 2. Event Sourcing

Store all changes to application state as a sequence of events — the event log _is_ the source of truth:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Event Store                                     │
├──────────────────────────────────────────────────────────────────────────┤
│  OrderCreated(id=1, customer=A, items=[...])            @ 10:00          │
│  ItemAddedToOrder(id=1, product=X, qty=2)               @ 10:05          │
│  ItemRemovedFromOrder(id=1, product=Y)                  @ 10:10          │
│  OrderSubmitted(id=1)                                   @ 10:15          │
│  PaymentReceived(id=1, amount=99.99)                    @ 10:20          │
│  OrderShipped(id=1, tracking=ABC123)                    @ 14:30          │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Replay events to rebuild state
                                    ▼
                           ┌──────────────────┐
                           │  Current State    │
                           │   Order #1        │
                           │   Status: Shipped │
                           └──────────────────┘
```

#### 3. CQRS + Event Sourcing

Combine Command Query Responsibility Segregation with Event Sourcing — separate write and read models:

```
                                  Commands
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Write Side                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────┐  │
│  │  Command     │───▶│   Domain    │───▶│       Event Store           │  │
│  │  Handler     │    │   Model     │    │   (append-only log)        │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                                    │
                                                    │ Publish events
                                                    ▼
                                            ┌─────────────┐
                                            │  Event Bus   │
                                            └─────────────┘
                                                    │
                                                    │ Subscribe
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Read Side                                     │
│  ┌─────────────────────────────┐    ┌─────────────────────────────────┐ │
│  │       Event Handler          │───▶│      Read Database              │ │
│  │    (Projection Builder)      │    │   (Optimized for queries)      │ │
│  └─────────────────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
                                                Queries
```

## Implementation

### Event Definitions

```
// Base type for all domain events
ABSTRACT DATA DomainEvent
    eventId    : UUID = GENERATE_UUID()
    occurredAt : DateTime = NOW()
END DATA

IMMUTABLE DATA OrderPlaced EXTENDS DomainEvent
    orderId     : UUID
    customerId  : UUID
    items       : List<ItemData>
    totalAmount : Decimal
END DATA

IMMUTABLE DATA PaymentReceived EXTENDS DomainEvent
    orderId   : UUID
    paymentId : UUID
    amount    : Decimal
END DATA

IMMUTABLE DATA OrderShipped EXTENDS DomainEvent
    orderId        : UUID
    trackingNumber : String
    carrier        : String
END DATA
```

### Event Bus

```
// Event Bus — Abstraction for publishing and subscribing to events
INTERFACE EventBus
    FUNCTION publish(event : DomainEvent) -> Void
    FUNCTION subscribe(eventType : Type, handler : EventHandler) -> Void
END INTERFACE

// In-Memory implementation (for testing or single-process apps)
CLASS InMemoryEventBus IMPLEMENTS EventBus
    PROPERTIES
        handlers : Map<Type, List<EventHandler>> = EMPTY MAP

    FUNCTION publish(event : DomainEvent) -> Void
        eventType = TYPE_OF(event)
        IF handlers.HAS_KEY(eventType) THEN
            FOR EACH handler IN handlers.GET(eventType)
                handler.handle(event)
            END FOR
        END IF

    FUNCTION subscribe(eventType : Type, handler : EventHandler) -> Void
        IF NOT handlers.HAS_KEY(eventType) THEN
            handlers.PUT(eventType, EMPTY LIST)
        END IF
        handlers.GET(eventType).ADD(handler)
END CLASS
```

### Event Handlers

```
// Inventory handler — Reacts to OrderPlaced events
CLASS InventoryEventHandler
    CONSTRUCTOR(inventoryRepository : InventoryRepository)

    FUNCTION handleOrderPlaced(event : OrderPlaced) -> Void
        FOR EACH item IN event.items
            inventoryRepository.reserve(
                productId = item.productId,
                quantity  = item.quantity,
                orderId   = event.orderId
            )
        END FOR
END CLASS

// Notification handler — Reacts to OrderShipped events
CLASS NotificationEventHandler
    CONSTRUCTOR(notificationService : NotificationService)

    FUNCTION handleOrderShipped(event : OrderShipped) -> Void
        notificationService.sendShippingNotification(
            orderId        = event.orderId,
            trackingNumber = event.trackingNumber,
            carrier        = event.carrier
        )
END CLASS
```

### Event Producer (Aggregate)

```
// Aggregate that produces domain events
CLASS Order
    PROPERTIES
        id         : UUID
        customerId : UUID
        items      : List<OrderItem> = EMPTY LIST
        status     : String = "draft"
        _events    : List<DomainEvent> = EMPTY LIST  // Pending events

    FUNCTION pendingEvents() -> List<DomainEvent>
        RETURN COPY OF _events

    FUNCTION clearEvents() -> Void
        _events.CLEAR()

    FUNCTION place() -> Void
        IF status != "draft" THEN
            THROW InvalidOperationError("Order already placed")
        END IF

        status = "placed"

        _events.ADD(NEW OrderPlaced(
            orderId     = id,
            customerId  = customerId,
            items       = TUPLE(item.toData() FOR EACH item IN items),
            totalAmount = total()
        ))

    FUNCTION markShipped(trackingNumber : String, carrier : String) -> Void
        IF status != "paid" THEN
            THROW InvalidOperationError("Order must be paid before shipping")
        END IF

        status = "shipped"

        _events.ADD(NEW OrderShipped(
            orderId        = id,
            trackingNumber = trackingNumber,
            carrier        = carrier
        ))
END CLASS
```

### Application Service

```
CLASS OrderApplicationService
    CONSTRUCTOR(
        orderRepository : OrderRepository,
        eventBus        : EventBus,
        unitOfWork      : UnitOfWork
    )

    FUNCTION placeOrder(orderId : UUID) -> Void
        BEGIN TRANSACTION via unitOfWork

            order = orderRepository.findById(orderId)
            IF order IS NULL THEN
                THROW OrderNotFoundError(orderId)
            END IF

            order.place()
            orderRepository.save(order)
            unitOfWork.commit()

        END TRANSACTION

        // Publish events AFTER successful commit
        FOR EACH event IN order.pendingEvents()
            eventBus.publish(event)
        END FOR

        order.clearEvents()
END CLASS
```

### Event Sourcing Implementation

```
// Base class for event-sourced aggregates
ABSTRACT CLASS EventSourcedAggregate
    PROPERTIES
        _changes : List<DomainEvent> = EMPTY LIST
        _version : Integer = 0

    FUNCTION uncommittedChanges() -> List<DomainEvent>
        RETURN COPY OF _changes

    FUNCTION markChangesAsCommitted() -> Void
        _changes.CLEAR()

    FUNCTION loadFromHistory(events : List<DomainEvent>) -> Void
        FOR EACH event IN events
            apply(event)
            _version = _version + 1
        END FOR

    PROTECTED FUNCTION applyChange(event : DomainEvent) -> Void
        apply(event)
        _changes.ADD(event)

    ABSTRACT FUNCTION apply(event : DomainEvent) -> Void
        // Subclasses implement state mutation logic
END CLASS

// Event-sourced Order aggregate
CLASS Order EXTENDS EventSourcedAggregate
    PROPERTIES
        id         : UUID
        customerId : UUID
        items      : List<ItemData> = EMPTY LIST
        status     : String

    FUNCTION create(customerId : UUID, items : List<ItemData>) -> Void
        applyChange(NEW OrderCreated(
            orderId    = id,
            customerId = customerId,
            items      = items
        ))

    FUNCTION apply(event : DomainEvent) -> Void
        MATCH event
            CASE OrderCreated:
                id         = event.orderId
                customerId = event.customerId
                items      = event.items
                status     = "created"
            CASE OrderPlaced:
                status = "placed"
            CASE OrderShipped:
                status = "shipped"
        END MATCH
END CLASS

// Event Store — Append-only log of events per aggregate
CLASS EventStore
    PROPERTIES
        events : Map<UUID, List<(DomainEvent, Integer)>> = EMPTY MAP

    FUNCTION saveEvents(aggregateId : UUID,
                        newEvents : List<DomainEvent>,
                        expectedVersion : Integer) -> Void
        IF NOT events.HAS_KEY(aggregateId) THEN
            events.PUT(aggregateId, EMPTY LIST)
        END IF

        currentVersion = events.GET(aggregateId).SIZE()
        IF currentVersion != expectedVersion THEN
            THROW ConcurrencyError(
                "Expected version " + expectedVersion +
                ", but found " + currentVersion
            )
        END IF

        FOR i FROM 0 TO newEvents.SIZE() - 1
            events.GET(aggregateId).ADD(
                (newEvents[i], expectedVersion + i + 1)
            )
        END FOR

    FUNCTION getEvents(aggregateId : UUID) -> List<DomainEvent>
        RETURN [event FOR (event, version) IN events.GET(aggregateId, EMPTY LIST)]
END CLASS
```

## Project Structure

```
src/
├── domain/
│   ├── events/                     # Event definitions
│   ├── aggregates/                 # Aggregate roots
│   └── services/                   # Domain services
│
├── application/
│   ├── commands/                   # Command definitions
│   ├── queries/                    # Query definitions
│   ├── handlers/
│   │   ├── command_handlers/       # Command handlers
│   │   └── event_handlers/        # Event handlers (reactions)
│   └── services/                   # Application services
│
├── infrastructure/
│   ├── messaging/                  # Event bus, broker adapters
│   ├── persistence/
│   │   ├── event_store/           # Event store implementation
│   │   └── read_models/          # Projection repositories
│   ├── projections/               # Read model builders
│   └── web/
│       └── controllers/
│
├── config/
│   └── event_subscriptions        # Wires handlers to events
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

### Message Broker Integration

```
// Message Broker Adapter — Publishes events to an external broker
CLASS MessageBrokerEventPublisher IMPLEMENTS EventBus
    CONSTRUCTOR(connectionString : String)

    ON INITIALIZE
        connection = CONNECT(connectionString)
        channel    = connection.createChannel()
        channel.declareExchange(name = "domain_events", type = "topic")

    FUNCTION publish(event : DomainEvent) -> Void
        routingKey = event.eventType()
        body       = SERIALIZE_TO_JSON(event)

        channel.publish(
            exchange   = "domain_events",
            routingKey = routingKey,
            body       = body,
            options    = { contentType: "application/json", persistent: TRUE }
        )

    FUNCTION subscribe(eventType : Type, handler : EventHandler) -> Void
        queueName = handler.NAME + "." + eventType.NAME
        channel.declareQueue(name = queueName)
        channel.bindQueue(queue = queueName, exchange = "domain_events",
                          routingKey = eventType.NAME)
        channel.consume(queue = queueName, callback = handler.handle)
END CLASS
```

## Key Design Considerations

### Idempotency

Event handlers **must** be idempotent — they may receive the same event more than once due to retries, network issues, or at-least-once delivery guarantees:

```
CLASS IdempotentInventoryHandler
    CONSTRUCTOR(inventoryRepository : InventoryRepository,
                processedEvents : ProcessedEventStore)

    FUNCTION handleOrderPlaced(event : OrderPlaced) -> Void
        IF processedEvents.hasBeenProcessed(event.eventId) THEN
            RETURN  // Already handled, skip
        END IF

        // Process the event
        FOR EACH item IN event.items
            inventoryRepository.reserve(item.productId, item.quantity, event.orderId)
        END FOR

        processedEvents.markAsProcessed(event.eventId)
END CLASS
```

### Outbox Pattern

To ensure events are published reliably (avoiding dual-write problems), use the **Outbox Pattern**:

1. Save the domain event to an outbox table in the _same database transaction_ as the state change
2. A separate process reads the outbox and publishes events to the message broker
3. Mark outbox entries as published after successful broker delivery

### Schema Evolution

Event schemas must evolve carefully since events are stored immutably:

- Add new fields as optional with defaults
- Never remove or rename existing fields
- Use versioning (e.g., `OrderPlaced_v2`) for breaking changes
- Support upcasting older event versions to newer formats

## Benefits

1. **Loose Coupling** — Services don't need to know about each other, only about events
2. **Scalability** — Easy to add new consumers without modifying producers
3. **Resilience** — Async processing enables better failure handling and retry strategies
4. **Audit Trail** — Event sourcing provides complete, immutable history of all changes
5. **Flexibility** — Easy to add new projections, read models, or downstream processes
6. **Real-time** — Enables real-time reactions to business events

## Trade-offs

| Advantage                         | Consideration                                 |
| --------------------------------- | --------------------------------------------- |
| Loose coupling between services   | Eventual consistency requires design thought  |
| Highly scalable and extensible    | Increased operational complexity              |
| Full audit trail (event sourcing) | Event schema evolution needs careful planning |
| Resilient to partial failures     | Debugging async flows is harder               |
| Real-time reactions               | Ordering guarantees require explicit design   |

## When to Use

✅ **Good fit for:**

- Microservices architectures requiring loose coupling
- Systems requiring high scalability and resilience
- Applications with complex business workflows (sagas, choreography)
- Real-time data processing and notifications
- Audit and compliance requirements (event sourcing)
- Systems where multiple consumers react to the same business events

❌ **Not ideal for:**

- Simple CRUD applications with straightforward data flows
- Systems requiring strong, immediate consistency
- Small applications with few services or components
- Teams unfamiliar with asynchronous patterns and eventual consistency

## References

- [Event-Driven Architecture — Microsoft Azure Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Event Sourcing Pattern — Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Domain Events — Martin Fowler](https://martinfowler.com/eaaDev/DomainEvent.html)
- [Event Sourcing — Greg Young](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [Enterprise Integration Patterns — Gregor Hohpe](https://www.enterpriseintegrationpatterns.com/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)
