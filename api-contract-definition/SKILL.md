---
name: api-contract-definition
description: "Define API contracts (OpenAPI/gRPC) before implementation. Use when designing service-to-service APIs or public endpoints. Ensures versioning, pagination, error handling, and schema consistency."
---

# API Contract Definition

The contract is the cheapest place to get the API right. Once consumers integrate, the shape is frozen — a renamed field or an unversioned breaking change becomes everyone else's migration. This skill forces the decisions that are nearly free now and irreversible later: **versioning, error taxonomy, pagination, and backwards-compatibility promise** — agreed and validated against real scenarios *before* a line of implementation.

## When to Trigger
Before coding any API endpoint:
- Public APIs (external consumers)
- Service-to-service APIs
- Webhook specifications
- Event schema definitions

## When NOT to Trigger
- A single internal function call with no network boundary
- A throwaway endpoint in a prototype you'll delete (note the contract is deferred)
- A purely additive, non-breaking change to an endpoint already under contract (just extend the existing spec)

## Scale to the build
- **Public API / external consumers** → full spec, explicit versioning + deprecation policy, 5–10 scenarios, consumer testing guide.
- **Internal service-to-service** → schema + error taxonomy + versioning; lighter on the deprecation ceremony if both sides ship together.
- **Prototype** → sketch the resource shapes and the error envelope; skip formal versioning until it has a real consumer.

## What It Produces
1. OpenAPI spec (or gRPC proto)
2. Versioning strategy
3. Error taxonomy
4. Example requests/responses (5–10 scenarios)
5. Consumer testing guide

## Workflow

### Step 1: Gather Intent
```
For this API, tell me:

1. Primary use case?  (CRUD / async job / real-time stream / file upload-download)
2. Expected throughput?  ([req/sec at launch] → [req/sec at scale])
3. Backwards compatibility required?  (Yes = public/external · No = internal, can break)
4. Consumer types?  (web / mobile / server-to-server / CLIs/scripts)
```

### Step 2: Generate the Spec
```yaml
openapi: 3.0.0
info:
  title: [API Name]
  version: 1.0.0
  description: |
    [What this API does]
    Versioning: Semantic versioning (MAJOR.MINOR.PATCH)
    Deprecation policy: 6-month grace period before breaking changes
paths:
  /v1/[resource]:
    post:
      summary: Create [resource]
      parameters: [...]
      requestBody:
        content:
          application/json:
            schema: [...]
      responses:
        201:
          description: Created
          content:
            application/json:
              schema: [...]
        400:
          description: Validation error
          content:
            application/json:
              schema:
                type: object
                properties:
                  error_code: { type: string }   # e.g., "INVALID_EMAIL"
                  message:    { type: string }   # e.g., "Email format invalid"
                  details:    { type: object }   # e.g., {"field": "email", "reason": "..."}
        429:
          description: Rate limited
          headers:
            X-RateLimit-Remaining: { schema: { type: integer } }

components:
  schemas:
    ErrorResponse:
      type: object
      properties:
        error_code: { type: string }   # Machine-readable
        message:    { type: string }   # Human-readable
        request_id: { type: string }   # For support tickets
        timestamp:  { type: string }   # ISO 8601
  securitySchemes:
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - apiKey: []
```

### Step 3: Validate Against Use Cases
Walk 5–10 concrete scenarios — happy path, every error in the taxonomy, and pagination:
```
Scenario 1 — Create new user
  Request:  POST /v1/users { "email": "new@example.com", "name": "..." }
  Response: 201 { "id": "usr_123", "email": "new@example.com", "created_at": "..." }

Scenario 2 — Email already exists
  Request:  POST /v1/users { "email": "existing@example.com" }
  Response: 400 { "error_code": "EMAIL_ALREADY_EXISTS", "message": "..." }

Scenario 3 — Rate limited
  Response: 429 { "error_code": "RATE_LIMITED" } + X-RateLimit-Remaining: 0
...
```

### Step 4: PM Approval
- [ ] Errors are well-defined (error_code + message + request_id)
- [ ] Pagination strategy is clear (if applicable)
- [ ] Versioning path is explicit
- [ ] Backwards-compatibility promise is documented

## Handoff (Next in the SDLC Chain)
Once the contract is approved, always run **security-baseline** next — before any implementation — to validate PII handling, secrets, auth/authz, compliance scope, and dependency CVEs against the contract you just defined.

> "API contract locked. Before we write a line of implementation, I'm running the Security Baseline to make sure these endpoints handle PII and auth correctly."

## Anti-Patterns
- ❌ Designing the contract after the code exists → ✅ contract first, code to the contract
- ❌ Unversioned endpoints → ✅ explicit version path + deprecation policy from day one
- ❌ Free-text error strings → ✅ machine-readable `error_code` + human `message` + `request_id`
- ❌ Pagination "we'll add when it's slow" → ✅ decided up front for any list endpoint
- ❌ Validating only the happy path → ✅ a scenario for every error in the taxonomy
- ❌ Silent breaking changes → ✅ a documented backwards-compatibility promise consumers can rely on
