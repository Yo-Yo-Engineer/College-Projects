# Performance Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Perform a performance-focused review of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Prioritize measurable improvements over micro-optimizations
- Preserve readability and correctness — accept marginal performance trade-offs when necessary

## Focus Areas

### Profiling and Measurement

- Identify hot paths and bottlenecks — optimize based on evidence, not assumptions
- Check for missing or inadequate performance baselines and benchmarks
- Verify that performance-critical paths have appropriate instrumentation

### Algorithmic Efficiency

- Review time and space complexity of critical operations
- Identify unnecessary O(n²) or worse patterns where linear or O(n log n) alternatives exist
- Check for redundant computation, repeated traversals, or unnecessary sorting

### Data Access and I/O

- Identify N+1 query patterns and replace with batched or joined queries
- Ensure database queries use appropriate indexes and avoid full table scans
- Check for missing connection pooling, excessive connection churn, or leaked connections
- Verify efficient use of file I/O, network calls, and external service interactions

### Caching

- Identify repeated expensive computations or data fetches that should be cached
- Verify cache invalidation strategy — stale data, TTL, cache stampede prevention
- Ensure caching does not introduce correctness issues or excessive memory use
- Check for appropriate cache levels (in-memory, distributed, HTTP/CDN)

### Concurrency and Async Patterns

- Verify appropriate use of async/await, parallelism, and concurrency primitives
- Identify blocking calls on hot paths that should be non-blocking
- Check for thread-safety issues, race conditions, and deadlock potential
- Ensure proper use of connection pools, thread pools, and worker patterns

### Memory and Resource Management

- Identify memory leaks, unbounded growth, and unnecessary allocations
- Check for proper resource cleanup (connections, file handles, streams, timers)
- Verify efficient data structures for the access patterns used
- Look for excessive object creation in loops or hot paths

### Payload and Network Efficiency

- Check for oversized payloads, missing compression, and unnecessary data transfer
- Verify pagination is used for large collections
- Ensure lazy loading and on-demand fetching where appropriate
- Check for missing or misconfigured HTTP caching headers

### Frontend Performance (if applicable)

- Identify render-blocking resources, excessive DOM operations, and layout thrashing
- Check for bundle size, code splitting, and tree shaking opportunities
- Verify image optimization, lazy loading, and responsive asset delivery

## Principles

- Measure first, optimize second — avoid premature optimization
- Prefer algorithmic improvements over low-level tricks
- Keep optimizations maintainable and well-documented
- Ensure optimizations do not introduce correctness or security regressions

## Constraints

- Preserve existing functionality and correctness
- Justify all complexity additions with measurable performance gains
- Prefer standard library and framework solutions over custom implementations
- Document any trade-offs between performance and readability

## Output

1. Bottlenecks and performance issues identified, with estimated impact
2. Changes made or proposed with justification
3. Before/after comparisons where measurable
4. Trade-offs between performance, complexity, and maintainability
5. Recommendations for profiling, benchmarking, or load testing follow-up
