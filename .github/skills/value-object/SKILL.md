---
name: value-object
description: >-
  Use ao criar Value Objects (VOs) ou tipos de domínio seguindo DDD com Kotlin.
  Ative para termos como: "criar value object", "criar VO", "criar value class",
  "tipo de domínio", "campo de negócio", "encapsular primitivo", "value class",
  "identificador tipado", "ID de domínio", ou qualquer campo que represente
  um conceito de negócio e precise de validação ou semântica própria.
  Stack: Kotlin.
argument-hint: "[nome do Value Object] [agregado ou contexto]"
user-invocable: true
disable-model-invocation: false
---

# Como criar Value Objects — DDD com Kotlin

> Consulte `/domain-language` para os Value Objects já existentes no projeto
> antes de criar novos, evitando duplicação.

## Regra fundamental (NÃO NEGOCIÁVEL)

Tipos nativos (`String`, `UUID`, `Int`, `Long`, etc.) **nunca** aparecem como campos
de negócio em entidades, VOs ou agregados. Use sempre `value class`.

## Estrutura de pacotes

```
domain/aggregates/{agregado}/valueobjects/   # VO exclusivo do agregado
domain/aggregates/valueobjects/              # VOs compartilhados entre agregados
```

## Template: VO de identificador (campo único)

```kotlin
@JvmInline
value class {Entidade}Id(val value: UUID) {
    companion object {
        fun generate(): {Entidade}Id = {Entidade}Id(UUID.randomUUID())
        fun from(value: String): {Entidade}Id = {Entidade}Id(UUID.fromString(value))
    }
}
```

## Template: VO com validação de negócio

```kotlin
@JvmInline
value class {Nome}(val value: {TipoPrimitivo}) {
    init {
        require(/* invariante */) { "Mensagem clara de erro" }
    }
}
```

## Template: VO composto (mais de um campo → data class)

```kotlin
data class {Nome}(
    val campo1: {Tipo1},
    val campo2: {Tipo2},
) {
    init {
        require(/* invariante */) { "Mensagem clara de erro" }
    }
    // métodos de negócio relevantes
}
```

## Regras de design

1. `@JvmInline` em todo `value class` de campo único — evita boxing
2. Validação no `init` com `require()` — mensagem clara
3. Zero framework no domínio — sem Spring, JPA
4. Apenas `val` — sem `var`, sem setters
5. Métodos e operadores de negócio quando relevante
6. Factory methods no `companion object`

## Mapeamento JPA (responsabilidade exclusiva da infrastructure)

```kotlin
// O adapter em infrastructure converte entre JpaEntity (primitivos) e domínio (VOs)
@Entity
class {Agregado}JpaEntity(
    @Id val id: UUID,         // primitivo — mapeado de {Entidade}Id
    val valor: Long,          // primitivo — mapeado de value class do domínio
)
```

## Checklist

- [ ] `@JvmInline` + `value class` para VO de campo único
- [ ] `data class` para VO com mais de um campo
- [ ] Validação no `init` com `require()` e mensagem descritiva
- [ ] Nenhuma importação de framework
- [ ] Apenas `val`
- [ ] Factory methods no `companion object` quando necessário
- [ ] Teste: criação válida, criação inválida (invariantes), operações