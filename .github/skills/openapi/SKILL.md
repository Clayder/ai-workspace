---
name: openapi
description: >-
  Use ao documentar endpoints REST com OpenAPI/Swagger em projetos Kotlin + Spring Boot.
  Ative para termos como: "documentar endpoint", "adicionar Swagger", "criar ControllerDoc",
  "@Operation", "@Schema", "@ApiResponse", "@ExampleObject", "documentação da API",
  "anotações OpenAPI", "descrever endpoint", "adicionar exemplos no Swagger",
  ou qualquer tarefa que envolva documentação de contratos HTTP.
argument-hint: "[endpoint ou recurso a documentar]"
user-invocable: true
disable-model-invocation: false
---

# Como documentar com OpenAPI — Kotlin/Spring Boot

> Consulte `/domain-language` para exemplos realistas do domínio do projeto
> (IDs, valores, formatos) ao preencher `@ExampleObject` e `@Schema`.

## Regra fundamental (NÃO NEGOCIÁVEL)

Anotações OpenAPI **NUNCA** ficam no controller.
Toda documentação fica na interface `{Recurso}ControllerDoc.kt`.

## Idioma (NÃO NEGOCIÁVEL)

Toda documentação OpenAPI em **português do Brasil**.

---

## Template: {Recurso}ControllerDoc.kt

```kotlin
@Tag(name = "{Recurso}", description = "Descrição do recurso em PT-BR")
interface {Recurso}ControllerDoc {

    @Operation(
        summary = "Ação curta em PT-BR",
        description = "Objetivo. Pré-condições. Papéis autorizados.",
        requestBody = io.swagger.v3.oas.annotations.parameters.RequestBody(
            content = [Content(
                mediaType = "application/json",
                examples = [ExampleObject(
                    name = "Exemplo principal",
                    summary = "Cenário em PT-BR",
                    value = """{ "campo": "valor realista do domínio" }"""
                )]
            )]
        )
    )
    @ApiResponse(
        responseCode = "201",
        description = "Recurso criado com sucesso",
        content = [Content(examples = [ExampleObject(
            value = """{ "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" }"""
        )])]
    )
    @ApiResponse(responseCode = "400", description = "Dados de entrada inválidos (CO_001)")
    @ApiResponse(responseCode = "401", description = "Token JWT ausente ou expirado")
    @ApiResponse(responseCode = "422", description = "Regra de negócio violada")
    fun {operacao}(
        @RequestBody request: {Caso}Request,
        authentication: Authentication,
    ): ResponseEntity<{Caso}Response>
}
```

## Template: @Schema em campos de Request e Response

```kotlin
@Schema(
    description = "Descrição do campo em PT-BR — significado no negócio",
    example = "valor realista do domínio",   // nunca "string", "0" ou "uuid"
)
val campo: Tipo
```

---

## Proibições absolutas

| Proibido | Correto |
|---|---|
| `example = "string"` | `example = "valor real do domínio"` |
| `example = "0"` | Valor numérico realista |
| `example = "uuid"` | `example = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"` |
| `description = "Id"` | `description = "Identificador único do recurso"` |
| Documentação em inglês | Documentação em PT-BR |
| `@Operation` no controller | Apenas na interface `ControllerDoc` |

---

## Checklist

- [ ] Toda documentação em PT-BR
- [ ] `@Tag` com `name` e `description`
- [ ] `@Operation` com `summary` e `description` completa
- [ ] `@ApiResponse` para todos os status possíveis (2xx, 400, 401, 403, 422)
- [ ] `@ExampleObject` com dados realistas do domínio
- [ ] `@Schema` em todos os campos de Request e Response
- [ ] Nenhum exemplo genérico (`"string"`, `"0"`, `"uuid"`)
- [ ] Anotações OpenAPI apenas na interface `ControllerDoc`
- [ ] Codes de erro dos `@ApiResponse` documentados no Swagger e em `/domain-language`