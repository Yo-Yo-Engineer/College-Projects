# API Design Best Practices Review

#todo #agent [OPTIONAL_TAGS]

Task:
Review and improve the API design of [TARGET].

Reference materials:

- [RESOURCE_1]
- [RESOURCE_2]
- [RESOURCE_3]

Repository context:

- #file:[OPTIONAL_CONTEXT_FILE]
- #file:[TARGET_FILE_OR_FOLDER]

Scope:

- Focus on [TARGET_SCOPE]
- Review public-facing APIs and internal service interfaces
- Preserve existing client compatibility unless versioning is in place

## Focus Areas

### Resource Design and URL Structure

- Verify resources are modeled as nouns, not actions (e.g., /orders not /createOrder)
- Ensure consistent, hierarchical URL structure reflecting resource relationships
- Check for consistent pluralization, casing (kebab-case or snake_case), and naming conventions
- Avoid deeply nested URLs — prefer flat structures with query parameters for filtering

### HTTP Semantics

- Verify correct use of HTTP methods: GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE (remove)
- Ensure appropriate HTTP status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500
- Check that GET requests are safe and idempotent, PUT and DELETE are idempotent
- Verify proper use of HTTP headers for content negotiation, caching, and authentication

### Request and Response Design

- Ensure consistent response envelope or structure across all endpoints
- Verify error responses follow a standard format (RFC 7807 Problem Details or equivalent)
- Check that error responses include: status, error type/code, human-readable message, and correlation ID
- Ensure successful responses return only necessary data — avoid over-fetching
- Verify request validation with clear, actionable error messages for invalid input

### Pagination, Filtering, and Sorting

- Verify pagination is implemented for all list endpoints (offset/limit or cursor-based)
- Check for consistent filtering and sorting query parameter conventions
- Ensure pagination metadata is included in responses (total count, next/previous links)
- Verify that default page sizes are reasonable and maximum limits are enforced

### Versioning

- Verify an API versioning strategy exists (URL path, header, or query parameter)
- Ensure breaking changes are managed through versioning, not silent modifications
- Check for deprecated endpoints with appropriate documentation and sunset timelines

### Idempotency and Reliability

- Verify idempotency for all non-read operations where applicable (idempotency keys)
- Check for proper handling of duplicate requests and concurrent modifications
- Ensure retry-safe design — clients can safely retry failed requests
- Verify appropriate use of ETags or optimistic concurrency control for updates

### Rate Limiting and Throttling

- Verify rate limiting is applied to protect against abuse and ensure fair usage
- Check that rate limit headers are returned (X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset)
- Ensure 429 Too Many Requests is returned with a Retry-After header

### Documentation and Contracts

- Verify an OpenAPI/Swagger specification exists and is accurate
- Ensure API documentation includes request/response examples
- Check for contract-first design — specification drives implementation
- Verify API changelog or migration guides exist for version transitions

## Reference Standards

- REST architectural constraints (Fielding)
- RFC 7231 (HTTP Semantics), RFC 7807 (Problem Details)
- JSON:API, HAL, or equivalent hypermedia conventions (where adopted)
- OpenAPI Specification 3.x

## Constraints

- Maintain backward compatibility — introduce breaking changes only with a versioning strategy in place
- Preserve existing client contracts unless migration support is provided
- Prefer incremental improvements over wholesale API redesign
- Follow existing project API conventions unless they conflict with standards

## Output

1. API design issues identified, categorized by impact
2. Changes made or proposed with justification
3. Breaking vs. non-breaking change classification
4. Recommendations for API documentation, testing, or tooling
5. Migration considerations for any breaking changes
