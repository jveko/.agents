---
name: api-design
description: Use when designing or reviewing REST API endpoints, response formats, pagination, filtering, versioning, or error handling for HTTP APIs
---

# API Design

Design HTTP APIs around stable resources, semantic HTTP behavior, and predictable response formats.

**Core principle:** Prefer consistent resource modeling, explicit tradeoffs, and boring semantics over custom endpoint behavior.

## When to Use

- Designing new REST API endpoints
- Reviewing or revising an existing API contract
- Choosing response formats, pagination, filtering, or sorting behavior
- Defining status code and error response conventions
- Deciding whether a change requires API versioning

## When NOT to Use

- Non-HTTP protocols like gRPC, WebSockets, or queue consumers
- Pure implementation/debugging work where the API contract is already settled
- Project-specific backend rules that belong in repository docs instead of a general skill

## Decision Framework

### Resource naming

- Model URLs around nouns, not verbs
- Use plural resource names for collections
- Use nested resources only when the parent-child relationship is central to the API
- Use action endpoints sparingly for operations that do not fit CRUD cleanly

### Status codes

- Return status codes that match the outcome instead of encoding everything into a 200 response body
- Distinguish malformed input, validation failures, authorization failures, missing resources, and true server errors
- Keep success responses predictable: 200/201/204 should each mean something distinct

### Response shape

- Prefer one consistent response envelope across the API surface
- Use a `data` wrapper when consistency and metadata matter more than minimal payload shape
- Keep error responses structurally consistent so clients can handle them uniformly

### Pagination

- Use offset pagination for small datasets, admin screens, and page-number-driven UIs
- Use cursor pagination for large datasets, feeds, and high-write environments
- Do not ship unbounded list endpoints

### Versioning

- Avoid version churn for additive, non-breaking changes
- Introduce a new version only for breaking contract changes
- Use one versioning strategy consistently across the API

## Quick Reference

| Question | Default guidance |
|---|---|
| How should I name a resource? | Plural noun, lowercase, no verb in the path |
| How should I model errors? | Consistent error body + semantic HTTP status code |
| Should a list endpoint paginate? | Yes, unless the dataset is truly bounded and small |
| Offset or cursor pagination? | Offset for page-based UIs, cursor for scale and stability |
| When do I version? | Only when a change breaks existing clients |

## Common Mistakes

- Using verb-heavy URLs like `/getUsers` instead of resource-oriented paths
- Returning `200 OK` for validation errors, missing resources, or authorization failures
- Using a different error shape on every endpoint
- Leaving list endpoints unpaginated
- Introducing versioning too early or inconsistently

## Reference

For detailed examples, status code tables, pagination patterns, and longer REST API examples, see `rest-api-reference.md`.
