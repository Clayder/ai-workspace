---
name: value-object
description: >-
  Use ao criar Value Objects (VOs) ou tipos de domínio seguindo DDD com TypeScript.
  Ative para termos como: "criar value object", "criar VO", "tipo de domínio",
  "campo de negócio", "encapsular primitivo", "identificador tipado", "domainType",
  ou qualquer campo que represente um conceito de negócio e precise de validação
  ou semântica própria. Stack: TypeScript + NestJS.
argument-hint: "[nome do Value Object] [agregado ou contexto]"
user-invocable: true
disable-model-invocation: false
---

# Como criar Value Objects — DDD com TypeScript

> Consulte o `copilot-instructions.md` para verificar os Value Objects já definidos
> no projeto antes de criar novos e evitar duplicação.

## Regra inegociável

Tipos nativos (`string`, `number`, `Date`, etc.) **nunca** aparecem como campos
de negócio em entidades, VOs ou agregados. Sempre encapsule em uma classe de VO.

## Localização dos arquivos

```
domain/aggregates/{agregado}/value-objects/   # VO exclusivo do agregado
domain/shared/value-objects/                   # VOs compartilhados entre agregados
```

---

## Template: VO identificador (campo único)

```typescript
// domain/aggregates/{Agregado}/value-objects/{Agregado}Id.ts

export class {Entidade}Id {
  private constructor(readonly value: string) {}

  static generate(): {Entidade}Id {
    return new {Entidade}Id(crypto.randomUUID());
  }

  static from(value: string): {Entidade}Id {
    if (!value || value.trim().length === 0) {
      throw new Error('{Entidade}Id não pode ser vazio');
    }
    return new {Entidade}Id(value);
  }

  equals(other: {Entidade}Id): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

---

## Template: VO com validação de negócio

```typescript
// domain/aggregates/{Agregado}/value-objects/{Nome}.ts

export class {Nome} {
  private constructor(readonly value: {TipoPrimitivo}) {}

  static create(value: {TipoPrimitivo}): {Nome} {
    // Valide a invariante de negócio aqui
    if (/* condição de invalidez */) {
      throw new DomainException('Mensagem clara da regra violada');
    }
    return new {Nome}(value);
  }

  equals(other: {Nome}): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return String(this.value);
  }
}
```

---

## Template: VO composto (mais de um campo)

```typescript
// domain/aggregates/{Agregado}/value-objects/{Nome}.ts

export class {Nome} {
  private constructor(
    readonly campo1: {Tipo1},
    readonly campo2: {Tipo2},
  ) {}

  static create(campo1: {Tipo1}, campo2: {Tipo2}): {Nome} {
    if (/* invariante */) {
      throw new DomainException('Mensagem da regra violada');
    }
    return new {Nome}(campo1, campo2);
  }

  equals(other: {Nome}): boolean {
    return this.campo1 === other.campo1 && this.campo2 === other.campo2;
  }
}
```

---

## Exemplo real — StarContent (conquista STAR)

```typescript
export class StarContent {
  private constructor(
    readonly situation: string,
    readonly task: string,
    readonly action: string,
    readonly result: string,
  ) {}

  static create(situation: string, task: string, action: string, result: string): StarContent {
    const campos = { situation, task, action, result };
    for (const [nome, valor] of Object.entries(campos)) {
      if (!valor || valor.trim().length < 20) {
        throw new DomainException(`Campo STAR '${nome}' deve ter no mínimo 20 caracteres`);
      }
    }
    return new StarContent(situation, task, action, result);
  }

  get isComplete(): boolean {
    return [this.situation, this.task, this.action, this.result]
      .every(campo => campo.trim().length >= 20);
  }

  equals(other: StarContent): boolean {
    return (
      this.situation === other.situation &&
      this.task === other.task &&
      this.action === other.action &&
      this.result === other.result
    );
  }
}
```

---

## Regras de design

1. **Construtor privado** — instância criada apenas via factory estático `create()` ou `generate()`
2. **Imutabilidade** — todos os campos `readonly`, sem setters
3. **Validação no factory** — lança `DomainException` em caso de dado inválido
4. **Zero framework no domínio** — sem imports de NestJS, Prisma ou qualquer biblioteca externa
5. **Igualdade por valor** — método `equals()` sempre presente
6. **Mensagens claras** — erros descrevem a regra violada, não apenas "inválido"

---

## Mapeamento Prisma (responsabilidade exclusiva da infrastructure)

```typescript
// infrastructure/persistence/{Agregado}/{Agregado}RepositoryAdapter.ts

// O adaptador converte entre primitivos do Prisma e VOs do domínio
private toDomain(record: PrismaAchievement): Achievement {
  return Achievement.reconstitute({
    id: AchievementId.from(record.id),
    star: StarContent.create(
      record.star_situation,
      record.star_task,
      record.star_action,
      record.star_result,
    ),
    complexity: record.complexity as Complexity,
  });
}
```

---

## Checklist

- [ ] Construtor `private` — instância apenas via factory estático
- [ ] Todos os campos `readonly`
- [ ] Validação no `create()` com `DomainException` e mensagem descritiva
- [ ] Zero imports de NestJS, Prisma ou bibliotecas externas no domínio
- [ ] Método `equals()` implementado
- [ ] Testes: criação válida, criação inválida (cada invariante), operações de negócio