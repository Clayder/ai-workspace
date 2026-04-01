---
name: error-handling
description: >-
  Use when implementing error handling, domain exceptions, or HTTP error responses
  following Clean Architecture with Kotlin and Spring Boot. Activate for terms like:
  "create domain exception", "business error", "GlobalExceptionHandler",
  "error 400", "error 422", "error 403", "DomainException", "handle exception",
  "error response", "error code", "violations", "API error pattern",
  or any task related to HTTP error handling.
argument-hint: "[error type or exception to implement]"
user-invocable: true
disable-model-invocation: false
---

# How to implement error handling — Clean Architecture (Kotlin/Spring Boot)

> Consult `/domain-language` for error code prefixes
> and the existing code catalog for this project.

## HTTP error pattern (NON-NEGOTIABLE)

| Code | When to use | Origin | Structure |
|---|---|---|---|
| `400` | **Input** validation (invalid field, missing, wrong format) | `@Valid` in `presentation` | `{ message, code, violations[] }` |
| `422` | **Business rule** violation | Domain exception | `{ message, code }` — no violations |
| `403` | Resource exists but user lacks permission | Use case | `{ message, code }` |
| `404` | Resource genuinely does not exist | Use case | `{ message, code }` |
| `401` | Token missing or expired | Spring Security | `{ message, code }` |

**PROHIBITED:**
- `400` for business rules
- `422` for invalid input fields
- `404` to mask existence when it is an authorization problem (use `403`)
- Stack trace or Java/Kotlin exception message in error responses

---

## Domain exception hierarchy

```kotlin
// domain/exceptions/DomainException.kt
abstract class DomainException(
    override val message: String,
    val code: String,
) : RuntimeException(message)

abstract class ResourceNotFoundException(message: String, code: String) :
    DomainException(message, code)

abstract class BusinessRuleException(message: String, code: String) :
    DomainException(message, code)

abstract class AuthorizationException(message: String, code: String) :
    DomainException(message, code)
```

## Aggregate-specific exceptions

```kotlin
// Service prefix defined in /domain-language

class {Entity}NotFoundException(id: {Entity}Id) : ResourceNotFoundException(
    message = "{Entity} not found: ${id.value}",
    code = "{PREFIX}_NNN",    // → HTTP 404
)

class {Entity}AccessDeniedException(id: {Entity}Id) : AuthorizationException(
    message = "Access denied: ${id.value}",
    code = "{PREFIX}_NNN",    // → HTTP 403
)

class {Rule}Exception(...) : BusinessRuleException(
    message = "Clear description of the violated rule",
    code = "{PREFIX}_NNN",    // → HTTP 422
)
```

---

## ErrorResponse (DTO — presentation/shared)

```kotlin
data class ErrorResponse(
    val message: String,
    val code: String,
    val violations: List<ViolationResponse>? = null,
)

data class ViolationResponse(
    val property: String,
    val message: String,
    val value: Any?,
)
```

---

## GlobalExceptionHandler (sole owner of ErrorResponse)

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(ex: ResourceNotFoundException): ErrorResponse =
        ErrorResponse(message = ex.message, code = ex.code)

    @ExceptionHandler(BusinessRuleException::class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    fun handleBusinessRule(ex: BusinessRuleException): ErrorResponse =
        ErrorResponse(message = ex.message, code = ex.code)

    @ExceptionHandler(AuthorizationException::class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    fun handleAuthorization(ex: AuthorizationException): ErrorResponse =
        ErrorResponse(message = ex.message, code = ex.code)

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(ex: MethodArgumentNotValidException): ErrorResponse =
        ErrorResponse(
            message = "Dados de entrada inválidos",
            code = "CO_001",
            violations = ex.bindingResult.fieldErrors.map {
                ViolationResponse(it.field, it.defaultMessage ?: "Invalid value", it.rejectedValue)
            }
        )

    @ExceptionHandler(HttpMessageNotReadableException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleNotReadable(ex: HttpMessageNotReadableException): ErrorResponse =
        ErrorResponse(message = "Corpo da requisição inválido ou ausente", code = "CO_002")

    @ExceptionHandler(Exception::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleGeneric(ex: Exception): ErrorResponse =
        ErrorResponse(message = "Erro interno do servidor", code = "CO_999")
}
```

---

## Checklist

- [ ] Exception extends `DomainException` (or subclass) with `code` in the project format
- [ ] `400` only for input — `422` only for business rule violations
- [ ] `403` for existing resource without permission — never `404`
- [ ] `GlobalExceptionHandler` is the only place building `ErrorResponse`
- [ ] Response never exposes a stack trace
- [ ] Code registered in the catalog in `/domain-language` and documented in Swagger