---
name: controller
description: >-
  Use ao criar um Controller REST na camada presentation seguindo Clean Architecture.
  Ative para termos como: "criar controller", "novo endpoint", "criar rota", "expor API",
  "criar ControllerDoc", "criar Request", "criar Response", "documentar endpoint",
  "adicionar endpoint", ou qualquer tarefa que envolva receber requisições HTTP
  e delegar para um Use Case. Stack: Kotlin + Spring Boot + OpenAPI.
argument-hint: "[recurso] [operação]"
user-invocable: true
disable-model-invocation: false
---

# Como criar um Controller REST — Clean Architecture (Kotlin/Spring Boot)

> Consulte `/domain-language` para nomes de recursos, prefixos de erro
> e endpoints base do projeto.

## Estrutura de pacotes

```
presentation/controllers/{recurso}/
├── {Recurso}ControllerDoc.kt    # interface — APENAS anotações OpenAPI
├── {Recurso}Controller.kt       # implementa Doc — APENAS anotações Spring MVC
├── {Caso}Request.kt             # DTO de entrada
└── {Caso}Response.kt            # DTO de saída
```

## Regra fundamental (NÃO NEGOCIÁVEL)

Anotações OpenAPI **NUNCA** ficam no controller.
O controller implementa `{Recurso}ControllerDoc` e contém **apenas** Spring MVC.

## Template: {Recurso}ControllerDoc.kt

```kotlin
@Tag(name = "{Recurso}", description = "Descrição em PT-BR")
interface {Recurso}ControllerDoc {

    @Operation(summary = "Ação curta", description = "Objetivo, pré-condições, papéis.")
    @ApiResponse(responseCode = "201", description = "Criado com sucesso")
    @ApiResponse(responseCode = "400", description = "Dados inválidos (CO_001)")
    @ApiResponse(responseCode = "401", description = "Token ausente ou expirado")
    @ApiResponse(responseCode = "422", description = "Regra de negócio violada")
    fun criar{Recurso}(
        @RequestBody request: Criar{Recurso}Request,
        authentication: Authentication,
    ): ResponseEntity<Criar{Recurso}Response>
}
```

## Template: {Recurso}Controller.kt

```kotlin
@RestController
@RequestMapping("/api/v1/{recursos}")
class {Recurso}Controller(
    private val useCase: Criar{Recurso}UseCase,
) : {Recurso}ControllerDoc {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    override fun criar{Recurso}(
        @RequestBody @Valid request: Criar{Recurso}Request,
        authentication: Authentication,
    ): ResponseEntity<Criar{Recurso}Response> {
        val userId = UserId(UUID.fromString(authentication.name))
        val output = useCase.execute(Criar{Recurso}Input(userId = userId /*, ... */))
        return ResponseEntity.status(HttpStatus.CREATED).body(Criar{Recurso}Response.from(output))
    }
}
```

## Template: {Caso}Request.kt

```kotlin
data class Criar{Recurso}Request(
    @Schema(description = "Descrição do campo em PT-BR", example = "valor realista")
    @field:NotNull(message = "Campo obrigatório")
    val campo: Tipo,
)
```

Regras:
- Campos JSON **sempre em inglês** camelCase — `@JsonProperty` com português é **proibido**
- `@field:` prefix obrigatório em data classes Kotlin para Bean Validation funcionar
- Validação com `@Valid` → HTTP 400 quando falha

## Template: {Caso}Response.kt

```kotlin
data class Criar{Recurso}Response(
    @Schema(description = "Descrição em PT-BR", example = "valor realista")
    val campo: Tipo,
) {
    companion object {
        fun from(output: Criar{Recurso}Output) = Criar{Recurso}Response(campo = output.campo)
    }
}
```

## Proibições

| Proibido | Correto |
|---|---|
| `@Operation` no controller | `@Operation` somente na interface Doc |
| `@JsonProperty("nomePT")` | Campos JSON em inglês camelCase |
| Verificar autorização no controller | Delegar ao UseCase |
| Construir `ErrorResponse` no controller | Apenas `GlobalExceptionHandler` |

## Checklist

- [ ] `ControllerDoc` contém apenas OpenAPI, `Controller` contém apenas Spring MVC
- [ ] Campos JSON em inglês camelCase em Request e Response
- [ ] `@field:` prefix em todas as anotações de validação
- [ ] `@Valid` no `@RequestBody` do controller
- [ ] `@Schema` com `description` e `example` realista em todos os campos
- [ ] Controller não verifica autorização