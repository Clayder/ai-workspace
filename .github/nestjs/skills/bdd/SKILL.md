---
name: bdd
description: >-
  Use ao criar cenários BDD com Cucumber seguindo Arquitetura Hexagonal com TypeScript e NestJS.
  Ative para termos como: "criar feature", "cenário BDD", "Cucumber", "Gherkin",
  "fluxo crítico", "teste end-to-end", "criar .feature", "escrever cenário",
  "dado", "quando", "então", "step definition", "teste de integração BDD",
  ou qualquer tarefa que envolva especificação de comportamento em linguagem natural.
  Stack: TypeScript + NestJS + Cucumber + TestContainers.
argument-hint: "[fluxo de negócio a especificar]"
user-invocable: true
disable-model-invocation: false
---

# Como criar cenários BDD com Cucumber — Arquitetura Hexagonal (TypeScript/NestJS)

> Consulte `docs/prd/fluxos-de-usuario.md` para os fluxos críticos já mapeados
> e os agregados envolvidos em cada cenário.

## Quando usar BDD

Apenas para **fluxos críticos de ponta a ponta** que envolvem múltiplas camadas.
Módulos simples são cobertos por testes unitários no `application/`.
Não crie `.feature` para cada endpoint individual.

**Fluxos candidatos a BDD no uMeritor:**
- Registro de conquista STAR (inclui verificação de tier e empresa ativa)
- Encerramento de vínculo profissional
- Geração de pauta de reunião 1:1
- Análise de Job Fit
- Configuração de página pública

---

## Localização dos arquivos

```
test/integration/
├── features/{agregado}/
│   └── {fluxo_de_negocio}.feature       # um .feature por fluxo, não por endpoint
└── steps/
    └── {Agregado}Steps.ts
```

---

## Template: arquivo .feature (Gherkin em português — obrigatório)

```gherkin
# language: pt

Funcionalidade: {Nome do fluxo de negócio}
  Como {persona}
  Quero {ação}
  Para {objetivo de negócio}

  Contexto:
    Dado que estou autenticado como "{email}" com plano "{tier}"
    E tenho um vínculo profissional ativo na empresa "{empresa}"

  Cenário: {Caminho feliz}
    Quando {ação do usuário com dados concretos de negócio}
    Então {resultado esperado no domínio}

  Cenário: {Caso de erro — limite de tier}
    Dado que já possuo {N} conquistas cadastradas
    Quando tento registrar mais uma conquista
    Então devo receber o erro "{ACHIEVEMENT_LIMIT_REACHED}" com status 403

  Cenário: {Caso de erro — regra de negócio}
    Dado que o vínculo profissional está encerrado
    Quando tento registrar uma conquista
    Então devo receber o erro "{CAREER_COMPANY_CONCLUDED}" com status 422
```

---

## Template: Step Definitions

```typescript
// test/integration/steps/{Agregado}Steps.ts

import { Given, When, Then, Before } from '@cucumber/cucumber';
import { HttpStatus } from '@nestjs/common';
import * as request from 'supertest';
import { getTestApp } from '../support/TestApp';

let authToken: string;
let lastResponse: request.Response;
let companyId: string;

Before(async () => {
  // Limpa estado entre cenários se necessário
});

Given('que estou autenticado como {string} com plano {string}', async (email: string, tier: string) => {
  const app = getTestApp();
  const response = await request(app.getHttpServer())
    .post('/api/v1/auth/login')
    .send({ email, password: 'senha-de-teste' });

  authToken = response.body.access_token;
});

Given('tenho um vínculo profissional ativo na empresa {string}', async (empresa: string) => {
  const app = getTestApp();
  const response = await request(app.getHttpServer())
    .post('/api/v1/career-companies')
    .set('Authorization', `Bearer ${authToken}`)
    .send({
      name: empresa,
      started_at: '2024-01-01T00:00:00Z',
      initial_role: { title: 'Engenheiro Pleno', started_at: '2024-01-01T00:00:00Z' },
    });

  companyId = response.body.id;
});

When('registro uma conquista com complexidade {string} e categoria {string}', async (
  complexity: string,
  category: string,
) => {
  const app = getTestApp();
  const company = await getCompanyWithRole(authToken);

  lastResponse = await request(app.getHttpServer())
    .post(`/api/v1/career-companies/${company.id}/roles/${company.roles[0].id}/achievements`)
    .set('Authorization', `Bearer ${authToken}`)
    .send({
      complexity,
      impact_category: category,
      star: {
        situation: 'O sistema de pagamentos estava com latência crítica de 5 segundos',
        task: 'Identificar e resolver o gargalo de performance no serviço de checkout',
        action: 'Implementei cache distribuído com Redis e otimizei as queries N+1',
        result: 'Latência reduziu de 5s para 200ms, impactando 50k transações por dia',
      },
    });
});

Then('a conquista deve ser registrada com sucesso', () => {
  expect(lastResponse.status).toBe(HttpStatus.CREATED);
  expect(lastResponse.body).toHaveProperty('id');
  expect(lastResponse.body).toHaveProperty('registered_at');
});

Then('devo receber o erro {string} com status {int}', (errorCode: string, statusCode: number) => {
  expect(lastResponse.status).toBe(statusCode);
  expect(lastResponse.body.error_code).toBe(errorCode);
});
```

---

## Configuração do ambiente de integração

```typescript
// test/integration/support/TestApp.ts

import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../../../src/AppModule';
import { GlobalExceptionFilter } from '../../../src/shared/filters/GlobalExceptionFilter';
import { ValidationPipe } from '@nestjs/common';

let app: INestApplication;

export async function initTestApp(): Promise<void> {
  const moduleRef = await Test.createTestingModule({
    imports: [AppModule],
  }).compile();

  app = moduleRef.createNestApplication();
  app.useGlobalFilters(new GlobalExceptionFilter());
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  await app.init();
}

export function getTestApp(): INestApplication {
  return app;
}
```

```typescript
// test/integration/support/TestContainersSetup.ts
// PostgreSQL real via TestContainers — isolado por suite

import { PostgreSqlContainer } from '@testcontainers/postgresql';

export async function startPostgres(): Promise<string> {
  const container = await new PostgreSqlContainer()
    .withDatabase('umeritor_test')
    .withUsername('test')
    .withPassword('test')
    .start();

  return container.getConnectionUri();
}
```

---

## Boas práticas

| Faça | Evite |
|---|---|
| Cenários independentes entre si | Cenários que dependem da ordem de execução |
| Dados em linguagem de negócio | IDs técnicos, UUIDs brutos, centavos |
| `Contexto` para pré-condições compartilhadas | Repetir pré-condições em cada cenário |
| Um `.feature` por fluxo de negócio | Um `.feature` por endpoint da API |
| Caminho feliz + pelo menos 1 cenário de erro | Cenários apenas de sucesso |
| Dados realistas do domínio uMeritor | Dados genéricos como `"teste"` ou `"foo"` |

---

## Checklist

- [ ] `.feature` em `test/integration/features/{agregado}/`
- [ ] `# language: pt` no topo do arquivo
- [ ] `Funcionalidade` com persona e objetivo de negócio
- [ ] `Contexto` com pré-condições compartilhadas
- [ ] Steps em português: `Dado`, `Quando`, `Então`
- [ ] Dados em linguagem de negócio (não técnica)
- [ ] Caminho feliz + pelo menos 1 cenário de erro de tier + 1 de regra de negócio
- [ ] Step definitions em `test/integration/steps/`
- [ ] Testes rodando com PostgreSQL real via TestContainers
- [ ] Cenários independentes — sem dependência de ordem de execução