# Data Architecture

## Overview

**Data Architecture** defines how data is collected, stored, managed, integrated, and used within a system. It provides the foundational blueprint for organizing data assets, defining data flows, and selecting appropriate storage technologies. Rather than prescribing a single database, data architecture addresses the full lifecycle of data — from ingestion and modeling through querying, consistency, and evolution.

Core principles:

- **Data Ownership** — Every piece of data has a clear owner responsible for its schema, quality, and lifecycle
- **Polyglot Persistence** — Use the storage technology best suited to each data access pattern rather than forcing all data into one model
- **Consistency Boundaries** — Explicitly define where strong (ACID) consistency is required versus where eventual consistency is acceptable
- **Schema Evolution** — Data models change over time; design for backward/forward-compatible schema changes from the start
- **Separation of Read and Write Concerns** — Optimize write paths for integrity and read paths for performance independently

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Data Architecture                                  │
│                                                                             │
│   Data Sources                                                              │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                      │
│   │ User     │ │ External │ │ IoT /    │ │ Internal │                       │
│   │ Input    │ │ APIs     │ │ Sensors  │ │ Events   │                       │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                      │
│        │             │            │             │                            │
│        ▼             ▼            ▼             ▼                            │
│   ┌─────────────────────────────────────────────────────┐                   │
│   │                 Data Ingestion Layer                  │                   │
│   │   (validation, transformation, routing)               │                   │
│   └───────────┬─────────────────┬───────────────────────┘                   │
│               │                 │                                            │
│     ┌─────────▼──────┐   ┌─────▼──────────────┐                            │
│     │  Operational    │   │  Analytical         │                            │
│     │  Data Stores    │   │  Data Stores        │                            │
│     │                 │   │                     │                            │
│     │  ┌───────────┐ │   │  ┌───────────────┐  │                            │
│     │  │ Relational│ │   │  │ Data Warehouse│  │                            │
│     │  │ (ACID)    │ │   │  │ (Star Schema) │  │                            │
│     │  └───────────┘ │   │  └───────────────┘  │                            │
│     │  ┌───────────┐ │   │  ┌───────────────┐  │                            │
│     │  │ Document  │ │   │  │ Data Lake     │  │                            │
│     │  │ Store     │ │   │  │ (Raw + Curated│  │                            │
│     │  └───────────┘ │   │  └───────────────┘  │                            │
│     │  ┌───────────┐ │   │  ┌───────────────┐  │                            │
│     │  │ Key-Value │ │   │  │ Search Index  │  │                            │
│     │  │ / Cache   │ │   │  │               │  │                            │
│     │  └───────────┘ │   │  └───────────────┘  │                            │
│     └────────────────┘   └─────────────────────┘                            │
│                                                                             │
│   Cross-Cutting: ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐  │
│                   │ Data Catalog │ │ Data Quality │ │ Schema Registry   │  │
│                   │ & Lineage    │ │ & Governance │ │ & Versioning      │  │
│                   └──────────────┘ └──────────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Data Modeling

Data modeling defines the structure, relationships, and constraints of data. Three levels of abstraction guide the process:

**Conceptual Model** — Business-level entities and relationships (no technology details):

```
// Conceptual: What entities exist and how they relate
ENTITY Customer
    HAS MANY Orders
    HAS ONE Profile
    BELONGS TO Region

ENTITY Order
    HAS MANY LineItems
    BELONGS TO Customer
    HAS ONE ShippingAddress
```

**Logical Model** — Detailed attributes, keys, and relationship cardinality:

```
// Logical: Attributes, types, and keys defined
ENTITY Customer
    PROPERTIES
        customerId  : UUID            [PRIMARY KEY]
        email       : String          [UNIQUE, NOT NULL]
        name        : String          [NOT NULL]
        createdAt   : DateTime        [NOT NULL]

ENTITY Order
    PROPERTIES
        orderId     : UUID            [PRIMARY KEY]
        customerId  : UUID            [FOREIGN KEY -> Customer]
        status      : Enum(pending, confirmed, shipped, delivered)
        total       : Decimal         [NOT NULL]
        orderedAt   : DateTime        [NOT NULL]

    RELATIONSHIP
        Customer (1) ──── (many) Order
        Order    (1) ──── (many) LineItem
```

**Physical Model** — Technology-specific implementation (indexes, partitions, storage engine choices):

```
// Physical: Technology-specific decisions
TABLE customers
    PARTITION BY region
    INDEX ON email (unique)
    INDEX ON createdAt (for range queries)

TABLE orders
    PARTITION BY orderedAt (monthly)
    INDEX ON customerId (for lookups)
    INDEX ON status, orderedAt (for filtered scans)
```

### Consistency Models

Consistency determines the guarantees a system provides about data correctness after writes:

#### ACID (Strong Consistency)

Atomicity, Consistency, Isolation, Durability — all operations within a transaction succeed or all fail. The data is always in a valid state.

```
// ACID transaction — all-or-nothing
FUNCTION transferFunds(fromAccount, toAccount, amount)
    BEGIN TRANSACTION
        balance = QUERY("SELECT balance FROM accounts WHERE id = ?", fromAccount)

        IF balance < amount THEN
            ROLLBACK
            THROW InsufficientFundsError
        END IF

        EXECUTE("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, fromAccount)
        EXECUTE("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, toAccount)
        EXECUTE("INSERT INTO transfers (from, to, amount, timestamp) VALUES (?, ?, ?, NOW())",
                fromAccount, toAccount, amount)
    COMMIT TRANSACTION
END FUNCTION
```

#### BASE (Eventual Consistency)

Basically Available, Soft state, Eventually consistent — the system prioritizes availability and partition tolerance; replicas converge over time.

```
// BASE — eventual consistency with event-driven propagation
FUNCTION placeOrder(order)
    // Write to the order service's own store (immediate)
    orderStore.save(order)

    // Publish event — other services will eventually catch up
    eventBus.publish(NEW OrderPlacedEvent(
        orderId    = order.id,
        customerId = order.customerId,
        items      = order.items,
        total      = order.total
    ))
    // Inventory service, shipping service, analytics
    // will process this event asynchronously
END FUNCTION
```

#### CAP Theorem

In a distributed system experiencing a network partition, you must choose between **Consistency** (every read returns the latest write) and **Availability** (every request receives a response). No distributed system can guarantee all three simultaneously.

```
// Decision framework
WHEN designing a distributed data store:

    IF data requires financial accuracy (banking, billing)
        CHOOSE Consistency + Partition Tolerance (CP)
        ACCEPT that some requests may be rejected during partitions
        EXAMPLE: Relational database with synchronous replication

    ELSE IF data requires high availability (social feeds, product catalog)
        CHOOSE Availability + Partition Tolerance (AP)
        ACCEPT that reads may return slightly stale data
        EXAMPLE: Document store with eventual consistency

    NOTE: In practice, most systems are not purely CP or AP
          — they tune consistency per operation or per data type
```

### Data Store Categories

Different data shapes demand different storage engines:

```
┌─────────────────────┬──────────────────────────────┬──────────────────────────┐
│ Store Type          │ Best For                     │ Example Use Cases         │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Relational (SQL)    │ Structured data, complex     │ Financial transactions,  │
│                     │ joins, ACID transactions     │ user accounts, inventory │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Document            │ Semi-structured, flexible    │ Product catalogs, CMS,   │
│                     │ schema, nested objects       │ user profiles            │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Key-Value           │ Simple lookups, high         │ Session storage, caching,│
│                     │ throughput, low latency      │ feature flags            │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Column-Family       │ Wide rows, time-series,      │ IoT telemetry, event    │
│                     │ write-heavy workloads        │ logs, analytics          │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Graph               │ Highly connected data,       │ Social networks,         │
│                     │ relationship traversal       │ recommendations, fraud   │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Search Engine       │ Full-text search, faceted    │ Product search, log      │
│                     │ queries, scoring             │ analysis, autocomplete   │
├─────────────────────┼──────────────────────────────┼──────────────────────────┤
│ Time-Series         │ Timestamped data, downsampl- │ Metrics, monitoring,     │
│                     │ ing, range queries by time   │ financial tick data      │
└─────────────────────┴──────────────────────────────┴──────────────────────────┘
```

### Polyglot Persistence

Use the right store for each data access pattern rather than forcing everything into one database:

```
// E-commerce system using polyglot persistence
SYSTEM ECommercePlatform

    // Transactional data → Relational (ACID required)
    OrderService
        STORE: RelationalDatabase
        REASON: Complex joins between orders, line items, customers
                ACID transactions for payment processing

    // Product catalog → Document Store (flexible schema)
    CatalogService
        STORE: DocumentDatabase
        REASON: Products have variable attributes (clothing vs electronics)
                Schema-on-read allows flexible product definitions

    // Session & cart → Key-Value Store (low latency, ephemeral)
    CartService
        STORE: KeyValueStore
        REASON: Simple get/set by session ID
                High throughput, data is short-lived

    // Product search → Search Engine (full-text, facets)
    SearchService
        STORE: SearchIndex
        REASON: Full-text search with relevance scoring
                Faceted navigation (filter by brand, price, rating)

    // Recommendations → Graph Database (relationship traversal)
    RecommendationService
        STORE: GraphDatabase
        REASON: "Customers who bought X also bought Y"
                Efficient traversal of purchase relationships

END SYSTEM
```

### Data Partitioning

Partitioning splits data across multiple nodes for scalability:

**Horizontal Partitioning (Sharding)** — Rows distributed across shards by a partition key:

```
// Shard by customer region
FUNCTION resolvePartition(customerId)
    region = customerRegistry.getRegion(customerId)

    SWITCH region
        CASE "north-america" -> RETURN shard_na
        CASE "europe"        -> RETURN shard_eu
        CASE "asia-pacific"  -> RETURN shard_apac
        DEFAULT              -> RETURN shard_default
    END SWITCH
END FUNCTION

// Hash-based sharding for even distribution
FUNCTION resolvePartitionByHash(entityId, totalShards)
    shardIndex = HASH(entityId) MOD totalShards
    RETURN shards[shardIndex]
END FUNCTION
```

**Vertical Partitioning** — Columns split across stores based on access patterns:

```
// Frequently accessed columns in a fast store
FAST_STORE user_profiles
    userId, name, avatarUrl, lastSeen

// Rarely accessed columns in a separate store
ARCHIVE_STORE user_details
    userId, dateOfBirth, taxId, auditLog, preferences
```

### Schema Evolution

Schemas change over time. Design for backward and forward compatibility:

```
// Strategy 1: Additive changes only (backward compatible)
// v1 schema
DATA UserEvent_v1
    userId : UUID
    name   : String
    email  : String

// v2 schema — new optional field, old consumers still work
DATA UserEvent_v2
    userId  : UUID
    name    : String
    email   : String
    phone   : String OR NULL    // New field, optional

// Strategy 2: Schema registry with compatibility checks
INTERFACE SchemaRegistry
    FUNCTION register(subject, schema, compatibilityMode) -> SchemaVersion
    FUNCTION validate(subject, newSchema) -> CompatibilityResult
    FUNCTION getLatest(subject) -> Schema

// Compatibility modes:
// BACKWARD  — new schema can read data written by old schema
// FORWARD   — old schema can read data written by new schema
// FULL      — both backward and forward compatible
// NONE      — no compatibility guarantee

// Strategy 3: Migration scripts with versioning
MIGRATION "003_add_phone_to_users"
    UP:
        ALTER TABLE users ADD COLUMN phone String NULLABLE
        // Backfill from external source if needed
    DOWN:
        ALTER TABLE users DROP COLUMN phone
```

### Data Mesh

Data Mesh is an organizational and architectural approach for large-scale data management. It applies domain-driven design principles to data:

```
// Four pillars of Data Mesh

// 1. Domain-Oriented Ownership — each domain owns its data
DOMAIN OrdersDomain
    OWNS: orders data, order events, order analytics
    TEAM: Orders squad
    PUBLISHES: OrderDataProduct

// 2. Data as a Product — treat shared data with product thinking
DATA PRODUCT OrderDataProduct
    OWNER:        OrdersDomain
    SLA:          99.9% availability, < 5 minute freshness
    SCHEMA:       registered in central catalog
    DISCOVERABLE: listed in data catalog with documentation
    QUALITY:      automated checks for completeness, freshness, accuracy

    INTERFACE
        FUNCTION query(filters) -> ResultSet   // Self-serve access
        FUNCTION subscribe() -> EventStream    // Real-time stream

// 3. Self-Serve Data Platform — infrastructure that enables domains
PLATFORM DataPlatform
    PROVIDES: storage provisioning, pipeline templates,
              schema registry, access control, monitoring

// 4. Federated Computational Governance — shared standards
GOVERNANCE FederatedGovernance
    ENFORCES: naming conventions, security policies, SLA standards
    ENABLES:  cross-domain interoperability
    REVIEWS:  data product quality periodically
```

## Implementation

### Repository Pattern for Data Access

Abstract data access behind repository interfaces so the storage technology is swappable:

```
// Port — technology-agnostic interface
INTERFACE OrderRepository
    FUNCTION save(order : Order) -> Void
    FUNCTION findById(id : UUID) -> Order OR NULL
    FUNCTION findByCustomer(customerId : UUID, pagination : Page) -> PagedResult<Order>
    FUNCTION delete(id : UUID) -> Void

// Adapter — relational implementation
CLASS SqlOrderRepository IMPLEMENTS OrderRepository
    CONSTRUCTOR(connectionPool)
        this.pool = connectionPool

    FUNCTION save(order)
        connection = this.pool.acquire()
        TRY
            connection.beginTransaction()
            connection.execute(
                "INSERT INTO orders (id, customer_id, status, total, ordered_at)
                 VALUES (?, ?, ?, ?, ?)
                 ON CONFLICT (id) DO UPDATE SET status = ?, total = ?",
                order.id, order.customerId, order.status, order.total, order.orderedAt,
                order.status, order.total
            )
            FOR EACH item IN order.items
                connection.execute(
                    "INSERT INTO order_items (order_id, product_id, quantity, price)
                     VALUES (?, ?, ?, ?)
                     ON CONFLICT (order_id, product_id) DO UPDATE SET quantity = ?, price = ?",
                    order.id, item.productId, item.quantity, item.price,
                    item.quantity, item.price
                )
            END FOR
            connection.commit()
        CATCH error
            connection.rollback()
            THROW error
        FINALLY
            this.pool.release(connection)
        END TRY

    FUNCTION findById(id)
        result = this.pool.query(
            "SELECT o.*, oi.product_id, oi.quantity, oi.price
             FROM orders o
             LEFT JOIN order_items oi ON o.id = oi.order_id
             WHERE o.id = ?", id
        )
        IF result IS EMPTY THEN RETURN NULL
        RETURN mapToOrder(result)
END CLASS

// Adapter — document store implementation
CLASS DocumentOrderRepository IMPLEMENTS OrderRepository
    CONSTRUCTOR(documentStore)
        this.collection = documentStore.collection("orders")

    FUNCTION save(order)
        this.collection.upsert(order.id, orderToDocument(order))

    FUNCTION findById(id)
        doc = this.collection.findOne(id)
        IF doc IS NULL THEN RETURN NULL
        RETURN documentToOrder(doc)
END CLASS
```

### Unit of Work Pattern

Coordinate multiple repository operations within a single transaction boundary:

```
INTERFACE UnitOfWork
    FUNCTION begin() -> Void
    FUNCTION commit() -> Void
    FUNCTION rollback() -> Void
    FUNCTION orderRepository() -> OrderRepository
    FUNCTION customerRepository() -> CustomerRepository

CLASS DatabaseUnitOfWork IMPLEMENTS UnitOfWork
    CONSTRUCTOR(connectionPool)
        this.connection = connectionPool.acquire()

    FUNCTION begin()
        this.connection.beginTransaction()

    FUNCTION commit()
        this.connection.commit()

    FUNCTION rollback()
        this.connection.rollback()

    FUNCTION orderRepository()
        RETURN NEW SqlOrderRepository(this.connection)

    FUNCTION customerRepository()
        RETURN NEW SqlCustomerRepository(this.connection)
END CLASS

// Usage in a use case
FUNCTION processOrder(command, unitOfWork)
    unitOfWork.begin()
    TRY
        customer = unitOfWork.customerRepository().findById(command.customerId)
        IF customer IS NULL THEN THROW CustomerNotFoundError

        order = Order.create(customer, command.items)
        unitOfWork.orderRepository().save(order)

        customer.incrementOrderCount()
        unitOfWork.customerRepository().save(customer)

        unitOfWork.commit()
    CATCH error
        unitOfWork.rollback()
        THROW error
    END TRY
END FUNCTION
```

### Data Transfer and Synchronization

When data must flow between services or stores:

```
// Change Data Capture (CDC) — stream changes from a database
PROCESS ChangeDataCapture
    SUBSCRIBE TO database.changelog("orders")

    ON EACH change
        event = NEW DataChangeEvent(
            table     = change.table,
            operation = change.type,   // INSERT, UPDATE, DELETE
            before    = change.oldRow,
            after     = change.newRow,
            timestamp = change.commitTimestamp
        )
        eventBus.publish("data-changes.orders", event)
    END ON
END PROCESS

// Materialized View — pre-computed read model
PROCESS MaterializedViewBuilder
    SUBSCRIBE TO eventBus.topic("data-changes.orders")

    ON EACH event
        SWITCH event.operation
            CASE "INSERT"
                readStore.insert(transformForRead(event.after))
            CASE "UPDATE"
                readStore.update(event.after.id, transformForRead(event.after))
            CASE "DELETE"
                readStore.delete(event.before.id)
        END SWITCH
    END ON
END PROCESS
```

## Project Structure

```
src/
│
├── domain/                             # Domain models (no infrastructure deps)
│   ├── entities/                       # Aggregate roots and entities
│   ├── value-objects/                  # Immutable value types
│   └── events/                         # Domain event definitions
│
├── application/                        # Use cases and orchestration
│   ├── commands/                       # Write operations
│   ├── queries/                        # Read operations
│   └── ports/                          # Repository interfaces (abstractions)
│
├── infrastructure/
│   ├── persistence/
│   │   ├── relational/                 # SQL-based repository implementations
│   │   │   ├── repositories/
│   │   │   ├── migrations/             # Schema migration scripts (versioned)
│   │   │   └── config/                 # Connection pool, ORM configuration
│   │   │
│   │   ├── document/                   # Document store implementations
│   │   │   ├── repositories/
│   │   │   └── config/
│   │   │
│   │   ├── cache/                      # Key-value / cache implementations
│   │   │   └── repositories/
│   │   │
│   │   └── search/                     # Search index implementations
│   │       ├── indexers/               # Index building and sync
│   │       └── queries/                # Search query builders
│   │
│   ├── data-sync/                      # CDC, ETL, materialized view builders
│   │   ├── change-capture/
│   │   └── projections/
│   │
│   └── schema-registry/                # Schema definitions and versioning
│       ├── schemas/
│       └── migrations/
│
└── tests/
    ├── unit/                           # Domain and application logic tests
    ├── integration/                    # Repository tests against real stores
    └── data-quality/                   # Data validation and consistency checks
```

## Benefits

1. **Right Tool for the Job** — Polyglot persistence matches each data access pattern to the most suitable storage engine, optimizing both performance and developer experience
2. **Scalability** — Partitioning and replication strategies allow data stores to scale independently based on actual workload demands
3. **Evolvability** — Schema evolution strategies and migration tooling allow data models to change without system-wide downtime
4. **Clear Ownership** — Data mesh principles ensure every dataset has a responsible owner, improving quality and discoverability
5. **Resilience** — Explicit consistency boundaries prevent a failure in one data store from cascading across the entire system
6. **Performance** — Separating read and write paths (and using materialized views) allows each path to be tuned independently

## Trade-offs

| Advantage                            | Consideration                                                    |
| ------------------------------------ | ---------------------------------------------------------------- |
| Polyglot persistence optimizes fit   | Operational complexity — more technologies to manage and monitor |
| Eventual consistency enables scaling | Harder to reason about data correctness across services          |
| Schema evolution supports agility    | Requires discipline — breaking changes can cascade downstream    |
| Data partitioning enables throughput | Cross-partition queries become expensive or impossible           |
| Data mesh distributes ownership      | Requires organizational maturity and governance investment       |
| Read/write separation boosts perf    | Data synchronization lag introduces staleness windows            |

## When to Use

✅ **Good fit for:**

- Systems with multiple distinct data access patterns (transactional, analytical, search)
- Microservices architectures where each service owns its data
- Applications that must scale read and write workloads independently
- Organizations large enough to benefit from domain-owned data products
- Systems where data schemas evolve frequently

❌ **Not ideal for:**

- Small applications where a single relational database covers all needs
- Prototypes or MVPs where operational simplicity matters most
- Teams without the capacity to manage multiple storage technologies
- Systems where all data access patterns are uniformly relational

## References

- [Designing Data-Intensive Applications — Martin Kleppmann (2017)](https://dataintensive.net/)
- [Data Mesh: Delivering Data-Driven Value at Scale — Zhamak Dehghani (2022)](https://www.oreilly.com/library/view/data-mesh/9781492092391/)
- [Azure Architecture Center — Understand Data Store Models](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview)
- [Azure Architecture Center — Data Considerations for Microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/data-considerations)
- [Azure Cloud Adoption Framework — What is a Data Mesh?](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/architectures/what-is-data-mesh)
- [CAP Theorem — Eric Brewer (2000)](https://en.wikipedia.org/wiki/CAP_theorem)
