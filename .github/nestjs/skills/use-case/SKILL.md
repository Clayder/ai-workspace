---
name: use-case
description: >-
  Use ao criar um novo Caso de Uso na camada application seguindo Arquitetura Hexagonal + DDD.
  Ative para termos como: "criar caso de uso", "novo fluxo de negócio",
  "implementar feature", "orquestrar domínio", "criar UseCase", "nova funcionalidade",
  ou qualquer ação que envolva orquestrar entidades de domínio em resposta
  a uma intenção do usuário. Stack: TypeScript + NestJS.
argument-hint: "[nome do caso de uso] [agregado envolvido]"
user-invocable: true
disable-model-invocation: false
---

# Como criar um Caso de Uso — Arquitetura Hexagonal + DDD (TypeScript/NestJS)

> Nomes de agregados, Value Objects e códigos de erro estão definidos em
> `copilot-instructions.md` e em `docs/prd/`. Consulte antes de nomear tipos e códigos.

## Localização dos arquivos

```
application/use-cases/{agregado}/{acao}/
├── {Acao}UseCase.ts
├── {Acao}Input.ts
└── {Acao}Output.ts
```

## Regras inegociáveis

- Sufixos `Input`, `Output`, `UseCase` — nunca `DTO`, `Command`, `Request`, `Response`, `Result`
- `Input` e `Output` são **interfaces**, não classes
- Entidades de domínio **nunca** são retornadas diretamente — sempre via `Output`
- Verificação de **autorização** acontece no UseCase **antes** de qualquer operação de domínio
- Verificação de **limites de tier** acontece no UseCase consultando a `SubscriptionPort`
- Injeção de dependência via `@Injectable()` do NestJS

---

## Template: {Acao}Input.ts

```typescript
// application/use-cases/{agregado}/{acao}/{Acao}Input.ts

export interface {Acao}Input {
  userId: string;
  // demais campos em camelCase com tipos explícitos
  traceId?: string;
}
```

---

## Template: {Acao}Output.ts

```typescript
// application/use-cases/{agregado}/{acao}/{Acao}Output.ts

export interface {Acao}Output {
  id: string;
  created_at: string;   // primitivos — mais fáceis de serializar
  // demais campos
}
```

---

## Template: {Acao}UseCase.ts

```typescript
// application/use-cases/{agregado}/{acao}/{Acao}UseCase.ts

import { Injectable, Inject } from '@nestjs/common';
import { {Acao}Input } from './{Acao}Input';
import { {Acao}Output } from './{Acao}Output';
import { {Agregado}Repository } from '../../domain/aggregates/{Agregado}/repositories/{Agregado}Repository';
import { {AGREGADO}_REPOSITORY } from '../../{Agregado}.tokens';
import { {Entidade}NotFoundException } from '../../domain/aggregates/{Agregado}/exceptions/{Entidade}NotFoundException';
import { {Entidade}AccessDeniedException } from '../../domain/aggregates/{Agregado}/exceptions/{Entidade}AccessDeniedException';

@Injectable()
export class {Acao}UseCase {
  constructor(
    @Inject({AGREGADO}_REPOSITORY)
    private readonly repository: {Agregado}Repository,
  ) {}

  async execute(input: {Acao}Input): Promise<{Acao}Output> {
    // 1. Buscar recurso
    const entidade = await this.repository.findById(input.entidadeId);
    if (!entidade) {
      throw new {Entidade}NotFoundException(input.entidadeId);    // → HTTP 404
    }

    // 2. Verificar autorização ANTES de qualquer operação de domínio
    if (entidade.userId !== input.userId) {
      throw new {Entidade}AccessDeniedException(input.entidadeId); // → HTTP 403
    }

    // 3. Verificar limites de tier (quando aplicável)
    // const subscription = await this.subscriptionPort.findByUserId(input.userId);
    // if (!subscription.features.{feature}) throw new FeatureNotAvailableException();

    // 4. Executar lógica de domínio
    entidade.realizarOperacao(/* parâmetros */);

    // 5. Persistir
    const salvo = await this.repository.save(entidade);

    // 6. Retornar Output — NUNCA a entidade diretamente
    return {
      id: salvo.id.value,
      created_at: salvo.createdAt.toISOString(),
    };
  }
}
```

---

## Exemplo real — RegisterAchievementUseCase

```typescript
@Injectable()
export class RegisterAchievementUseCase {
  constructor(
    @Inject(CAREER_COMPANY_REPOSITORY)
    private readonly careerCompanyRepository: CareerCompanyRepository,
    @Inject(SUBSCRIPTION_PORT)
    private readonly subscriptionPort: SubscriptionPort,
  ) {}

  async execute(input: RegisterAchievementInput): Promise<RegisterAchievementOutput> {
    // 1. Verificar empresa e cargo
    const company = await this.careerCompanyRepository.findById(input.companyId);
    if (!company) throw new CareerCompanyNotFoundException(input.companyId);
    if (company.userId !== input.userId) throw new CareerCompanyAccessDeniedException(input.companyId);

    // 2. Verificar se empresa está ativa
    if (company.status === CareerCompanyStatus.CONCLUDED) {
      throw new CareerCompanyConcludedException(input.companyId);  // → HTTP 422
    }

    // 3. Verificar limite de tier
    const subscription = await this.subscriptionPort.findByUserId(input.userId);
    if (subscription.limits.achievements_limit !== null) {
      const count = await this.careerCompanyRepository.countAchievementsByUser(input.userId);
      if (count >= subscription.limits.achievements_limit) {
        throw new AchievementLimitReachedException();               // → HTTP 403
      }
    }

    // 4. Lógica de domínio
    const role = company.findRole(input.roleId);
    if (!role) throw new RoleNotFoundException(input.roleId);

    const achievement = Achievement.create({
      roleId: RoleId.from(input.roleId),
      complexity: input.complexity,
      impactCategory: input.impactCategory,
      star: StarContent.create(input.situation, input.task, input.action, input.result),
    });

    role.addAchievement(achievement);

    // 5. Persistir
    await this.careerCompanyRepository.save(company);

    return {
      id: achievement.id.value,
      registered_at: achievement.registeredAt.toISOString(),
      complexity: achievement.complexity,
      impact_category: achievement.impactCategory,
    };
  }
}
```

---

## Exceções de domínio esperadas

```typescript
// → HTTP 404
export class {Entidade}NotFoundException extends ApplicationException {
  constructor(id: string) {
    super(`{Entidade} não encontrada: ${id}`, '{PREFIXO}_NNN', 404);
  }
}

// → HTTP 403
export class {Entidade}AccessDeniedException extends ApplicationException {
  constructor(id: string) {
    super(`Acesso negado: ${id}`, '{PREFIXO}_NNN', 403);
  }
}

// → HTTP 422
export class {Regra}Exception extends DomainException {
  constructor() {
    super('Descrição clara da regra violada', '{PREFIXO}_NNN');
  }
}
```

---

## Domain Events

Se o caso de uso emite um Domain Event, o mecanismo de entrega (síncrono in-process ou assíncrono via fila) **deve estar definido na especificação** antes de qualquer implementação.

> ⚠️ **Nunca assuma o mecanismo.** Se a especificação não indicar como o evento deve ser entregue, pare e solicite essa definição no refinamento. Implementar o mecanismo errado pode introduzir inconsistências silenciosas difíceis de reverter.

Quando o mecanismo estiver definido na especificação, o padrão de publicação é:

```typescript
// 1. Persiste o agregado
await this.repository.save(entidade);

// 2. Coleta e despacha os eventos acumulados no agregado
// O mecanismo concreto (EventEmitter2, BullMQ, etc.)
// é determinado pela especificação do caso de uso
await this.eventDispatcher.dispatch(entidade.pullEvents());
```

---

## Registro no módulo NestJS

```typescript
// {Contexto}Module.ts
@Module({
  providers: [
    {Acao}UseCase,
    {
      provide: {AGREGADO}_REPOSITORY,
      useClass: {Agregado}RepositoryAdapter,
    },
  ],
  exports: [{Acao}UseCase],
})
export class {Contexto}Module {}
```

---

## Checklist

- [ ] Arquivos em `application/use-cases/{agregado}/{acao}/`
- [ ] Sufixos `Input`, `Output`, `UseCase` presentes
- [ ] `Input` e `Output` são interfaces (não classes)
- [ ] `@Injectable()` no UseCase
- [ ] Autorização verificada antes de qualquer operação de domínio
- [ ] Limites de tier verificados quando a feature tem restrição por plano
- [ ] Nenhuma entidade de domínio retornada diretamente
- [ ] Exceções com código no formato do projeto
- [ ] Se o caso de uso emite Domain Events: mecanismo de entrega está definido na especificação
- [ ] Teste unitário cobrindo: happy path, not found, acesso negado, limite de tier