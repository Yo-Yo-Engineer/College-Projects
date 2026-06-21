# Cell-Based Architecture

## Overview

**Cell-Based Architecture** is a structural pattern for building resilient, scalable distributed systems by decomposing an application into isolated, self-contained units called **cells**. Each cell is a complete, independently deployable instance of a set of services with its own compute, storage, and routing — designed to limit the **blast radius** of failures so that a problem in one cell cannot cascade across the entire system.

Key principles:

- **Blast Radius Containment** — Failures, misconfigurations, or bad deployments are isolated to a single cell, protecting the rest of the system
- **Cell Independence** — Each cell operates autonomously with its own data store, compute, and dependencies; no shared mutable state between cells
- **Deterministic Routing** — A stable routing layer assigns each request (or tenant/user) to a specific cell using a partitioning key
- **Homogeneous Cells** — All cells run the same software and configuration, differing only in the data/traffic they serve
- **Independent Deployability** — Cells can be deployed, upgraded, and scaled independently, enabling safe progressive rollouts
- **Bounded Scale** — Each cell has a known, tested capacity ceiling; scaling is achieved by adding more cells rather than growing existing ones

### Cell vs. Microservice

```
┌───────────────────────────────────────────────────────────────────────────┐
│                      Cell vs. Microservice                               │
├──────────────────┬────────────────────────────────────────────────────────┤
│  Microservice    │  A single service decomposed by business capability.   │
│                  │  Services call each other across the mesh.            │
│                  │  Failure can cascade through service dependencies.    │
├──────────────────┼────────────────────────────────────────────────────────┤
│  Cell            │  A self-contained unit containing multiple services.  │
│                  │  Each cell is a full stack of services + data.        │
│                  │  Failure is contained within the cell boundary.       │
└──────────────────┴────────────────────────────────────────────────────────┘

Microservices answer: "How do we decompose functionality?"
Cells answer: "How do we isolate failure and scale safely?"
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         Cell-Based Architecture                                  │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Cell Router                                        │ │
│  │      (Routes requests to cells based on partition key — e.g. tenant ID)    │ │
│  │      ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │      │   Partition Map:  tenant-A → Cell 1  |  tenant-B → Cell 2  |... │  │ │
│  │      └──────────────────────────────────────────────────────────────────┘  │ │
│  └───────────┬──────────────────────┬──────────────────────┬──────────────────┘ │
│              │                      │                      │                    │
│              ▼                      ▼                      ▼                    │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐           │
│  │      Cell 1        │  │      Cell 2        │  │      Cell 3        │           │
│  │  ┌──────────────┐ │  │  ┌──────────────┐ │  │  ┌──────────────┐ │           │
│  │  │  API Gateway  │ │  │  │  API Gateway  │ │  │  │  API Gateway  │ │           │
│  │  └──────┬───────┘ │  │  └──────┬───────┘ │  │  └──────┬───────┘ │           │
│  │         │          │  │         │          │  │         │          │           │
│  │  ┌──────▼───────┐ │  │  ┌──────▼───────┐ │  │  ┌──────▼───────┐ │           │
│  │  │  Service A    │ │  │  │  Service A    │ │  │  │  Service A    │ │           │
│  │  │  Service B    │ │  │  │  Service B    │ │  │  │  Service B    │ │           │
│  │  │  Service C    │ │  │  │  Service C    │ │  │  │  Service C    │ │           │
│  │  └──────┬───────┘ │  │  └──────┬───────┘ │  │  └──────┬───────┘ │           │
│  │         │          │  │         │          │  │         │          │           │
│  │  ┌──────▼───────┐ │  │  ┌──────▼───────┐ │  │  ┌──────▼───────┐ │           │
│  │  │  Cell DB      │ │  │  │  Cell DB      │ │  │  │  Cell DB      │ │           │
│  │  │  Cell Cache   │ │  │  │  Cell Cache   │ │  │  │  Cell Cache   │ │           │
│  │  │  Cell Queue   │ │  │  │  Cell Queue   │ │  │  │  Cell Queue   │ │           │
│  │  └──────────────┘ │  │  └──────────────┘ │  │  └──────────────┘ │           │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘           │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                    Control Plane (Global)                                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │  Cell         │  │  Partition    │  │  Deployment   │  │  Monitoring   │   │ │
│  │  │  Provisioning │  │  Management   │  │  Orchestrator │  │  & Alerting   │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### 1. Cell Router

The cell router deterministically maps each request to a cell based on a partition key:

```
// Cell Router — deterministic routing with partition map
CLASS CellRouter
    CONSTRUCTOR(
        partitionMap    : PartitionMap,
        healthChecker   : CellHealthChecker,
        overflowPolicy  : OverflowPolicy
    )

    FUNCTION route(request : IncomingRequest) -> CellEndpoint
        // Extract partition key from request (e.g. tenant ID, user ID)
        partitionKey = extractPartitionKey(request)

        // Look up assigned cell
        cell = partitionMap.getCellForKey(partitionKey)

        IF cell IS NULL THEN
            RAISE UnknownPartitionError(partitionKey)
        END IF

        // Check cell health before routing
        IF NOT healthChecker.isHealthy(cell) THEN
            IF overflowPolicy == FAILOVER THEN
                cell = partitionMap.getFailoverCell(partitionKey)
            ELSE
                RAISE CellUnavailableError(cell.id)
            END IF
        END IF

        RETURN cell.endpoint

    FUNCTION rebalance(reason : RebalanceReason) -> RebalanceResult
        // Move partitions between cells for capacity or failure reasons
        currentAssignments = partitionMap.allAssignments()
        cellCapacities = healthChecker.capacities()

        newAssignments = rebalanceAlgorithm.compute(
            current    = currentAssignments,
            capacities = cellCapacities,
            reason     = reason
        )

        // Apply partition moves atomically
        partitionMap.updateAssignments(newAssignments)

        RETURN NEW RebalanceResult(
            moved    = newAssignments.movedPartitions,
            fromCell = newAssignments.sourceCells,
            toCell   = newAssignments.targetCells
        )
END CLASS

// Partition Map — maps partition keys to cells
CLASS PartitionMap
    CONSTRUCTOR(store : ConsistentStore)

    FUNCTION getCellForKey(key : String) -> Cell
        partition = hashToPartition(key)
        RETURN store.get("partition:" + partition)

    FUNCTION assignPartition(partition : Partition, cell : Cell) -> Void
        store.put("partition:" + partition.id, cell)
        store.put("cell:" + cell.id + ":partitions",
                  store.get("cell:" + cell.id + ":partitions").add(partition))
END CLASS
```

### 2. Cell Lifecycle Management

Provision, deploy, and decommission cells as self-contained units:

```
// Cell lifecycle — provision, deploy, scale, decommission
CLASS CellLifecycleManager
    CONSTRUCTOR(
        infraProvisioner : InfrastructureProvisioner,
        deployManager    : DeploymentManager,
        partitionMap     : PartitionMap,
        cellRegistry     : CellRegistry
    )

    FUNCTION provisionCell(cellSpec : CellSpec) -> Cell
        // Provision isolated infrastructure for a new cell
        infra = infraProvisioner.provision(
            cellId       = cellSpec.id,
            region       = cellSpec.region,
            resources    = cellSpec.resourceSpec,
            isolation    = cellSpec.isolationLevel    // account, VPC, namespace
        )

        // Deploy the standard cell stack
        deployment = deployManager.deployStack(
            infrastructure = infra,
            services       = cellSpec.serviceManifests,
            config         = cellSpec.configuration
        )

        // Register cell as available
        cell = NEW Cell(
            id          = cellSpec.id,
            endpoint    = deployment.endpoint,
            capacity    = cellSpec.maxPartitions,
            region      = cellSpec.region,
            status      = ACTIVE
        )

        cellRegistry.register(cell)
        RETURN cell

    FUNCTION decommissionCell(cellId : String) -> DecommissionResult
        cell = cellRegistry.get(cellId)

        // Drain partitions to other cells first
        partitions = partitionMap.getPartitionsForCell(cellId)
        targetCells = cellRegistry.getHealthyCells(excludeId = cellId)

        FOR EACH partition IN partitions
            target = selectTarget(targetCells, partition)
            migratePartition(partition, fromCell = cell, toCell = target)
        END FOR

        // Tear down infrastructure after all partitions drained
        cell.status = DRAINING
        WAIT_UNTIL(partitionMap.getPartitionsForCell(cellId).isEmpty())

        infraProvisioner.teardown(cellId)
        cellRegistry.deregister(cellId)

        RETURN DecommissionResult.SUCCESS(migratedPartitions = partitions.count())
END CLASS
```

### 3. Progressive Cell Deployment

Deploy changes cell-by-cell to limit blast radius:

```
// Progressive deployment across cells
CLASS CellDeploymentOrchestrator
    CONSTRUCTOR(
        cellRegistry    : CellRegistry,
        deployManager   : DeploymentManager,
        monitor         : CellMonitor,
        rollbackManager : RollbackManager
    )

    FUNCTION progressiveRollout(release : Release) -> RolloutResult
        cells = cellRegistry.getAllCells()

        // Stage 1: Deploy to canary cell (smallest / lowest-risk)
        canaryCell = selectCanaryCell(cells)
        deployToCell(release, canaryCell)

        IF NOT monitorCanary(canaryCell, release, observationPeriod = Duration.minutes(30)) THEN
            rollbackManager.rollback(release, canaryCell)
            RETURN RolloutResult.CANARY_FAILED(canaryCell)
        END IF

        // Stage 2: Deploy to a wave of cells (e.g. 10%)
        wave1 = cells.sample(percentage = 10, exclude = [canaryCell])
        FOR EACH cell IN wave1
            deployToCell(release, cell)
        END FOR

        IF NOT monitorWave(wave1, release, observationPeriod = Duration.minutes(15)) THEN
            rollbackManager.rollbackAll(release, wave1 + [canaryCell])
            RETURN RolloutResult.WAVE_FAILED(wave1)
        END IF

        // Stage 3: Deploy to remaining cells
        remaining = cells.exclude(wave1 + [canaryCell])
        FOR EACH cell IN remaining
            deployToCell(release, cell)
        END FOR

        RETURN RolloutResult.COMPLETE(release, totalCells = cells.count())

    FUNCTION monitorCanary(cell : Cell, release : Release, observationPeriod : Duration) -> Boolean
        endTime = NOW() + observationPeriod

        WHILE NOW() < endTime
            metrics = monitor.getMetrics(cell)

            IF metrics.errorRate > release.maxErrorRate THEN RETURN FALSE
            IF metrics.latencyP99 > release.maxLatencyP99 THEN RETURN FALSE
            IF metrics.availabilityPercent < release.minAvailability THEN RETURN FALSE

            WAIT(Duration.seconds(30))
        END WHILE

        RETURN TRUE
END CLASS
```

### 4. Cross-Cell Operations

Handle operations that span multiple cells (rare by design):

```
// Cross-cell coordination for global operations
CLASS CrossCellCoordinator
    CONSTRUCTOR(
        cellRegistry   : CellRegistry,
        eventBus       : GlobalEventBus
    )

    FUNCTION executeGlobalQuery(query : GlobalQuery) -> AggregatedResult
        // Fan-out query to all cells, aggregate results
        cells = cellRegistry.getAllHealthyCells()
        futures = EMPTY LIST

        FOR EACH cell IN cells
            future = ASYNC cell.executeQuery(query)
            futures.ADD(future)
        END FOR

        results = AWAIT_ALL(futures, timeout = query.timeout)

        // Aggregate results from all cells
        aggregated = query.aggregator.aggregate(results)

        RETURN aggregated

    FUNCTION migratePartition(partition : Partition, fromCell : Cell, toCell : Cell) -> MigrationResult
        // Phase 1: Copy data to target cell
        toCell.importData(partition, fromCell.exportData(partition))

        // Phase 2: Enable dual-write (both cells receive writes)
        partitionMap.enableDualWrite(partition, fromCell, toCell)

        // Phase 3: Verify consistency
        IF NOT verifyConsistency(partition, fromCell, toCell) THEN
            partitionMap.disableDualWrite(partition)
            RETURN MigrationResult.CONSISTENCY_FAILURE
        END IF

        // Phase 4: Switch routing to target cell
        partitionMap.updateAssignment(partition, toCell)

        // Phase 5: Clean up source cell
        partitionMap.disableDualWrite(partition)
        fromCell.deletePartitionData(partition)

        RETURN MigrationResult.SUCCESS
END CLASS
```

## Project Structure

```
cell-platform/
├── cell-router/                    # Global routing layer
│   ├── router/
│   ├── partition-map/
│   └── health-checker/
│
├── cell-template/                  # Standard cell stack (deployed per cell)
│   ├── services/
│   │   ├── service-a/
│   │   ├── service-b/
│   │   └── service-c/
│   ├── data/
│   │   ├── database/
│   │   ├── cache/
│   │   └── queue/
│   └── config/
│
├── control-plane/                  # Global management plane
│   ├── cell-provisioner/
│   ├── deployment-orchestrator/
│   ├── partition-manager/
│   └── monitoring/
│
├── infrastructure/                 # IaC for cell provisioning
│   ├── cell-template/
│   ├── control-plane/
│   └── networking/
│
└── tests/
    ├── cell-isolation/
    ├── routing/
    ├── failover/
    └── migration/
```

## Benefits

1. **Blast Radius Containment** — Failures affect a single cell, not the entire system
2. **Safe Deployments** — Progressive cell-by-cell rollouts limit the impact of bad releases
3. **Predictable Scaling** — Each cell has a known capacity; scale by adding cells, not growing them
4. **Independent Operations** — Cells can be maintained, upgraded, and debugged independently
5. **Multi-Region Resilience** — Cells can be placed in different regions for geographic redundancy
6. **Regulatory Compliance** — Data residency requirements can be satisfied by cell placement

## Trade-offs

| Advantage                     | Consideration                                         |
| ----------------------------- | ----------------------------------------------------- |
| Failure isolation             | Cross-cell operations are complex and expensive       |
| Safe progressive deployments  | More infrastructure overhead (duplicated per cell)    |
| Predictable per-cell capacity | Partition rebalancing requires careful orchestration  |
| Independent cell operations   | Global queries must fan out to all cells              |
| Data residency compliance     | Cell provisioning and lifecycle management is complex |
| Simple scaling model          | Not cost-effective at small scale (minimum 2+ cells)  |

## When to Use

✅ **Good fit for:**

- Large-scale SaaS platforms requiring tenant isolation and blast radius control
- Systems where availability is critical and failure isolation is a top priority
- Multi-region deployments with data residency or sovereignty requirements
- Platforms that have experienced cascading failures from shared infrastructure
- Organizations with mature platform engineering teams

❌ **Not ideal for:**

- Small-scale applications where a single deployment unit is sufficient
- Systems with extensive cross-tenant or cross-partition queries
- Early-stage products where simplicity and velocity are more important than isolation
- Teams without the operational maturity to manage cell lifecycle and partition routing
- Applications with highly interconnected data that cannot be cleanly partitioned

## References

- [Cell-Based Architecture — WSO2](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md)
- [Shuffle Sharding — AWS Builders' Library](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/)
- [Cell-Based Architectures — AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/what-is-a-cell-based-architecture.html)
- [Scaling Up the Prime Video Audio/Video Monitoring Service — Amazon](https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90)
- [Designing Distributed Systems — Brendan Burns (O'Reilly)](https://www.oreilly.com/library/view/designing-distributed-systems/9781491983638/)
