---
name: use-case
description: >-
  Use ao criar um novo Use Case na camada application seguindo Clean Architecture + DDD.
  Ative para termos como: "criar use case", "caso de uso", "novo fluxo de negócio",
  "implementar feature", "orquestrar domínio", ou qualquer ação que envolva
  orquestrar entidades de domínio em resposta a uma intenção do usuário.
  Stack: Kotlin + Spring Boot.
argument-hint: "[nome do caso de uso] [agregado envolvido]"
user-invocable: true
disable-model-invocation: false
---

# Como criar um Use Case — Clean Architecture + DDD (Kotlin/Spring Boot)

> Os nomes de agregados, Value Objects e códigos de erro do projeto estão
> definidos na skill `/domain-language`. Consulte-a ao nomear tipos e códigos.

## Estrutura de pacotes

```
application/usecases/{agregado}/{nomecasodeuso}/
├── {Nome}Input.kt
├── {Nome}Output.kt
└── {Nome}UseCase.kt
```

## Regras obrigatórias

- Sufixos `Input`, `Output`, `UseCase` — nunca `DTO`, `Command`, `Request`, `Response`, `Result`
- `@Transactional` **somente** no `execute()` — nunca no domain service
- Entidades de domínio **nunca** retornadas diretamente — sempre via `Output`
- Verificação de **autorização** ocorre no UseCase **antes** de qualquer operação de domínio
- Campos de `Input` usam `value class` do domínio — nunca `String`, `UUID`, `Int` puros

## Template: {Nome}Input.kt

```kotlin
data class {Nome}Input(
    val userId: UserId,
    // demais campos como value class do domínio
)
```

## Template: {Nome}Output.kt

```kotlin
data class {Nome}Output(
    val id: UUID,           // tipos primitivos — facilitam serialização
    val createdAt: Instant,
    // demais campos como tipos primitivos
)
```

## Template: {Nome}UseCase.kt

```kotlin
@Service
class {Nome}UseCase(
    private val repository: {Agregado}Repository,
) {
    @Transactional
    fun execute(input: {Nome}Input): {Nome}Output {
        // 1. Verificar autorização ANTES de qualquer operação
        val entity = repository.findById(input.entityId)
            ?: throw {Entidade}NotFoundException(input.entityId)

        if (entity.userId != input.userId) {
            throw {Entidade}AccessDeniedException(input.entityId)  // → HTTP 403
        }

        // 2. Executar lógica de domínio
        entity.executarOperacao()

        // 3. Persistir
        val saved = repository.save(entity)

        // 4. Retornar Output — NUNCA a entidade
        return {Nome}Output(
            id = saved.id.value,
            createdAt = saved.createdAt,
        )
    }
}
```

## Exceções de domínio

```kotlin
// → HTTP 404
class {Entidade}NotFoundException(id: {Entidade}Id) : ResourceNotFoundException(
    message = "{Entidade} não encontrada: ${id.value}",
    code = "{PREFIXO}_NNN",   // prefixo definido em /domain-language
)

// → HTTP 403
class {Entidade}AccessDeniedException(id: {Entidade}Id) : AuthorizationException(
    message = "Acesso negado: ${id.value}",
    code = "{PREFIXO}_NNN",
)

// → HTTP 422
class {Regra}Exception(...) : BusinessRuleException(
    message = "Descrição da regra violada",
    code = "{PREFIXO}_NNN",
)
```

## Checklist

- [ ] Pacote: `application/usecases/{agregado}/{caso}/`
- [ ] Sufixos `Input`, `Output`, `UseCase` presentes
- [ ] `@Transactional` somente no `execute()`
- [ ] Autorização verificada antes de qualquer operação de domínio
- [ ] Nenhuma entidade de domínio retornada diretamente
- [ ] Campos de `Input` usam `value class` do domínio
- [ ] Exceções com código no formato do projeto (ver `/domain-language`)
- [ ] Teste unitário com builder