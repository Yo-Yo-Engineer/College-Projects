# Database Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve database design, queries, and data access patterns for [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review schema design, query patterns, and data access code
- Apply all schema changes through migration scripts

## Focus Areas

### Schema Design and Normalization

- Verify appropriate normalization level for the use case — avoid both under- and over-normalization
- Check for appropriate data types, constraints, and NOT NULL where applicable
- Ensure primary keys, foreign keys, and unique constraints are defined
- Verify consistent naming conventions for tables, columns, indexes, and constraints
- Check for proper use of enums, lookup tables, or domain types for constrained values

### Indexing Strategy

- Verify indexes exist for columns used in WHERE, JOIN, ORDER BY, and GROUP BY clauses
- Check for missing composite indexes on multi-column query patterns
- Identify unused or redundant indexes that add write overhead without read benefit
- Verify covering indexes for high-frequency queries where appropriate
- Ensure index naming follows a consistent convention

### Query Performance

- Identify N+1 query patterns and replace with eager loading, joins, or batched queries
- Check for SELECT \* usage — ensure only required columns are fetched
- Verify pagination is used for large result sets — not unbounded queries
- Identify slow queries: missing indexes, full table scans, unnecessary subqueries, Cartesian products
- Check for proper use of query parameterization — no string concatenation for SQL construction
- Verify appropriate use of database-side filtering vs. application-side filtering

### Connection Management

- Verify connection pooling is configured with appropriate min/max pool sizes
- Check for leaked connections — ensure connections are properly returned to the pool
- Verify connection timeout and idle timeout settings
- Ensure connection strings do not contain hardcoded credentials
- Check for proper transaction management: short transactions, appropriate isolation levels

### Migrations and Schema Changes

- Verify all schema changes are managed through versioned migration scripts
- Ensure migrations are reversible (up and down) where practical
- Check that migrations are safe for zero-downtime deployments (no long-running locks)
- Verify data migrations handle large tables efficiently (batched updates, not full-table locks)
- Ensure migration naming follows a consistent, chronological convention

### Data Integrity and Validation

- Verify data validation at both application and database layers
- Check for proper use of transactions to maintain consistency across related operations
- Ensure referential integrity is enforced through foreign keys or application logic
- Verify soft-delete patterns are consistent and queries filter appropriately
- Check for orphaned records and cascading delete/update behavior

### Security and Access Control

- Verify database access uses least-privilege accounts — not root/admin for application access
- Ensure sensitive data (PII, credentials) is encrypted at rest where required
- Check that audit trails exist for sensitive data access and modifications
- Verify SQL injection prevention through parameterized queries throughout the codebase
- Ensure database credentials are managed through secret stores, not configuration files

### Backup and Recovery

- Verify backup strategy exists and is documented
- Check for point-in-time recovery capability where needed
- Ensure backup restoration has been tested or is testable

## Reference Standards

- Database normalization (1NF through 3NF/BCNF as appropriate)
- ACID properties for transactional integrity
- CAP theorem awareness for distributed databases
- The Twelve-Factor App (IV: Backing services)

## Constraints

- Apply all data changes through migrations — never modify production data directly
- Preserve backward compatibility with existing application code during schema changes
- Prefer incremental migration over destructive rebuild
- Use only ORM features and patterns already adopted in the project

## Output

1. Schema and query issues identified, with impact assessment
2. Changes made or proposed (migrations, index additions, query rewrites)
3. N+1 and performance issues resolved
4. Security and access control improvements
5. Recommendations for monitoring, maintenance, and scaling
6. Migration scripts or instructions for proposed schema changes
