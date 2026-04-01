---
name: openapi
description: >-
  Use when documenting REST endpoints with OpenAPI/Swagger in Kotlin + Spring Boot projects.
  Activate for terms like: "document endpoint", "add Swagger", "create ControllerDoc",
  "@Operation", "@Schema", "@ApiResponse", "@ExampleObject", "API documentation",
  "OpenAPI annotations", "describe endpoint", "add Swagger examples",
  or any task involving HTTP contract documentation.
argument-hint: "[endpoint or resource to document]"
user-invocable: true
disable-model-invocation: false
---

# How to document with OpenAPI — Kotlin/Spring Boot

> Consult `/domain-language` for realistic domain examples
> (IDs, values, formats) when filling in `@ExampleObject` and `@Schema`.

## Core rule (NON-NEGOTIABLE)

OpenAPI annotations **NEVER** go in the controller.
All documentation goes in the `{Resource}ControllerDoc.kt` interface.

## Language (NON-NEGOTIABLE)

All OpenAPI documentation must be in **Brazilian Portuguese**.

---

## Template: {Resource}ControllerDoc.kt

```kotlin
@Tag(name = "{Resource}", description = "Resource description in PT-BR")
interface {Resource}ControllerDoc {

    @Operation(
        summary = "Short action in PT-BR",
        description = "Objective. Preconditions. Authorized roles.",
        requestBody = io.swagger.v3.oas.annotations.parameters.RequestBody(
            content = [Content(
                mediaType = "application/json",
                examples = [ExampleObject(
                    name = "Main example",
                    summary = "Scenario in PT-BR",
                    value = """{ "field": "realistic domain value" }"""
                )]
            )]
        )
    )
    @ApiResponse(
        responseCode = "201",
        description = "Resource created successfully",
        content = [Content(examples = [ExampleObject(
            value = """{ "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" }"""
        )])]
    )
    @ApiResponse(responseCode = "400", description = "Invalid input data (CO_001)")
    @ApiResponse(responseCode = "401", description = "JWT token missing or expired")
    @ApiResponse(responseCode = "422", description = "Business rule violation")
    fun {operation}(
        @RequestBody request: {Case}Request,
        authentication: Authentication,
    ): ResponseEntity<{Case}Response>
}
```

## Template: @Schema on Request and Response fields

```kotlin
@Schema(
    description = "Field description in PT-BR — business meaning",
    example = "realistic domain value",   // never "string", "0", or "uuid"
)
val field: Type
```

---

## Absolute prohibitions

| Prohibited | Correct |
|---|---|
| `example = "string"` | `example = "realistic domain value"` |
| `example = "0"` | Realistic numeric value |
| `example = "uuid"` | `example = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"` |
| `description = "Id"` | `description = "Unique resource identifier"` |
| Documentation in English | Documentation in PT-BR |
| `@Operation` in controller | Only in the `ControllerDoc` interface |

---

## Checklist

- [ ] All documentation in PT-BR
- [ ] `@Tag` with `name` and `description`
- [ ] `@Operation` with complete `summary` and `description`
- [ ] `@ApiResponse` for all possible statuses (2xx, 400, 401, 403, 422)
- [ ] `@ExampleObject` with realistic domain data
- [ ] `@Schema` on all Request and Response fields
- [ ] No generic examples (`"string"`, `"0"`, `"uuid"`)
- [ ] OpenAPI annotations only in the `ControllerDoc` interface
- [ ] Error codes in `@ApiResponse` documented in Swagger and in `/domain-language`