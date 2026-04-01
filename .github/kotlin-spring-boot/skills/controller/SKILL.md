---
name: controller
description: >-
  Use when creating a REST Controller in the presentation layer following Clean Architecture.
  Activate for terms like: "create controller", "new endpoint", "create route", "expose API",
  "create ControllerDoc", "create Request", "create Response", "document endpoint",
  "add endpoint", or any task that involves receiving HTTP requests
  and delegating to a Use Case. Stack: Kotlin + Spring Boot + OpenAPI.
argument-hint: "[resource] [operation]"
user-invocable: true
disable-model-invocation: false
---

# How to create a REST Controller — Clean Architecture (Kotlin/Spring Boot)

> Consult `/domain-language` for resource names, error prefixes,
> and base endpoints of the project.

## Package structure

```
presentation/controllers/{resource}/
├── {Resource}ControllerDoc.kt    # interface — ONLY OpenAPI annotations
├── {Resource}Controller.kt       # implements Doc — ONLY Spring MVC annotations
├── {Case}Request.kt              # input DTO
└── {Case}Response.kt             # output DTO
```

## Core rule (NON-NEGOTIABLE)

OpenAPI annotations **NEVER** go in the controller.
The controller implements `{Resource}ControllerDoc` and contains **only** Spring MVC.

## Template: {Resource}ControllerDoc.kt

```kotlin
@Tag(name = "{Resource}", description = "Descrição do recurso em PT-BR")
interface {Resource}ControllerDoc {
    
    @Operation(summary = "Ação curta em PT-BR", description = "Objetivo, pré-condições, papéis.")
    @ApiResponse(responseCode = "201", description = "Criado com sucesso")
    @ApiResponse(responseCode = "400", description = "Dados inválidos (CO_001)")
    @ApiResponse(responseCode = "401", description = "Token ausente ou expirado")
    @ApiResponse(responseCode = "422", description = "Regra de negócio violada")
    fun create{Resource}(
        @RequestBody request: Create{Resource}Request,
        authentication: Authentication,
    ): ResponseEntity<Create{Resource}Response>
}
```

## Template: {Resource}Controller.kt

```kotlin
@RestController
@RequestMapping("/api/v1/{resources}")
class {Resource}Controller(
    private val useCase: Create{Resource}UseCase,
) : {Resource}ControllerDoc {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    override fun create{Resource}(
        @RequestBody @Valid request: Create{Resource}Request,
        authentication: Authentication,
    ): ResponseEntity<Create{Resource}Response> {
        val userId = UserId(UUID.fromString(authentication.name))
        val output = useCase.execute(Create{Resource}Input(userId = userId /*, ... */))
        return ResponseEntity.status(HttpStatus.CREATED).body(Create{Resource}Response.from(output))
    }
}
```

## Template: {Case}Request.kt

```kotlin
data class Create{Resource}Request(
    @Schema(description = "Field description in PT-BR", example = "realistic value")
    @field:NotNull(message = "Required field")
    val field: Type,
)
```

Rules:
- JSON fields **always in English** camelCase — `@JsonProperty` with Portuguese is **prohibited**
- `@field:` prefix required in Kotlin data classes for Bean Validation to work
- Validation with `@Valid` → HTTP 400 on failure

## Template: {Case}Response.kt

```kotlin
data class Create{Resource}Response(
    @Schema(description = "Description in PT-BR", example = "realistic value")
    val field: Type,
) {
    companion object {
        fun from(output: Create{Resource}Output) = Create{Resource}Response(field = output.field)
    }
}
```

## Prohibitions

| Prohibited | Correct |
|---|---|
| `@Operation` in controller | `@Operation` only in the Doc interface |
| `@JsonProperty("ptName")` | JSON fields in English camelCase |
| Authorization check in controller | Delegate to the UseCase |
| Building `ErrorResponse` in controller | Only `GlobalExceptionHandler` |

## Checklist

- [ ] `ControllerDoc` contains only OpenAPI, `Controller` contains only Spring MVC
- [ ] JSON fields in English camelCase in Request and Response
- [ ] `@field:` prefix on all validation annotations
- [ ] `@Valid` on `@RequestBody` in controller
- [ ] `@Schema` with `description` and realistic `example` on all fields
- [ ] Controller does not check authorization