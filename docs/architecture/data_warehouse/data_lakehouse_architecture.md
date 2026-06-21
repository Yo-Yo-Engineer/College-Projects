# Data Lakehouse Architecture

## Overview

**Data Lakehouse Architecture** is a data management paradigm that combines the low-cost, scalable storage of a **data lake** with the data management, ACID transaction, and performance features of a **data warehouse**. It eliminates the traditional two-tier architecture (lake + warehouse) by introducing a metadata and governance layer directly on top of open file formats stored in object storage.

Key principles:

- **Open File Formats** — Data is stored in open, columnar formats (Parquet, ORC) rather than proprietary warehouse formats, avoiding vendor lock-in
- **ACID Transactions** — Transaction logs enable atomic, consistent, isolated, and durable operations on data lake storage
- **Schema Enforcement & Evolution** — Schemas are enforced on write and can evolve without rewriting entire datasets
- **Unified Batch & Streaming** — A single architecture serves both batch analytics and real-time streaming workloads
- **Decoupled Storage & Compute** — Storage (object store) and compute (query engines) scale independently
- **Direct BI Access** — BI tools query data directly in the lakehouse without requiring a separate warehouse copy

### Evolution of Data Architectures

```
┌───────────────────────────────────────────────────────────────────────┐
│                  Data Architecture Evolution                         │
├──────────────────┬────────────────────────────────────────────────────┤
│  Data Warehouse  │  Structured data, SQL analytics, ACID             │
│  (1990s)         │  transactions. Expensive, proprietary, limited    │
│                  │  to structured data.                              │
├──────────────────┼────────────────────────────────────────────────────┤
│  Data Lake       │  All data types at low cost. Schema-on-read.      │
│  (2010s)         │  Became "data swamps" — no quality, governance,   │
│                  │  or ACID guarantees.                              │
├──────────────────┼────────────────────────────────────────────────────┤
│  Data Lakehouse  │  Best of both. ACID transactions + open formats   │
│  (2020s)         │  on object storage. Unified governance, direct    │
│                  │  BI access, ML integration.                       │
└──────────────────┴────────────────────────────────────────────────────┘
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         Data Lakehouse Architecture                              │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                       Consumption Layer                                     │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │
│  │  │   BI &    │  │  SQL      │  │  ML / Data   │  │  Real-time           │   │ │
│  │  │  Reports  │  │  Analysts │  │  Science     │  │  Applications        │   │ │
│  │  └──────────┘  └──────────┘  └──────────────┘  └──────────────────────┘   │ │
│  └──────────────────────────────────┬──────────────────────────────────────────┘ │
│                                     │                                            │
│  ┌──────────────────────────────────▼──────────────────────────────────────────┐ │
│  │                     Query / Compute Engines                                 │ │
│  │         (Spark, Trino, Presto, Flink, DuckDB, Snowflake, etc.)             │ │
│  └──────────────────────────────────┬──────────────────────────────────────────┘ │
│                                     │                                            │
│  ┌──────────────────────────────────▼──────────────────────────────────────────┐ │
│  │                   Table Format / Metadata Layer                             │ │
│  │                (Delta Lake, Apache Iceberg, Apache Hudi)                    │ │
│  │                                                                             │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐  │ │
│  │  │  Transaction │  │  Schema       │  │  Partition  │  │  Time Travel     │  │ │
│  │  │  Log (ACID)  │  │  Management   │  │  Pruning    │  │  & Versioning    │  │ │
│  │  └─────────────┘  └──────────────┘  └────────────┘  └──────────────────┘  │ │
│  └──────────────────────────────────┬──────────────────────────────────────────┘ │
│                                     │                                            │
│  ┌──────────────────────────────────▼──────────────────────────────────────────┐ │
│  │                   Governance & Catalog Layer                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │  Data Catalog │  │  Access       │  │  Data Quality │  │  Lineage     │   │ │
│  │  │  & Discovery  │  │  Control      │  │  Rules        │  │  Tracking    │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └──────────────────────────────────┬──────────────────────────────────────────┘ │
│                                     │                                            │
│  ┌──────────────────────────────────▼──────────────────────────────────────────┐ │
│  │                      Storage Layer (Object Storage)                         │ │
│  │          (S3, ADLS, GCS, MinIO — open columnar file formats)               │ │
│  │                                                                             │ │
│  │  ┌───────────────────────────────────────────────────────────────────────┐  │ │
│  │  │  Bronze (Raw)          │  Silver (Cleansed)    │  Gold (Curated)      │  │ │
│  │  │  Ingested as-is        │  Validated, deduped   │  Business-ready      │  │ │
│  │  │  Full fidelity         │  Conformed schemas    │  Aggregated, modeled │  │ │
│  │  └───────────────────────────────────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### 1. Table Formats (The Metadata Layer)

Table formats provide the ACID transaction and metadata layer on top of object storage:

```
// Table format managing ACID transactions on object storage
CLASS LakehouseTable
    CONSTRUCTOR(
        tablePath       : String,
        objectStorage   : ObjectStorage,
        transactionLog  : TransactionLog
    )

    FUNCTION write(data : DataFrame, mode : WriteMode) -> CommitInfo
        // Acquire write lock via optimistic concurrency
        currentVersion = transactionLog.latestVersion()

        // Validate schema compatibility
        IF mode == WriteMode.APPEND THEN
            validateSchemaCompatibility(data.schema, currentSchema())
        END IF

        // Write data files in open columnar format (Parquet)
        newFiles = objectStorage.writeParquet(
            data      = data,
            path      = tablePath + "/data/",
            partition = partitionSpec
        )

        // Commit atomically to transaction log
        commit = transactionLog.commit(
            version     = currentVersion + 1,
            addedFiles  = newFiles,
            removedFiles = IF mode == WriteMode.OVERWRITE THEN currentFiles() ELSE EMPTY,
            schema      = data.schema,
            timestamp   = NOW()
        )

        RETURN commit

    FUNCTION readAtVersion(version : Integer) -> DataFrame
        // Time travel — read data as it existed at a specific version
        snapshot = transactionLog.getSnapshot(version)
        files = snapshot.activeFiles()
        RETURN objectStorage.readParquet(files)

    FUNCTION evolveSchema(newSchema : Schema) -> CommitInfo
        currentSchema = currentSchema()
        evolution = computeSchemaEvolution(currentSchema, newSchema)

        IF evolution.hasBreakingChanges AND NOT allowBreakingChanges THEN
            RAISE SchemaEvolutionError(evolution.breakingChanges)
        END IF

        RETURN transactionLog.commit(
            version    = transactionLog.latestVersion() + 1,
            schemaUpdate = newSchema,
            timestamp  = NOW()
        )
END CLASS
```

### 2. Medallion Architecture (Bronze / Silver / Gold)

Organize data into layers of increasing quality and business value:

```
// Medallion data quality pipeline
CLASS MedallionPipeline
    CONSTRUCTOR(
        lakehouse     : LakehouseTable,
        qualityRules  : DataQualityRules,
        catalog       : DataCatalog
    )

    FUNCTION ingestBronze(source : DataSource) -> BronzeResult
        // Ingest raw data as-is — full fidelity, no transformations
        rawData = source.extract()

        lakehouse.write(
            data  = rawData,
            table = "bronze." + source.name,
            mode  = WriteMode.APPEND,
            metadata = {
                source       = source.identifier,
                ingestedAt   = NOW(),
                format       = source.format,
                recordCount  = rawData.count()
            }
        )

        RETURN NEW BronzeResult(table = "bronze." + source.name, records = rawData.count())

    FUNCTION refineSilver(bronzeTable : String) -> SilverResult
        // Cleanse, validate, deduplicate, conform schemas
        rawData = lakehouse.read("bronze." + bronzeTable)

        cleaned = rawData
            .dropDuplicates(keys = deduplicationKeys)
            .filter(qualityRules.notNull(requiredColumns))
            .withColumn("_processed_at", NOW())

        // Apply data quality checks
        qualityReport = qualityRules.validate(cleaned)
        quarantined = cleaned.filter(qualityReport.failedRowIds)
        passed = cleaned.filter(NOT qualityReport.failedRowIds)

        lakehouse.write(data = passed, table = "silver." + bronzeTable, mode = WriteMode.MERGE)
        lakehouse.write(data = quarantined, table = "quarantine." + bronzeTable, mode = WriteMode.APPEND)

        RETURN NEW SilverResult(passed = passed.count(), quarantined = quarantined.count())

    FUNCTION curateGold(silverTables : List<String>, businessModel : TransformSpec) -> GoldResult
        // Aggregate, join, model for business consumption
        datasets = MAP(silverTables, table -> lakehouse.read("silver." + table))

        goldData = businessModel.transform(datasets)

        lakehouse.write(
            data  = goldData,
            table = "gold." + businessModel.targetTable,
            mode  = WriteMode.OVERWRITE
        )

        // Register in catalog for discovery
        catalog.register(
            table       = "gold." + businessModel.targetTable,
            description = businessModel.description,
            owner       = businessModel.owner,
            tags        = businessModel.tags,
            lineage     = silverTables
        )

        RETURN NEW GoldResult(table = "gold." + businessModel.targetTable, records = goldData.count())
END CLASS
```

### 3. Governance & Access Control

Enforce fine-grained access, data quality, and lineage tracking:

```
// Unified governance layer
CLASS LakehouseGovernance
    CONSTRUCTOR(
        catalog        : DataCatalog,
        accessControl  : AccessControlManager,
        lineageTracker : LineageTracker
    )

    FUNCTION grantAccess(principal : Principal, table : String, permissions : List<Permission>) -> Void
        // Fine-grained access: table, column, and row-level security
        accessControl.grant(
            principal   = principal,
            resource    = table,
            permissions = permissions,
            columnMask  = IF hasSensitiveColumns(table) THEN maskingPolicy(table) ELSE NONE,
            rowFilter   = IF isMultiTenant(table) THEN tenantFilter(principal) ELSE NONE
        )

    FUNCTION trackLineage(sourceTable : String, targetTable : String, transformation : String) -> Void
        lineageTracker.record(
            source         = sourceTable,
            target         = targetTable,
            transformation = transformation,
            executedBy     = currentPrincipal(),
            executedAt     = NOW()
        )

    FUNCTION discoverData(query : String) -> List<CatalogEntry>
        RETURN catalog.search(
            query       = query,
            includeSchema    = TRUE,
            includeLineage   = TRUE,
            includeSamples   = TRUE,
            filterByAccess   = currentPrincipal()
        )
END CLASS
```

## Project Structure

```
lakehouse/
├── ingestion/                      # Data ingestion pipelines
│   ├── batch/
│   ├── streaming/
│   └── connectors/
│
├── pipelines/                      # Medallion transformation pipelines
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── governance/                     # Governance configuration
│   ├── access_policies/
│   ├── quality_rules/
│   ├── masking_policies/
│   └── catalog_config/
│
├── schemas/                        # Schema definitions & evolution
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── infrastructure/                 # IaC for storage & compute
│   ├── storage/
│   ├── compute/
│   └── networking/
│
└── tests/
    ├── data_quality/
    ├── pipeline/
    └── integration/
```

## Benefits

1. **Unified Platform** — Single architecture for BI, SQL analytics, data science, and streaming workloads
2. **Cost Efficiency** — Decoupled storage (cheap object stores) and compute (scale independently)
3. **Open Formats** — Avoids vendor lock-in; Parquet/ORC files readable by any engine
4. **ACID Transactions** — Reliable data operations with rollback, time travel, and concurrent writes
5. **Data Governance** — Centralized catalog, lineage, access control, and quality enforcement
6. **Schema Evolution** — Schemas evolve safely without full data rewrites

## Trade-offs

| Advantage                      | Consideration                                                         |
| ------------------------------ | --------------------------------------------------------------------- |
| Unified analytics platform     | Query performance may lag behind optimized warehouses                 |
| Open format, no vendor lock-in | Table format ecosystem is still maturing (Delta vs. Iceberg vs. Hudi) |
| Low-cost scalable storage      | Requires careful data organization (medallion layers)                 |
| ACID on object storage         | Compaction and maintenance operations are necessary                   |
| Supports all workload types    | Operational complexity higher than managed warehouses                 |

## When to Use

✅ **Good fit for:**

- Organizations consolidating separate data lake and warehouse systems
- Multi-workload environments (BI + ML + streaming on the same data)
- Teams requiring data versioning, time travel, and audit trails
- Cost-sensitive environments with large data volumes
- Organizations wanting open-format, vendor-neutral data platforms

❌ **Not ideal for:**

- Small datasets where a managed data warehouse is simpler and sufficient
- Pure OLTP workloads requiring sub-millisecond transactional performance
- Teams without data engineering capacity to manage medallion pipelines
- Organizations already well-served by a fully managed warehouse (Snowflake, BigQuery) with no lock-in concerns

## References

- [Lakehouse: A New Generation of Open Platforms — Armbrust et al., CIDR 2021](http://cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)
- [Delta Lake: High-Performance ACID Table Storage — Databricks](https://delta.io/)
- [Apache Iceberg — Table Format for Huge Analytic Datasets](https://iceberg.apache.org/)
- [Apache Hudi — Hadoop Upserts, Deletes, and Incrementals](https://hudi.apache.org/)
- [Medallion Architecture — Databricks](https://www.databricks.com/glossary/medallion-architecture)
- [What is a Data Lakehouse? — Databricks](https://www.databricks.com/glossary/data-lakehouse)
