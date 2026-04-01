---
name: error-handling
description: >-
  Use ao implementar tratamento de erros, exceções de domínio ou respostas HTTP de erro
  seguindo Clean Architecture com Kotlin e Spring Boot. Ative para termos como:
  "criar exceção de domínio", "erro de negócio", "GlobalExceptionHandler",
  "erro 400", "erro 422", "erro 403", "DomainException", "tratar exceção",
  "response de erro", "error code", "violations", "padrão de erro da API",
  ou qualquer tarefa relacionada ao tratamento de erros HTTP.
argument-hint: "[tipo de erro ou exceção a implementar]"
user-invocable: true
disable-model-invocation: false
---

# Como implementar tratamento de erros — Clean Architecture (Kotlin/Spring Boot)

> Consulte `/domain-language` para os prefixos de código de erro
> e o catálogo de codes já existentes no projeto.

## Padrão de erros HTTP (NÃO NEGOCIÁVEL)

| Código | Quando usar | Origem | Estrutura |
|---|---|---|---|
| `400` | Validação de **entrada** (campo inválido, ausente, formato errado) | `@Valid` na `presentation` | `{ message, code, violations[] }` |
| `422` | Violação de **regra de negócio** | Exceção de domínio | `{ message, code }` — sem violations |
| `403` | Recurso existe mas usuário sem permissão | Use case | `{ message, code }` |
| `404` | Recurso genuinamente inexistente | Use case | `{ message, code }` |
| `401` | Token ausente ou expirado | Spring Security | `{ message, code }` |

**PROIBIDO:**
- `400` para regra de negócio
- `422` para campo inválido de entrada
- `404` para mascarar existência quando é problema de autorização (use `403`)
- Stack trace ou mensagem de exceção Java/Kotlin em respostas de erro

---

## Hierarquia de exceções de domínio

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

## Exceções específicas por agregado

```kotlin
// Prefixo do serviço definido em /domain-language

class {Entidade}NotFoundException(id: {Entidade}Id) : ResourceNotFoundException(
    message = "{Entidade} não encontrada: ${id.value}",
    code = "{PREFIXO}_NNN",    // → HTTP 404
)

class {Entidade}AccessDeniedException(id: {Entidade}Id) : AuthorizationException(
    message = "Acesso negado: ${id.value}",
    code = "{PREFIXO}_NNN",    // → HTTP 403
)

class {Regra}Exception(...) : BusinessRuleException(
    message = "Descrição clara da regra violada",
    code = "{PREFIXO}_NNN",    // → HTTP 422
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

## GlobalExceptionHandler (ÚNICO responsável por ErrorResponse)

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
                ViolationResponse(it.field, it.defaultMessage ?: "Valor inválido", it.rejectedValue)
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

- [ ] Exceção estende `DomainException` (ou subclasse) com `code` no formato do projeto
- [ ] `400` apenas para entrada — `422` apenas para regra de negócio
- [ ] `403` para recurso existente sem permissão — nunca `404`
- [ ] `GlobalExceptionHandler` é o único construindo `ErrorResponse`
- [ ] Resposta nunca expõe stack trace
- [ ] Code registrado no catálogo em `/domain-language` e documentado no Swagger