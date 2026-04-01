# Copilot Instructions — {{SERVICE_NAME}}

<!--
  TEMPLATE — Clean Architecture + DDD + Kotlin + Spring Boot
  =============================================================
  Fill in all fields marked with {{...}} before using.
  Remove this comment block after filling in.

  CHECKLIST:
  [ ] {{SERVICE_NAME}}               → service name (e.g. expense-service)
  [ ] {{PLATFORM_NAME}}              → platform name (e.g. FinanceFlow)
  [ ] {{SERVICE_RESPONSIBILITY}}     → responsibility in one line
  [ ] {{OUT_OF_SCOPE}}               → what this service does NOT do
  [ ] {{OTHER_SERVICES}}             → table of other services in the ecosystem
  [ ] {{ERROR_PREFIX}}               → 2-letter error code prefix (e.g. EX, WA, AU)
  [ ] {{VERSION_SERIES}}             → version series (e.g. 1.x.y)
  [ ] Review the Stack section if any technology differs
-->

---

## Overview

**{{SERVICE_NAME}}** is a microservice of the **{{PLATFORM_NAME}}** platform
responsible for: {{SERVICE_RESPONSIBILITY}}.

**Out of scope for this service:** {{OUT_OF_SCOPE}}.

### Service ecosystem

{{OTHER_SERVICES}}
<!--
  Example:
  | Service          | Responsibility               | Status              |
  |------------------|------------------------------|---------------------|
  | expense-service  | Expense tracking             | v1.0 — in progress  |
  | income-service   | Income tracking              | Planned             |
  | report-service   | Consolidated financial view  | Planned             |
-->

- Each service owns its **own database** — no shared storage
- Inter-service communication: **HTTP/REST** (synchronous)
- Each service manages its **own authentication** in v1.0

---

## Stack

- **Language:** Kotlin — never `Any`, never `!!`
- **Framework:** Spring Boot
- **Build:** Gradle (Kotlin DSL)
- **Database:** PostgreSQL — schema changes via **Flyway** only
- **ORM:** JPA / Hibernate — `FetchType.LAZY` required, `EAGER` forbidden
- **Auth:** JWT Bearer Token (24h expiry) + refresh token — bcrypt min. 12 rounds
- **Testing:** JUnit 5, Testcontainers, Cucumber (critical flows only)
- **Quality:** ktlint (CI blocks merge on failure), ArchUnit required
- **Local infra:** Docker + Docker Compose
- **API docs:** OpenAPI / Swagger

> When generating code involving libraries or frameworks, use **Context7 (MCP)** to ensure up-to-date APIs.

---

## Architecture: Clean Architecture + DDD

| Layer | Responsibility | Allowed dependencies |
|---|---|---|
| `domain` | Entities, aggregates, Value Objects, business rules | **None** |
| `application` | Use cases, orchestration, outbound ports | `domain` |
| `presentation` | REST controllers, DTOs, OpenAPI | `application` |
| `infrastructure` | JPA, adapters, external integrations | `application`, `domain` |

### Non-negotiable rules

- `domain` never imports any framework (Spring, JPA, etc.)
- Business rules live **exclusively** in `domain`
- `@Service` and `@Component` are **forbidden** in `domain` — use `@DomainService` (registered via `DomainServiceScanConfig` in infrastructure)
- Native Kotlin types (`String`, `UUID`, `Int`, `Long`) **never** used as business fields in entities, VOs or aggregates — always use `value class`
- Modules never import each other's internal classes — only contracts (ports, events, commands)

---

## Available skills

Skills complement these instructions with detailed implementation recipes.
Copilot loads them automatically when relevant, or invoke manually with `/name`:

| Skill | When it loads |
|---|---|
| `/domain-language` | **Always consult** — domain names, error codes, tables, realistic examples |
| `/use-case` | Creating Use Cases in the `application` layer |
| `/controller` | Creating REST Controllers and DTOs in the `presentation` layer |
| `/value-object` | Creating `value class` and domain types |
| `/migration` | Creating Flyway migrations |
| `/test-builder` | Creating unit tests, integration tests and builders |
| `/error-handling` | Creating domain exceptions and `GlobalExceptionHandler` |
| `/openapi` | Documenting endpoints with OpenAPI/Swagger |
| `/bdd` | Creating Cucumber scenarios for critical flows |
| `/ddd-modeling` | Modeling aggregates, bounded contexts and ubiquitous language |

---

## API

- **Base path:** `/api/v1/`
- **Auth:** `Authorization: Bearer <token>`
- **Responses:** JSON — all fields **in English**, camelCase
- **Error prefix for this service:** `{{ERROR_PREFIX}}` (e.g. `{{ERROR_PREFIX}}_001`)
- **Common codes:** `CO_001` (400 invalid input), `CO_002` (400 invalid body), `CO_999` (500)

### Public endpoints (no authentication required)

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/forgot-password`

### HTTP error pattern (NON-NEGOTIABLE)

| Code | When to use |
|---|---|
| `400` | **Input** validation failure — invalid, missing or malformed field |
| `422` | **Business rule** violation |
| `403` | Resource exists but user lacks permission — **never use `404`** to hide existence |
| `404` | Resource genuinely does not exist |

- `GlobalExceptionHandler` is the **only** place that builds `ErrorResponse`
- Responses **never** expose stack traces or Java/Kotlin exception messages

---

## Security

- Every endpoint requires JWT, except the 3 public auth endpoints
- Authorization is verified **in the use case**, before any domain operation
- Controllers **never** check permissions
- Data isolation: all queries filtered by the authenticated `user_id`

---

## Database

- Schema changes via **Flyway** only
- **Soft delete required** — hard delete forbidden — `deleted_at TIMESTAMP NULL` column on all business tables
- `deleted_at IS NULL` filter is the exclusive responsibility of the `infrastructure` layer
- Monetary values always stored in **cents** (`BIGINT`) — never `FLOAT` or `DECIMAL`

---

## Quality and testing

- **Minimum coverage:** `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`
- **Test builders required** — Kotlin `object` pattern, in `src/testFixtures/kotlin/testbuilders/`
- Unit tests in `src/test/kotlin/`, integration tests in `src/integrationTest/kotlin/`
- **Testcontainers** for tests that hit a real database
- **BDD with Cucumber** for critical flows only — `.feature` scenarios written in Portuguese (PT-BR)
- **ArchUnit** required: `domain` without Spring, `infrastructure` without importing `presentation`
- **ktlint** required — CI blocks merge on lint failure
- N+1 prevention: `FetchType.EAGER` forbidden, never load associations inside a loop

---

## Documentation

- OpenAPI documentation written in **Brazilian Portuguese (PT-BR)**
- **Mermaid** diagrams (domain and ERD) updated whenever the domain changes
- KDoc only when the "why" is not obvious from the code

---

## Versioning

Series `{{VERSION_SERIES}}` — MAJOR stays fixed throughout the MVP.

| Change | Version bump |
|---|---|
| New Spec or set of business features delivered | MINOR |
| Fixes and adjustments to already-delivered specs | PATCH |

**Required commit format:** `type(module): description in English — SPEC-XX`
Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`
Example: `feat(expense): register installment purchase on credit card — SPEC-03`

### `/release` command

When `/release` is received:
1. List commits since the last tag
2. Calculate next version (`feat` → MINOR, only `fix`/`chore` → PATCH)
3. Update `CHANGELOG.md` with delivered specs and features
4. Create commit `chore(release): bump version to X.Y.Z` + annotated tag `vX.Y.Z`
5. Display summary and **wait for confirmation** before `git push --follow-tags`