# Hexagonal Architecture (Ports and Adapters)

## Overview

**Hexagonal Architecture**, also known as **Ports and Adapters**, was introduced by Alistair Cockburn in 2005. The pattern aims to create loosely coupled application components that can be easily connected to their software environment through ports and adapters.

The core idea is:

> Create your application to work without either a UI or a database so you can run automated regression-tests against the application, work when the database becomes unavailable, and link applications together without any user involvement.

### Intent

Allow an application to equally be driven by:

- Users
- Programs
- Automated tests
- Batch scripts

And to be developed and tested in isolation from its eventual runtime devices and databases.

## Core Concepts

### The Inside-Outside Asymmetry

The key insight is recognizing the **inside-outside asymmetry** rather than left-right or up-down layers:

- **Inside** — Application core (domain model + business logic)
- **Outside** — Everything else (UI, database, external services, message brokers)

The asymmetry to exploit is not between the left and right sides of the application but between the _inside_ and _outside_. Code pertaining to the inside must not leak into the outside.

### Ports

A **port** is a technology-agnostic boundary of the application. It defines a **purposeful conversation** between the application and the outside world.

- Ports are interfaces that define _how_ the application communicates
- The protocol for a port is given by the _purpose_ of the conversation
- Takes the form of an Application Programming Interface (API)
- The number of ports is a matter of design judgment — typically two to four

### Adapters

An **adapter** converts the technology-specific signals to and from the application's port interface:

- Translates between the application's language and external technologies
- Multiple adapters can implement the same port (e.g., SQL adapter, in-memory adapter, mock)
- Enables swapping technologies without changing the application core

## Architecture

```
                          ┌─────────────────────┐
                          │   Primary Adapters   │
                          │  (Driving Adapters)  │
                          │                      │
                          │  ┌────────────────┐  │
                          │  │ REST Controller │  │
                          │  └───────┬────────┘  │
                          │  ┌───────┴────────┐  │
                          │  │  CLI Handler    │  │
                          │  └───────┬────────┘  │
                          │  ┌───────┴────────┐  │
                          │  │  Test Harness   │  │
                          │  └────────────────┘  │
                          └─────────┬────────────┘
                                    │
                         ┌──────────▼──────────┐
                         │   Primary Ports      │
                         │   (Input Ports)      │
                         │   ┌──────────────┐   │
                         │   │  Interfaces  │   │
                         │   └──────────────┘   │
              ┌──────────┴─────────────────────┴──────────┐
              │                                            │
              │             APPLICATION CORE                │
              │                                            │
              │  ┌──────────────────────────────────────┐  │
              │  │           Domain Model                │  │
              │  │     (Entities, Business Rules)        │  │
              │  └──────────────────────────────────────┘  │
              │  ┌──────────────────────────────────────┐  │
              │  │         Application Services          │  │
              │  │           (Use Cases)                 │  │
              │  └──────────────────────────────────────┘  │
              │                                            │
              └──────────┬─────────────────────┬──────────┘
                         │   Secondary Ports    │
                         │   (Output Ports)     │
                         │   ┌──────────────┐   │
                         │   │  Interfaces  │   │
                         │   └──────────────┘   │
                         └──────────┬──────────┘
                                    │
                          ┌─────────▼───────────┐
                          │  Secondary Adapters  │
                          │  (Driven Adapters)   │
                          │                      │
                          │  ┌────────────────┐  │
                          │  │  SQL Database   │  │
                          │  └────────────────┘  │
                          │  ┌────────────────┐  │
                          │  │  In-Memory Mock │  │
                          │  └────────────────┘  │
                          │  ┌────────────────┐  │
                          │  │  External API   │  │
                          │  └────────────────┘  │
                          └──────────────────────┘
```

### Primary vs Secondary

#### Primary Ports (Driving / Input)

- **Purpose** — Allow external actors to _drive_ the application
- **Actor** — Primary actors initiate the conversation (users, external systems, test harnesses)
- **Implementation** — Define interfaces that the application core implements
- **Testing** — Substitute with automated test harnesses (e.g., FIT, integration tests)

**Examples:**

- User interface port
- REST / GraphQL API port
- CLI port
- Message consumer port

#### Secondary Ports (Driven / Output)

- **Purpose** — Allow the application to _drive_ external systems
- **Actor** — Secondary actors respond to the application (databases, services, queues)
- **Implementation** — Define interfaces that adapters implement
- **Testing** — Substitute with mocks, stubs, or in-memory implementations

**Examples:**

- Repository port (database access)
- Notification port (email, SMS, push)
- External service port (payment gateway, third-party API)
- Message publisher port

## Implementation

### Port Definitions

```
// Primary Port (Input) — What the application offers to the outside world
INTERFACE OrderServicePort
    FUNCTION createOrder(request : OrderRequest) -> OrderResponse
    FUNCTION getOrder(orderId : String) -> OrderResponse
END INTERFACE

// Secondary Port (Output) — What the application needs from the outside world
INTERFACE OrderRepositoryPort
    FUNCTION save(order : Order) -> Void
    FUNCTION findById(orderId : String) -> Order OR NULL
END INTERFACE

INTERFACE NotificationPort
    FUNCTION sendOrderConfirmation(email : String, order : Order) -> Void
END INTERFACE
```

### Application Core

```
// Application Service — Implements primary port, depends on secondary ports
CLASS OrderService IMPLEMENTS OrderServicePort

    CONSTRUCTOR(orderRepository : OrderRepositoryPort,
                notificationService : NotificationPort)

    FUNCTION createOrder(request : OrderRequest) -> OrderResponse
        // Business logic using domain entities
        order = Order.create(
            customerId = request.customerId,
            items      = request.items
        )

        // Use secondary ports — application is unaware of the technology behind them
        orderRepository.save(order)
        notificationService.sendOrderConfirmation(order.customerEmail, order)

        RETURN NEW OrderResponse(
            orderId = order.id,
            total   = order.total,
            status  = order.status
        )
END CLASS
```

### Adapters

```
// ── Primary Adapter (Driving) — REST Controller ──
CLASS OrderRestController
    CONSTRUCTOR(orderService : OrderServicePort)

    ENDPOINT POST "/orders"
    FUNCTION handleCreateOrder(httpRequest) -> HttpResponse
        request = parseBody(httpRequest, OrderRequest)
        response = orderService.createOrder(request)
        RETURN HttpResponse(status = 201, body = response)
END CLASS

// ── Secondary Adapter (Driven) — SQL Repository ──
CLASS SqlOrderRepository IMPLEMENTS OrderRepositoryPort
    CONSTRUCTOR(dbSession : DatabaseSession)

    FUNCTION save(order : Order) -> Void
        dbModel = OrderMapper.toDbModel(order)
        dbSession.add(dbModel)
        dbSession.commit()

    FUNCTION findById(orderId : String) -> Order OR NULL
        dbModel = dbSession.query(OrderTable).whereId(orderId)
        IF dbModel IS NULL THEN RETURN NULL
        RETURN OrderMapper.toDomain(dbModel)
END CLASS

// ── Secondary Adapter (Driven) — In-Memory Mock for Testing ──
CLASS InMemoryOrderRepository IMPLEMENTS OrderRepositoryPort
    PROPERTIES
        orders : Map<String, Order> = EMPTY MAP

    FUNCTION save(order : Order) -> Void
        orders.PUT(order.id, order)

    FUNCTION findById(orderId : String) -> Order OR NULL
        RETURN orders.GET(orderId) OR NULL
END CLASS
```

## Project Structure

```
src/
├── core/                           # Application Core (Hexagon Interior)
│   ├── domain/
│   │   ├── entities/
│   │   └── value_objects/
│   ├── ports/
│   │   ├── input/                  # Primary Ports
│   │   └── output/                 # Secondary Ports
│   └── services/                   # Application Services
│
├── adapters/                       # Adapters (Hexagon Exterior)
│   ├── primary/                    # Driving Adapters
│   │   ├── rest/
│   │   ├── cli/
│   │   └── grpc/
│   └── secondary/                  # Driven Adapters
│       ├── persistence/
│       ├── notifications/
│       └── external_services/
│
├── config/                         # Composition Root / DI Container
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

### The Hexagon Shape

The hexagon is not significant for the number six. It simply:

- Provides visual room for inserting ports and adapters as needed
- Breaks free from one-dimensional layered thinking
- Emphasizes the **inside-outside** nature of the architecture

> Typical applications have two to four ports. The "wrong" number of ports does not cause noticeable damage — it remains a matter of design intuition. — Alistair Cockburn

## Benefits

1. **Technology Independence** — Swap databases, UI frameworks, or external services without changing business logic
2. **Testability** — Test the application core in isolation with mock adapters
3. **Symmetry** — All external dependencies are treated uniformly regardless of direction
4. **Flexibility** — Support multiple interfaces simultaneously (REST, CLI, GraphQL, gRPC)
5. **Isolation** — Develop and test the application before infrastructure decisions are finalized

## Trade-offs

| Advantage                       | Consideration                                   |
| ------------------------------- | ----------------------------------------------- |
| Clean separation of concerns    | Additional interfaces and adapter classes       |
| Swappable infrastructure        | Indirection adds complexity for simple apps     |
| Excellent testability           | Requires team discipline to maintain boundaries |
| Multiple entry points supported | More upfront design effort                      |

## When to Use

✅ **Good fit for:**

- Applications with multiple entry points (API, CLI, events, scheduled jobs)
- Systems requiring swappable external dependencies
- Projects needing comprehensive, isolated test coverage
- Long-lived applications with evolving infrastructure
- Microservices with clear domain boundaries

❌ **Not ideal for:**

- Simple applications with a single entry point
- Prototypes where infrastructure is unlikely to change
- Scripts or utilities with minimal external dependencies
- Applications where development speed outweighs architectural flexibility

## References

- [Hexagonal Architecture — Alistair Cockburn (Original 2005 Article)](https://alistair.cockburn.us/hexagonal-architecture/)
- [Growing Object-Oriented Software, Guided by Tests — Steve Freeman & Nat Pryce](https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627)
- [Ports and Adapters Pattern — Mark Seemann](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/)
- [Object Design — Rebecca Wirfs-Brock & Alan McKean](https://www.amazon.com/Object-Design-Roles-Responsibilities-Collaborations/dp/0201379430)
