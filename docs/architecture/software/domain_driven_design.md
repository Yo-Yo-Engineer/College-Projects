# Domain-Driven Design (DDD)

## Overview

**Domain-Driven Design (DDD)** is an approach to software development that centers the design on a rich understanding of the business domain, its processes, and its rules. The name comes from Eric Evans's 2003 book _Domain-Driven Design: Tackling Complexity in the Heart of Software_, which introduced a vocabulary of patterns for modeling complex domains in software.

DDD is not a technology or a framework — it is a design philosophy and a set of patterns for organizing code around the business domain. It is particularly suited to complex domains where a lot of messy business logic needs to be organized, and where the cost of getting the model wrong is high.

Key principles:

- **Ubiquitous Language** — A shared language between developers and domain experts, used consistently in code, documentation, and conversation
- **Model-Driven Design** — The software model directly reflects domain concepts; changes to understanding of the domain drive changes to the code
- **Bounded Contexts** — Large domains are divided into explicit boundaries where a particular model applies consistently
- **Strategic over Tactical** — Getting the high-level boundaries and context relationships right matters more than any individual pattern
- **Continuous Refinement** — The domain model evolves through ongoing collaboration with domain experts, not through upfront design

## Core Concepts

### Strategic Design

Strategic design addresses the large-scale structure — how to divide a complex domain into manageable parts and how those parts relate to each other.

#### Bounded Contexts

A **Bounded Context** is an explicit boundary within which a domain model exists. The same real-world concept may have different meanings in different contexts:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        E-Commerce Domain                                     │
│                                                                             │
│   ┌───────────────────────┐      ┌───────────────────────┐                  │
│   │   Sales Context        │      │   Shipping Context     │                 │
│   │                        │      │                        │                 │
│   │  "Customer" means:     │      │  "Customer" means:     │                 │
│   │  - Contact info        │      │  - Shipping address    │                 │
│   │  - Purchase history    │      │  - Delivery prefs      │                 │
│   │  - Credit limit        │      │  - Package history     │                 │
│   │                        │      │                        │                 │
│   │  "Order" means:        │      │  "Order" means:        │                 │
│   │  - Line items + prices │      │  - Packages to ship    │                 │
│   │  - Discounts applied   │      │  - Weight & dimensions │                 │
│   │  - Payment status      │      │  - Tracking number     │                 │
│   └───────────┬────────────┘      └───────────┬────────────┘                │
│               │                               │                             │
│               │    Integration Events          │                             │
│               └───────────────────────────────┘                             │
│                                                                             │
│   ┌───────────────────────┐      ┌───────────────────────┐                  │
│   │   Inventory Context    │      │   Billing Context      │                 │
│   │                        │      │                        │                 │
│   │  "Product" means:      │      │  "Customer" means:     │                 │
│   │  - SKU                 │      │  - Billing address     │                 │
│   │  - Stock level         │      │  - Payment methods     │                 │
│   │  - Warehouse location  │      │  - Invoice history     │                 │
│   └────────────────────────┘      └────────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Key insight:** Do not force one unified model across all contexts. Each bounded context has its own model optimized for its specific purpose.

#### Context Mapping

A **Context Map** describes the relationships between bounded contexts. Common relationship patterns:

| Relationship              | Description                                                                       |
| ------------------------- | --------------------------------------------------------------------------------- |
| **Shared Kernel**         | Two contexts share a subset of the model; changes require joint agreement         |
| **Customer–Supplier**     | Upstream supplier provides data downstream; downstream has input on what's needed |
| **Conformist**            | Downstream accepts the upstream model as-is with no negotiation                   |
| **Anti-Corruption Layer** | Downstream translates the upstream model into its own language                    |
| **Open Host Service**     | Upstream provides a well-defined protocol for many downstream consumers           |
| **Published Language**    | A shared data interchange format (e.g., event schemas, JSON contracts)            |
| **Separate Ways**         | No integration — contexts are completely independent                              |

```
// Anti-Corruption Layer — Translates an external model into our domain language
CLASS ShippingAntiCorruptionLayer
    PROPERTIES
        salesApi : SalesServiceClient

    // Translate from Sales context "Order" to Shipping context "ShipmentRequest"
    FUNCTION toShipmentRequest(salesOrderId : UUID) -> ShipmentRequest
        salesOrder = salesApi.getOrder(salesOrderId)

        RETURN NEW ShipmentRequest(
            shipmentId      = generateUUID(),
            recipientName   = salesOrder.customerName,
            deliveryAddress = toShippingAddress(salesOrder.shippingAddress),
            packages        = salesOrder.lineItems.MAP(
                FUNCTION(item) ->
                    NEW Package(
                        sku       = item.productId,
                        quantity  = item.quantity,
                        weight    = inventoryLookup.getWeight(item.productId)
                    )
            ),
            priority = mapPriorityLevel(salesOrder.shippingTier)
        )

    // Maps Sales address structure to Shipping address structure
    PRIVATE FUNCTION toShippingAddress(salesAddr : SalesAddress) -> ShippingAddress
        RETURN NEW ShippingAddress(
            line1   = salesAddr.street,
            line2   = salesAddr.unit,
            city    = salesAddr.city,
            region  = salesAddr.state,
            postal  = salesAddr.zipCode,
            country = salesAddr.countryCode
        )
END CLASS
```

#### Subdomains

Domains are further classified by their strategic importance:

| Type                  | Description                                     | Design Approach                |
| --------------------- | ----------------------------------------------- | ------------------------------ |
| **Core Domain**       | What makes your business unique and competitive | Rich DDD model, invest heavily |
| **Supporting Domain** | Necessary but not a differentiator              | Simpler model, may outsource   |
| **Generic Domain**    | Standard functionality common across businesses | Buy or use off-the-shelf       |

```
// Example: E-commerce business
// Core Domain       → Pricing Engine, Recommendation Algorithm (competitive edge)
// Supporting Domain → Inventory Management, Order Fulfillment
// Generic Domain    → Authentication, Payment Processing, Email Sending
```

### Tactical Design

Tactical design provides the building blocks for implementing the domain model within a bounded context.

#### Entities

Objects with a unique identity that persists over time. Two entities are equal if they have the same identity, regardless of attribute values:

```
// Entity — Has identity and lifecycle
ENTITY Customer
    PROPERTIES
        id        : CustomerId            // Unique identity
        name      : PersonName            // Value object
        email     : EmailAddress           // Value object
        tier      : CustomerTier = STANDARD
        createdAt : DateTime

    // Business rule: Upgrade customer tier based on spending
    FUNCTION evaluateTierUpgrade(totalSpend : Money) -> Void
        IF totalSpend.amount >= 10000 AND tier != PREMIUM THEN
            tier = PREMIUM
            RAISE EVENT CustomerUpgraded(id, PREMIUM)
        ELSE IF totalSpend.amount >= 5000 AND tier == STANDARD THEN
            tier = GOLD
            RAISE EVENT CustomerUpgraded(id, GOLD)
        END IF

    // Equality is based on identity, not attributes
    FUNCTION equals(other : Customer) -> Boolean
        RETURN id == other.id
END ENTITY
```

#### Value Objects

Objects defined entirely by their attributes, with no conceptual identity. They are immutable and interchangeable:

```
// Value Object — Immutable, no identity, defined by attributes
IMMUTABLE DATA Money
    amount   : Decimal
    currency : CurrencyCode

    FUNCTION add(other : Money) -> Money
        IF currency != other.currency THEN
            THROW DomainError("Cannot add different currencies")
        END IF
        RETURN NEW Money(amount + other.amount, currency)

    FUNCTION multiply(factor : Decimal) -> Money
        RETURN NEW Money(amount * factor, currency)

    // Equality is based on all attributes
    FUNCTION equals(other : Money) -> Boolean
        RETURN amount == other.amount AND currency == other.currency
END DATA

IMMUTABLE DATA Address
    street  : String
    city    : String
    region  : String
    postal  : String
    country : CountryCode

    FUNCTION format() -> String
        RETURN street + ", " + city + ", " + region + " " + postal
END DATA

IMMUTABLE DATA DateRange
    start : Date
    end   : Date

    CONSTRUCTOR(start, end)
        IF end < start THEN
            THROW DomainError("End date must not precede start date")
        END IF

    FUNCTION contains(date : Date) -> Boolean
        RETURN date >= start AND date <= end

    FUNCTION overlaps(other : DateRange) -> Boolean
        RETURN start <= other.end AND end >= other.start
END DATA
```

#### Aggregates

A cluster of entities and value objects treated as a single unit for data changes. Each aggregate has a **root entity** that serves as the sole entry point:

```
// Aggregate Root — The only entry point into the aggregate
ENTITY Order   // ← Aggregate Root
    PROPERTIES
        id         : OrderId
        customerId : CustomerId
        items      : List<OrderItem>     // ← Entities within the aggregate
        status     : OrderStatus = DRAFT
        total      : Money = Money(0, "USD")

    // All modifications go through the aggregate root
    FUNCTION addItem(productId : ProductId,
                     quantity : Integer,
                     unitPrice : Money) -> Void
        IF status != DRAFT THEN
            THROW DomainError("Cannot modify a submitted order")
        END IF

        existingItem = items.FIND(i -> i.productId == productId)
        IF existingItem IS NOT NULL THEN
            existingItem.increaseQuantity(quantity)
        ELSE
            items.ADD(NEW OrderItem(
                id        = generateItemId(),
                productId = productId,
                quantity  = quantity,
                unitPrice = unitPrice
            ))
        END IF

        recalculateTotal()

    FUNCTION submit() -> Void
        IF items IS EMPTY THEN
            THROW DomainError("Cannot submit an empty order")
        END IF
        IF status != DRAFT THEN
            THROW DomainError("Order already submitted")
        END IF

        status = SUBMITTED
        RAISE EVENT OrderSubmitted(id, customerId, total, items)

    FUNCTION cancel(reason : String) -> Void
        IF status == SHIPPED THEN
            THROW DomainError("Cannot cancel a shipped order")
        END IF

        status = CANCELLED
        RAISE EVENT OrderCancelled(id, reason)

    PRIVATE FUNCTION recalculateTotal() -> Void
        total = items.MAP(i -> i.subtotal())
                     .REDUCE(Money(0, "USD"), (a, b) -> a.add(b))
END ENTITY

// Entity within the aggregate — accessed only through the root
ENTITY OrderItem
    PROPERTIES
        id        : OrderItemId
        productId : ProductId
        quantity  : Integer
        unitPrice : Money

    FUNCTION increaseQuantity(amount : Integer) -> Void
        quantity = quantity + amount

    FUNCTION subtotal() -> Money
        RETURN unitPrice.multiply(quantity)
END ENTITY
```

**Aggregate design rules:**

1. Reference other aggregates by identity (ID), not by direct object reference
2. One transaction modifies one aggregate — cross-aggregate consistency is eventual
3. Keep aggregates small — include only what must be consistent in a single transaction
4. Use domain events to communicate between aggregates

#### Domain Events

Things that happened in the domain that other parts of the system may care about:

```
// Domain Event — An immutable record of something that happened
IMMUTABLE DATA OrderSubmitted
    orderId    : OrderId
    customerId : CustomerId
    total      : Money
    items      : List<OrderItemSnapshot>
    occurredAt : DateTime = NOW()
END DATA

IMMUTABLE DATA CustomerUpgraded
    customerId : CustomerId
    newTier    : CustomerTier
    occurredAt : DateTime = NOW()
END DATA
```

#### Domain Services

Operations that do not naturally belong to any entity or value object, often involving multiple aggregates or external concepts:

```
// Domain Service — Logic that spans multiple aggregates
CLASS PricingService
    PROPERTIES
        discountPolicy : DiscountPolicy
        taxCalculator  : TaxCalculator

    FUNCTION calculateOrderTotal(
        order    : Order,
        customer : Customer
    ) -> PricingResult

        subtotal = order.total

        // Apply customer-tier discount
        discount = discountPolicy.calculateDiscount(customer.tier, subtotal)
        afterDiscount = subtotal.add(discount.negate())

        // Apply tax
        tax = taxCalculator.calculate(afterDiscount, order.shippingAddress)

        RETURN NEW PricingResult(
            subtotal  = subtotal,
            discount  = discount,
            tax       = tax,
            total     = afterDiscount.add(tax)
        )
END CLASS
```

#### Repositories

Provide the illusion of an in-memory collection of aggregates, hiding persistence details:

```
// Repository — Persistence abstraction for aggregate roots
INTERFACE OrderRepository
    FUNCTION findById(id : OrderId) -> Order OR NULL
    FUNCTION save(order : Order) -> Void
    FUNCTION delete(id : OrderId) -> Void
    FUNCTION findByCustomer(customerId : CustomerId) -> List<Order>
    FUNCTION nextId() -> OrderId
END INTERFACE

// Infrastructure implementation — hidden from domain layer
CLASS SqlOrderRepository IMPLEMENTS OrderRepository
    PROPERTIES
        database   : Database
        eventStore : EventPublisher

    FUNCTION save(order : Order) -> Void
        // Persist aggregate
        database.upsert("orders", mapToRow(order))

        FOR EACH item IN order.items
            database.upsert("order_items", mapItemToRow(item))
        END FOR

        // Publish domain events raised during this operation
        FOR EACH event IN order.pendingEvents()
            eventStore.publish(event)
        END FOR

        order.clearPendingEvents()
END CLASS
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Interface / API                        │
│   (Controllers, DTOs, view models, HTTP handlers)                │
├─────────────────────────────────────────────────────────────────┤
│                      Application Layer                           │
│   (Use cases, command/query handlers, orchestration)             │
│   Depends on: Domain Layer                                       │
├─────────────────────────────────────────────────────────────────┤
│                       Domain Layer                               │
│   (Entities, Value Objects, Aggregates, Domain Services,         │
│    Domain Events, Repository Interfaces)                         │
│   Depends on: NOTHING                                            │
├─────────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer                           │
│   (Repository implementations, ORM, messaging, external APIs)    │
│   Depends on: Domain Layer (implements its interfaces)           │
└─────────────────────────────────────────────────────────────────┘

        Dependencies always point inward toward the Domain →
```

```
// Application Layer — Orchestrates use cases using domain objects
CLASS SubmitOrderCommandHandler
    PROPERTIES
        orderRepo    : OrderRepository
        customerRepo : CustomerRepository
        pricing      : PricingService

    FUNCTION handle(command : SubmitOrderCommand) -> SubmitOrderResult
        // 1. Load aggregates
        order = orderRepo.findById(command.orderId)
        IF order IS NULL THEN
            THROW NotFoundError("Order not found")
        END IF

        customer = customerRepo.findById(order.customerId)

        // 2. Execute domain logic
        pricingResult = pricing.calculateOrderTotal(order, customer)
        order.applyPricing(pricingResult)
        order.submit()

        // 3. Persist
        orderRepo.save(order)

        // 4. Return result
        RETURN NEW SubmitOrderResult(
            orderId = order.id,
            total   = pricingResult.total,
            status  = order.status
        )
END CLASS
```

## Project Structure

```
bounded-context/
│
├── domain/                         # Domain Layer (zero external dependencies)
│   ├── model/
│   │   ├── aggregates/             # Aggregate roots and their children
│   │   ├── entities/               # Standalone entities
│   │   └── value-objects/          # Immutable value types
│   ├── events/                     # Domain event definitions
│   ├── services/                   # Domain services
│   ├── repositories/               # Repository interfaces (abstractions only)
│   └── exceptions/                 # Domain-specific errors
│
├── application/                    # Application Layer
│   ├── commands/                   # Command handlers (write operations)
│   ├── queries/                    # Query handlers (read operations)
│   ├── dtos/                       # Data transfer objects
│   └── services/                   # Application services / orchestrators
│
├── infrastructure/                 # Infrastructure Layer
│   ├── persistence/                # Repository implementations, ORM config
│   ├── messaging/                  # Event bus / message broker integration
│   ├── external/                   # Anti-corruption layers, external API clients
│   └── config/                     # Dependency injection, configuration
│
├── api/                            # Interface Layer
│   ├── endpoints/                  # HTTP controllers / gRPC services
│   ├── middleware/                 # Auth, validation, error handling
│   └── mappers/                    # DTO ↔ Domain object mappers
│
└── tests/
    ├── domain/                     # Pure domain logic tests
    ├── application/                # Use case tests
    └── integration/                # Infrastructure integration tests
```

## Benefits

1. **Ubiquitous Language** — Code reads like the business talks; reduces miscommunication between developers and domain experts
2. **Explicit Boundaries** — Bounded contexts prevent models from becoming entangled "big balls of mud"
3. **Focus Investment** — Core domains receive the richest modeling; generic domains are bought or kept simple
4. **Testable Domain Logic** — The domain layer has no infrastructure dependencies, making it fast to test in isolation
5. **Evolutionary Modeling** — The model evolves alongside business understanding through continuous collaboration
6. **Natural Microservice Boundaries** — Bounded contexts map directly to service boundaries in a microservices architecture

## Trade-offs

| Advantage                                      | Consideration                                             |
| ---------------------------------------------- | --------------------------------------------------------- |
| Business-aligned code structure                | High upfront learning curve for the team                  |
| Rich, expressive domain model                  | Overhead not justified for CRUD-heavy applications        |
| Ubiquitous Language reduces communication gaps | Requires ongoing access to domain experts                 |
| Bounded contexts prevent model corruption      | Context mapping adds architectural complexity             |
| Aggregates enforce consistency boundaries      | Cross-aggregate transactions require eventual consistency |
| Testable domain logic with no dependencies     | More classes and abstractions than simpler approaches     |

## When to Use

✅ **Good fit for:**

- Complex business domains with rich, evolving rules (insurance, finance, logistics, healthcare)
- Long-lived products where the domain model must evolve over years
- Teams with access to domain experts for ongoing collaboration
- Microservices architectures needing clear service boundary definitions
- Systems where getting the business rules wrong has significant cost

❌ **Not ideal for:**

- Simple CRUD applications with thin business logic
- Projects with tight deadlines and no access to domain experts
- Small utilities, scripts, or data pipelines
- Domains that are well-understood and unlikely to change
- Teams unwilling to invest in learning DDD vocabulary and patterns

## References

- [Domain-Driven Design: Tackling Complexity in the Heart of Software — Eric Evans (2003)](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)
- [Implementing Domain-Driven Design — Vaughn Vernon (2013)](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577)
- [Domain-Driven Design — Martin Fowler (Bliki)](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Design a DDD-Oriented Microservice — Microsoft (.NET Architecture)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [DDD Reference — Eric Evans (Free PDF)](https://www.domainlanguage.com/ddd/reference/)
- [Bounded Context — Martin Fowler](https://martinfowler.com/bliki/BoundedContext.html)
- [DDD Aggregate — Martin Fowler](https://martinfowler.com/bliki/DDD_Aggregate.html)
