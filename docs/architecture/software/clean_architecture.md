# Clean Architecture

## Overview

**Clean Architecture** is a software design philosophy introduced by Robert C. Martin (Uncle Bob) in 2012. It synthesizes ideas from several earlier architectural patterns — [Hexagonal Architecture](hexagonal_architecture.md), [Onion Architecture](onion_architecture.md), DCI, and BCE — with the goal of creating systems that are:

- **Independent of Frameworks** — The architecture does not depend on the existence of any library or feature-laden software. Frameworks are tools, not constraints.
- **Testable** — Business rules can be tested without UI, database, web server, or any other external element.
- **Independent of UI** — The UI can change easily without changing the rest of the system. A web UI could be replaced with a console UI without changing business rules.
- **Independent of Database** — Business rules are not bound to any particular database. You can swap out storage technologies freely.
- **Independent of External Agencies** — Business rules know nothing about the outside world.

## Core Concepts

### The Dependency Rule

The overriding rule that makes this architecture work is **The Dependency Rule**:

> Source code dependencies can only point inwards. Nothing in an inner circle can know anything at all about something in an outer circle.

This means:

- Inner layers define interfaces (abstractions)
- Outer layers implement those interfaces (concrete details)
- Data formats used in outer circles must not be used by inner circles
- Names declared in outer circles must not be referenced by code in inner circles

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  Interface Adapters                        │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │           Application Business Rules                │  │  │
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │
│  │  │  │         Enterprise Business Rules             │  │  │  │
│  │  │  │              (Entities)                       │  │  │  │
│  │  │  └───────────────────────────────────────────────┘  │  │  │
│  │  │                  (Use Cases)                         │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │           (Controllers, Presenters, Gateways)              │  │
│  └───────────────────────────────────────────────────────────┘  │
│             (Web, UI, DB, Devices, External Interfaces)         │
└─────────────────────────────────────────────────────────────────┘

        Dependencies always point INWARD →
```

> **Note:** The circles are schematic. There is no rule that says you must always have exactly four. However, _The Dependency Rule_ always applies — source code dependencies always point inwards.

### 1. Entities (Enterprise Business Rules)

The innermost layer containing enterprise-wide business rules:

- Encapsulate the most general and high-level rules
- Can be objects with methods, or data structures with functions
- Least likely to change when something external changes
- Could be used by many different applications in the enterprise

**Contains:**

- Domain entities (e.g., `User`, `Order`, `Product`)
- Value objects (e.g., `Money`, `Address`)
- Domain events
- Business rule validations

```
// Entity — No external dependencies
ENTITY Order
    PROPERTIES
        id          : UUID
        customerId  : UUID
        items       : List<OrderItem>
        status      : String = "pending"
        createdAt   : DateTime

    FUNCTION total() -> Decimal
        RETURN SUM(item.subtotal FOR EACH item IN items)

    FUNCTION addItem(productId, quantity, price)
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
```

### 2. Use Cases (Application Business Rules)

Contains application-specific business rules:

- Orchestrates the flow of data to and from entities
- Directs entities to use their business rules to achieve use case goals
- Changes here affect the application but not the entities
- Isolated from external concerns (database, UI, frameworks)

**Contains:**

- Use case interactors / application services
- Input ports (interfaces that use cases implement)
- Output ports (interfaces that use cases depend on)
- Data Transfer Objects (DTOs)

```
// Input Port — defines what the application can do
INTERFACE CreateOrderInputPort
    FUNCTION execute(request : CreateOrderRequest) -> CreateOrderResponse
END INTERFACE

// Output Port — defines what the application needs
INTERFACE OrderRepositoryPort
    FUNCTION save(order : Order) -> Void
    FUNCTION findById(orderId : UUID) -> Order OR NULL
END INTERFACE

// Use Case Interactor — implements input port, depends on output ports
CLASS CreateOrderUseCase IMPLEMENTS CreateOrderInputPort
    CONSTRUCTOR(orderRepository : OrderRepositoryPort,
                customerRepository : CustomerRepositoryPort)

    FUNCTION execute(request : CreateOrderRequest) -> CreateOrderResponse
        customer = customerRepository.findById(request.customerId)
        IF customer IS NULL THEN
            THROW CustomerNotFoundError(request.customerId)
        END IF

        order = NEW Order(customerId = request.customerId)

        FOR EACH item IN request.items
            order.addItem(item.productId, item.quantity, item.price)
        END FOR

        orderRepository.save(order)

        RETURN NEW CreateOrderResponse(
            orderId = order.id,
            total   = order.total(),
            status  = order.status
        )
END CLASS
```

### 3. Interface Adapters

Converts data between the format most convenient for use cases/entities and the format most convenient for external agencies:

- Contains Controllers, Presenters, and Gateways
- MVC architecture of the GUI belongs in this layer
- Converts data from database format to entity format and vice versa
- No code inward of this circle should know about the database

**Contains:**

- REST/GraphQL controllers
- ViewModels and presenters
- Repository implementations
- Data mappers and ORM configurations

```
// Controller — Primary adapter (drives the application)
CLASS OrderController
    CONSTRUCTOR(createOrder : CreateOrderInputPort)

    FUNCTION handleCreateOrder(httpRequest) -> HttpResponse
        request = NEW CreateOrderRequest(
            customerId = httpRequest.body.customerId,
            items      = httpRequest.body.items
        )
        response = createOrder.execute(request)
        RETURN HttpResponse(status = 201, body = response)
END CLASS

// Presenter — Transforms output for the view
CLASS OrderPresenter IMPLEMENTS OrderOutputPort
    FUNCTION presentOrder(response : CreateOrderResponse) -> OrderViewModel
        RETURN NEW OrderViewModel(
            displayId      = "ORD-" + response.orderId,
            formattedTotal = formatCurrency(response.total)
        )
END CLASS

// Repository Implementation — Secondary adapter (driven by the application)
CLASS SqlOrderRepository IMPLEMENTS OrderRepositoryPort
    CONSTRUCTOR(dbSession : DatabaseSession)

    FUNCTION save(order : Order) -> Void
        dbModel = mapToDbModel(order)
        dbSession.add(dbModel)
        dbSession.commit()

    FUNCTION findById(orderId : UUID) -> Order OR NULL
        dbModel = dbSession.query(OrderTable).whereId(orderId)
        IF dbModel IS NULL THEN RETURN NULL
        RETURN mapToDomain(dbModel)
END CLASS
```

### 4. Frameworks & Drivers

The outermost layer composed of frameworks and tools:

- Database engines and ORMs
- Web frameworks
- UI frameworks
- External service clients
- Contains primarily glue code that communicates with inner layers

**Examples:**

- Web frameworks (ASP.NET Core, Spring Boot, FastAPI, Express)
- ORM libraries (Entity Framework, Hibernate, SQLAlchemy, GORM)
- UI frameworks (React, Angular, Blazor)
- Message brokers (RabbitMQ, Kafka clients)
- Dependency injection containers

### Crossing Boundaries

When data crosses boundaries, it must always be in the form most convenient for the _inner_ circle:

- Use simple data structures or DTOs — not entities or database rows
- Apply the **Dependency Inversion Principle** to cross boundaries: the use case defines an output port (interface), and the outer layer implements it
- The flow of control may go outward, but source code dependencies always point inward

```
// Flow of control vs. dependency direction
//
//   Controller ──calls──▶ Use Case ──calls──▶ Presenter
//       │                    │                    ▲
//       │                    │                    │
//       ▼                    ▼                    │
//   (outer layer)      (inner layer)        (outer layer)
//
// The Use Case calls an OutputPort interface (defined in the inner layer).
// The Presenter implements that interface (in the outer layer).
// Source code dependency: Presenter depends on Use Case layer, not vice versa.
```

## Project Structure

```
src/
├── domain/                         # Entities Layer
│   ├── entities/
│   ├── value_objects/
│   └── exceptions/
│
├── application/                    # Use Cases Layer
│   ├── use_cases/
│   ├── ports/
│   │   ├── input/                  # Input ports (interfaces)
│   │   └── output/                 # Output ports (interfaces)
│   └── dto/
│
├── adapters/                       # Interface Adapters Layer
│   ├── controllers/
│   ├── presenters/
│   └── repositories/
│
├── infrastructure/                 # Frameworks & Drivers Layer
│   ├── database/
│   ├── web/
│   ├── messaging/
│   └── config/
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

## Benefits

1. **Testability** — Core business logic can be tested in complete isolation from infrastructure
2. **Maintainability** — Changes in external systems (DB, UI, services) do not affect business rules
3. **Flexibility** — Easy to swap frameworks, databases, or UIs with minimal impact
4. **Focus on Domain** — Forces developers to focus on business logic before infrastructure
5. **Clear Boundaries** — Well-defined separation between layers reduces accidental coupling
6. **Parallel Development** — Teams can work on different layers independently

## Trade-offs

| Advantage                     | Consideration                                |
| ----------------------------- | -------------------------------------------- |
| Strong separation of concerns | More boilerplate (interfaces, DTOs, mappers) |
| Framework independence        | Indirection adds initial complexity          |
| High testability              | Overhead not justified for simple CRUD apps  |
| Technology swappability       | Requires discipline to maintain boundaries   |

## When to Use

✅ **Good fit for:**

- Long-lived enterprise applications
- Applications with complex or evolving business logic
- Teams that need to support multiple UIs, APIs, or databases
- Projects requiring high testability and maintainability
- Microservices with rich domain logic

❌ **Not ideal for:**

- Simple CRUD applications with minimal business logic
- Prototypes or MVPs with tight deadlines
- Small utilities or scripts
- Applications where speed-to-market outweighs architectural purity

## References

- [The Clean Architecture — Robert C. Martin (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Clean Architecture: A Craftsman's Guide to Software Structure and Design — Robert C. Martin](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
- [Common Web Application Architectures — Microsoft](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [Screaming Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)
