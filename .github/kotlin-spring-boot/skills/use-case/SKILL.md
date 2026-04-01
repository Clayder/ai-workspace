---
name: use-case
description: >-
  Use when creating a new Use Case in the application layer following Clean Architecture + DDD.
  Activate for terms like: "create use case", "new business flow",
  "implement feature", "orchestrate domain", or any action that involves
  orchestrating domain entities in response to a user intent.
  Stack: Kotlin + Spring Boot.
argument-hint: "[use case name] [aggregate involved]"
user-invocable: true
disable-model-invocation: false
---

# How to create a Use Case — Clean Architecture + DDD (Kotlin/Spring Boot)

> Aggregate names, Value Objects, and error codes for this project are defined
> in the `/domain-language` skill. Consult it when naming types and codes.

## Package structure

```
application/usecases/{aggregate}/{usecasename}/
├── {Name}Input.kt
├── {Name}Output.kt
└── {Name}UseCase.kt
```

## Mandatory rules

- Suffixes `Input`, `Output`, `UseCase` — never `DTO`, `Command`, `Request`, `Response`, `Result`
- `@Transactional` **only** on `execute()` — never on the domain service
- Domain entities **never** returned directly — always via `Output`
- **Authorization** check happens in the UseCase **before** any domain operation
- `Input` fields use domain `value class` types — never raw `String`, `UUID`, `Int`

## Template: {Name}Input.kt

```kotlin
data class {Name}Input(
    val userId: UserId,
    // other fields as domain value classes
)
```

## Template: {Name}Output.kt

```kotlin
data class {Name}Output(
    val id: UUID,           // primitive types — easier serialization
    val createdAt: Instant,
    // other fields as primitive types
)
```

## Template: {Name}UseCase.kt

```kotlin
@Service
class {Name}UseCase(
    private val repository: {Aggregate}Repository,
) {
    @Transactional
    fun execute(input: {Name}Input): {Name}Output {
        // 1. Check authorization BEFORE any domain operation
        val entity = repository.findById(input.entityId)
            ?: throw {Entity}NotFoundException(input.entityId)

        if (entity.userId != input.userId) {
            throw {Entity}AccessDeniedException(input.entityId)  // → HTTP 403
        }

        // 2. Execute domain logic
        entity.performOperation()

        // 3. Persist
        val saved = repository.save(entity)

        // 4. Return Output — NEVER the entity
        return {Name}Output(
            id = saved.id.value,
            createdAt = saved.createdAt,
        )
    }
}
```

## Domain exceptions

```kotlin
// → HTTP 404
class {Entity}NotFoundException(id: {Entity}Id) : ResourceNotFoundException(
    message = "{Entity} not found: ${id.value}",
    code = "{PREFIX}_NNN",   // prefix defined in /domain-language
)

// → HTTP 403
class {Entity}AccessDeniedException(id: {Entity}Id) : AuthorizationException(
    message = "Access denied: ${id.value}",
    code = "{PREFIX}_NNN",
)

// → HTTP 422
class {Rule}Exception(...) : BusinessRuleException(
    message = "Description of the violated rule",
    code = "{PREFIX}_NNN",
)
```

## Checklist

- [ ] Package: `application/usecases/{aggregate}/{case}/`
- [ ] Suffixes `Input`, `Output`, `UseCase` present
- [ ] `@Transactional` only on `execute()`
- [ ] Authorization checked before any domain operation
- [ ] No domain entity returned directly
- [ ] `Input` fields use domain `value class` types
- [ ] Exceptions with code in the project format (see `/domain-language`)
- [ ] Unit test with builder