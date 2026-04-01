---
name: value-object
description: >-
  Use when creating Value Objects (VOs) or domain types following DDD with Kotlin.
  Activate for terms like: "create value object", "create VO", "create value class",
  "domain type", "business field", "encapsulate primitive", "value class",
  "typed identifier", "domain ID", or any field that represents
  a business concept and needs its own validation or semantics.
  Stack: Kotlin.
argument-hint: "[Value Object name] [aggregate or context]"
user-invocable: true
disable-model-invocation: false
---

# How to create Value Objects — DDD with Kotlin

> Consult `/domain-language` for Value Objects already defined in the project
> before creating new ones to avoid duplication.

## Core rule (NON-NEGOTIABLE)

Native types (`String`, `UUID`, `Int`, `Long`, etc.) **never** appear as business
fields in entities, VOs, or aggregates. Always use `value class`.

## Package structure

```
domain/aggregates/{aggregate}/valueobjects/   # VO exclusive to the aggregate
domain/aggregates/valueobjects/               # VOs shared across aggregates
```

## Template: identifier VO (single field)

```kotlin
@JvmInline
value class {Entity}Id(val value: UUID) {
    companion object {
        fun generate(): {Entity}Id = {Entity}Id(UUID.randomUUID())
        fun from(value: String): {Entity}Id = {Entity}Id(UUID.fromString(value))
    }
}
```

## Template: VO with business validation

```kotlin
@JvmInline
value class {Name}(val value: {PrimitiveType}) {
    init {
        require(/* invariant */) { "Clear error message" }
    }
}
```

## Template: composite VO (more than one field → data class)

```kotlin
data class {Name}(
    val field1: {Type1},
    val field2: {Type2},
) {
    init {
        require(/* invariant */) { "Clear error message" }
    }
    // relevant business methods
}
```

## Design rules

1. `@JvmInline` on every single-field `value class` — avoids boxing
2. Validation in `init` with `require()` — clear message
3. Zero framework in domain — no Spring, no JPA
4. Only `val` — no `var`, no setters
5. Business methods and operators when relevant
6. Factory methods in `companion object`

## JPA mapping (infrastructure's exclusive responsibility)

```kotlin
// The infrastructure adapter converts between JpaEntity (primitives) and domain (VOs)
@Entity
class {Aggregate}JpaEntity(
    @Id val id: UUID,       // primitive — mapped from {Entity}Id
    val amount: Long,       // primitive — mapped from domain value class
)
```

## Checklist

- [ ] `@JvmInline` + `value class` for single-field VO
- [ ] `data class` for VO with more than one field
- [ ] Validation in `init` with `require()` and descriptive message
- [ ] No framework imports
- [ ] Only `val`
- [ ] Factory methods in `companion object` when needed
- [ ] Tests: valid creation, invalid creation (invariants), operations