# Onion Architecture

## Overview

**Onion Architecture** was introduced by Jeffrey Palermo in 2008 as an architectural pattern that emphasizes separation of concerns and dependency management. The architecture uses concentric layers, like an onion, where dependencies always point inward toward the domain model at the center.

The core principle is:

> All code can depend on layers more central, but code cannot depend on layers further out from the core. All coupling is toward the center.

### Key Principles

1. **The application is built around an independent object model**
2. **Inner layers define interfaces; outer layers implement interfaces**
3. **Direction of coupling is toward the center**
4. **All application core code can be compiled and run separately from infrastructure**

### Why "Onion"?

The name comes from the visual representation of the architecture as concentric circles (layers), similar to an onion. Each layer wraps around the previous one, and you must peel through outer layers to reach the core.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Infrastructure                                  │
│   (UI, Tests, Database, External Services, IoC Container)                │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Application Services                          │  │
│  │           (Use Cases, Application Logic, DTOs)                     │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Domain Services                           │  │  │
│  │  │        (Domain Logic spanning multiple Entities)             │  │  │
│  │  │  ┌───────────────────────────────────────────────────────┐  │  │  │
│  │  │  │                   Domain Model                         │  │  │  │
│  │  │  │         (Entities, Value Objects, Interfaces)          │  │  │  │
│  │  │  │                                                        │  │  │  │
│  │  │  │              ← Dependencies flow inward                │  │  │  │
│  │  │  │                                                        │  │  │  │
│  │  │  └───────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Layer 1: Domain Model (Core)

The very center of the architecture:

- **Entities** — Business objects that have identity and lifecycle
- **Value Objects** — Immutable objects defined by their attributes
- **Domain Interfaces** — Abstractions for persistence and other concerns (defined here, implemented outside)
- **Domain Events** — Events that represent something meaningful that happened in the domain

**Key Characteristics:**

- No dependencies on anything outside itself
- Contains plain objects (POCOs / POJOs) with no infrastructure concerns
- Pure business logic and domain rules only

```
// Domain Entity — No external dependencies
ENTITY Order
    PROPERTIES
        id         : UUID = GENERATE_UUID()
        customerId : UUID
        items      : List<OrderItem> = EMPTY LIST
        status     : String = "pending"
        createdAt  : DateTime = NOW()

    FUNCTION total() -> Decimal
        RETURN SUM(item.subtotal FOR EACH item IN items)

    FUNCTION addItem(productId : UUID, quantity : Integer, price : Decimal)
        IF quantity <= 0 THEN
            THROW ValidationError("Quantity must be positive")
        END IF
        items.ADD(NEW OrderItem(productId, quantity, price))

    FUNCTION submit()
        IF items IS EMPTY THEN
            THROW ValidationError("Cannot submit empty order")
        END IF
        status = "submitted"
END ENTITY

// Domain Interface — Defined in core, implemented in outer layers
INTERFACE OrderRepository
    FUNCTION save(order : Order) -> Void
    FUNCTION findById(orderId : UUID) -> Order OR NULL
    FUNCTION findByCustomer(customerId : UUID) -> List<Order>
END INTERFACE
```

### Layer 2: Domain Services

Contains domain logic that doesn't naturally fit within a single entity:

- Cross-entity business rules
- Domain calculations and policies
- Orchestration of multiple entities for domain operations

```
// Domain service for complex pricing logic
CLASS OrderPricingService

    FUNCTION calculateDiscount(order : Order, customer : Customer) -> Decimal
        baseDiscount = 0.0

        // Loyalty discount
        IF customer.isPremium THEN
            baseDiscount = baseDiscount + 0.10
        END IF

        // Volume discount
        IF order.total() > 1000 THEN
            baseDiscount = baseDiscount + 0.05
        END IF

        RETURN MIN(baseDiscount, 0.25)  // Cap at 25%
END CLASS

// Domain service for inventory validation
CLASS InventoryDomainService
    CONSTRUCTOR(inventoryRepository : InventoryRepository)

    FUNCTION canFulfillOrder(order : Order) -> Boolean
        FOR EACH item IN order.items
            available = inventoryRepository.getAvailableQuantity(item.productId)
            IF available < item.quantity THEN
                RETURN FALSE
            END IF
        END FOR
        RETURN TRUE
END CLASS
```

### Layer 3: Application Services

Orchestrates use cases and application workflow:

- Implements use cases by coordinating domain objects and domain services
- Handles transactions (Unit of Work)
- Maps between DTOs and domain objects
- Does **not** contain business rules — delegates to domain layer

```
// DTOs for application boundary
DATA CreateOrderRequest
    customerId : UUID
    items      : List<ItemData>
END DATA

DATA CreateOrderResponse
    orderId : UUID
    total   : Decimal
    status  : String
END DATA

// Application Service — Orchestrates use cases
CLASS OrderApplicationService
    CONSTRUCTOR(
        orderRepository    : OrderRepository,
        customerRepository : CustomerRepository,
        inventoryService   : InventoryDomainService,
        pricingService     : OrderPricingService,
        unitOfWork         : UnitOfWork
    )

    FUNCTION createOrder(request : CreateOrderRequest) -> CreateOrderResponse
        BEGIN TRANSACTION via unitOfWork

            customer = customerRepository.findById(request.customerId)
            IF customer IS NULL THEN
                THROW CustomerNotFoundError(request.customerId)
            END IF

            order = NEW Order(customerId = request.customerId)

            FOR EACH item IN request.items
                order.addItem(item.productId, item.quantity, item.price)
            END FOR

            IF NOT inventoryService.canFulfillOrder(order) THEN
                THROW InsufficientInventoryError()
            END IF

            discount = pricingService.calculateDiscount(order, customer)
            order.applyDiscount(discount)

            orderRepository.save(order)
            unitOfWork.commit()

        END TRANSACTION

        RETURN NEW CreateOrderResponse(
            orderId = order.id,
            total   = order.total(),
            status  = order.status
        )
END CLASS
```

### Layer 4: Infrastructure

The outermost layer containing all external concerns:

- **UI** — Web controllers, CLI, GraphQL resolvers
- **Persistence** — Database implementations, ORM configurations
- **External Services** — API clients, message queues
- **IoC Container** — Dependency injection configuration
- **Tests** — Integration and end-to-end tests

```
// ── Repository Implementation ──
CLASS SqlOrderRepository IMPLEMENTS OrderRepository
    CONSTRUCTOR(dbSession : DatabaseSession)

    FUNCTION save(order : Order) -> Void
        dbModel = mapToDbModel(order)
        dbSession.add(dbModel)

    FUNCTION findById(orderId : UUID) -> Order OR NULL
        dbModel = dbSession.query(OrderTable).whereId(orderId)
        IF dbModel IS NULL THEN RETURN NULL
        RETURN mapToDomain(dbModel)
END CLASS

// ── Web Controller ──
CLASS OrderController
    CONSTRUCTOR(orderService : OrderApplicationService)

    ENDPOINT POST "/orders"
    FUNCTION handleCreateOrder(httpRequest) -> HttpResponse
        request = parseBody(httpRequest, CreateOrderRequest)
        response = orderService.createOrder(request)
        RETURN HttpResponse(status = 201, body = response)
END CLASS

// ── IoC / Dependency Injection Configuration ──
CONFIGURATION Container
    // Database
    REGISTER DatabaseSession

    // Repositories (implement domain interfaces)
    REGISTER SqlOrderRepository AS OrderRepository
    REGISTER SqlCustomerRepository AS CustomerRepository

    // Domain Services
    REGISTER OrderPricingService
    REGISTER InventoryDomainService

    // Application Services
    REGISTER OrderApplicationService
END CONFIGURATION
```

## Project Structure

```
src/
├── domain/                         # Layers 1 & 2 (Domain Model + Domain Services)
│   ├── model/
│   │   ├── entities/
│   │   ├── value_objects/
│   │   └── events/
│   ├── services/
│   └── interfaces/                 # Repository interfaces (defined here)
│
├── application/                    # Layer 3 (Application Services)
│   ├── services/
│   ├── dto/
│   └── interfaces/                 # Unit of Work interface
│
├── infrastructure/                 # Layer 4 (Infrastructure)
│   ├── persistence/
│   │   ├── models/                 # ORM / database models
│   │   ├── repositories/          # Interface implementations
│   │   └── unit_of_work/
│   ├── web/
│   │   └── controllers/
│   ├── messaging/
│   └── config/                     # IoC container, settings
│
└── tests/
    ├── unit/
    │   └── domain/
    ├── integration/
    │   └── infrastructure/
    └── e2e/
```

## Key Design Considerations

### Dependency Inversion in Practice

The Onion Architecture relies heavily on the **Dependency Inversion Principle**:

```
Domain Model (center)
    ↑
    │ Defines interfaces (abstractions)
    │
Domain Services
    ↑
    │ Uses domain interfaces
    │
Application Services
    ↑
    │ Orchestrates use cases
    │
Infrastructure (outer)
    │
    └── Implements interfaces (concrete details)
```

The application core defines _what_ it needs through interfaces. The infrastructure layer provides _how_ those needs are fulfilled, injecting implementations at runtime.

### Database is External

A key mindset shift:

> There are no "database applications." There are applications that _might use_ a database as a storage service.

The database is externalized just like the UI. The domain model does not know or care how data is persisted. This enables:

- Switching databases without touching business logic
- Running the full domain in tests with in-memory stores
- Deploying the same core against different storage backends

## Benefits

1. **Domain-Centric** — Business logic is at the heart of the architecture, not databases or frameworks
2. **Testability** — Core layers can be tested without infrastructure using mocks/stubs
3. **Flexibility** — External dependencies can be swapped with minimal impact
4. **Maintainability** — Clear separation reduces coupling and makes changes predictable
5. **Technology Independence** — The database is a detail, not the center of the system

## Trade-offs

| Advantage                   | Consideration                          |
| --------------------------- | -------------------------------------- |
| Strong domain focus         | More layers mean more mapping code     |
| Infrastructure independence | Requires upfront interface design      |
| High testability            | Overhead for simple CRUD use cases     |
| Clear dependency direction  | Team must understand and enforce rules |

## When to Use

✅ **Good fit for:**

- Long-lived enterprise applications
- Complex domain logic requiring Domain-Driven Design
- Applications requiring high testability and maintainability
- Teams practicing DDD with aggregates, repositories, and domain services

❌ **Not ideal for:**

- Simple CRUD applications
- Small utility applications or scripts
- Prototypes with tight deadlines
- Applications with minimal business logic

## References

- [The Onion Architecture: Part 1 — Jeffrey Palermo](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
- [The Onion Architecture: Part 2 — Jeffrey Palermo](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-2/)
- [The Onion Architecture: Part 3 — Jeffrey Palermo](https://jeffreypalermo.com/2008/08/the-onion-architecture-part-3/)
- [Onion Architecture: Part 4 — After Four Years](https://jeffreypalermo.com/2013/08/onion-architecture-part-4-after-four-years/)
- [Domain-Driven Design — Eric Evans](https://domainlanguage.com/ddd/)
- [Layers, Onions, Ports, Adapters: It's All the Same — Mark Seemann](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/)
