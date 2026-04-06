# Instruções para o Copilot — uMeritor Career API

> **Regra geral:** todos os arquivos `.md` do repositório devem ser escritos em **português (Brasil)**.
> **Diagramas:** todo diagrama criado em qualquer arquivo `.md` deve usar **Mermaid** — nunca ASCII art, imagens externas ou outras ferramentas.

---

## Stack de Tecnologia

| Camada | Tecnologia |
|---|---|
| Framework | NestJS 11 + Fastify 5 |
| Linguagem | TypeScript 6 |
| Banco de dados | PostgreSQL via Prisma 7.5 |
| Autenticação | JWT Bearer + Bcrypt |
| Validação | class-validator + class-transformer |
| Documentação | Swagger/OpenAPI (`@nestjs/swagger`) |
| Testes unitários | Jest 30 |
| Testes de integração | Cucumber BDD + TestContainers |

---

## Comandos

```bash
# Desenvolvimento
npm run start:dev                                                   # Modo watch com hot reload
npm run lint:fix                                                    # Corrige lint automaticamente
npm run format                                                      # Formata com Prettier

# Build
npm run build

# Testes
npm run test                                                        # Todos os testes unitários
npm run test -- --testPathPattern=src/modules/career               # Módulo específico
npm run test -- --testNamePattern="deve registrar conquista"       # Teste pelo nome
npm run test:cov                                                    # Cobertura (mínimo 90%)
npm run test:integrationTest                                        # Cucumber BDD + TestContainers

# Banco de dados
npx prisma migrate dev                                             # Cria e aplica migration (interativo)
npx prisma migrate deploy                                          # Aplica migrations (CI/prod)
npx prisma generate                                                # Regenera o cliente Prisma
npx prisma studio                                                  # Interface visual do banco
```

---

## Arquitetura

**Arquitetura Hexagonal + DDD — Monólito Modular.**

Bounded Contexts em `src/modules/`. Cada módulo é completamente isolado — **nunca importe classes internas de outro módulo diretamente**. A comunicação entre módulos ocorre exclusivamente via interfaces de Port ou Domain Events.

### Estrutura de Camadas por Módulo

```
src/modules/{contexto}/
├── domain/                          # Lógica de negócio pura — SEM imports de NestJS ou Prisma
│   ├── aggregates/{Agregado}/
│   │   ├── {Agregado}.ts            # Aggregate root
│   │   ├── entities/
│   │   ├── value-objects/
│   │   ├── exceptions/              # DomainException e subclasses
│   │   └── repositories/            # Interfaces de Port (abstratas)
│   ├── events/                      # Domain events
│   └── services/                    # Serviços de domínio (lógica entre agregados)
├── application/                     # Orquestração de casos de uso — importa apenas domain
│   ├── ports/                       # Interfaces de Port de saída (ex: PasswordHasherPort)
│   └── use-cases/{agregado}/{acao}/
│       ├── {Acao}UseCase.ts
│       ├── {Acao}Input.ts           # Interface de entrada (não classe)
│       └── {Acao}Output.ts          # Interface de saída (não classe)
├── presentation/                    # Camada HTTP — importa apenas application
│   └── controllers/{agregado}/
│       ├── {Recurso}Controller.ts   # Decorators NestJS + Swagger + delegação ao UseCase
│       ├── {Acao}Request.ts         # DTOs com class-validator
│       └── {Acao}Response.ts        # Classes de resposta em snake_case
└── infrastructure/                  # Adaptadores técnicos — importa domain + application
    └── persistence/{agregado}/
        ├── {Agregado}PrismaEntity.ts
        └── {Agregado}RepositoryAdapter.ts
```

### Regra de Dependência (inegociável)

```
domain  ←  application  ←  presentation
infrastructure  →  domain + application   (nunca presentation)
```

### Comunicação entre Módulos

| Padrão | Quando usar |
|---|---|
| **Port síncrono** | O resultado bloqueia o fluxo atual |
| **Domain Event assíncrono** | Reação a algo que já aconteceu |
| **ACL (Anti-Corruption Layer)** | Integração com fonte externa (ex: WhatsApp) |

**Exemplo prático:** o módulo `Career` precisa verificar o tier do usuário → declara `AccessPolicyPort` em `application/ports/`; o módulo `AccessControl` fornece o adaptador concreto.

---

## Domínio

> A documentação completa de DDD está em **`docs/ddd/`**. Esta seção contém o essencial que o Copilot precisa em toda interação. Consulte os arquivos fonte para detalhes de invariantes, diagramas de estado e fluxos completos.

### Bounded Contexts e Responsabilidades

| Contexto | Tipo | Responsabilidade |
|---|---|---|
| **Career** ⭐ | Core Domain | Memória profissional — vínculos, cargos, conquistas, impedimentos, feedbacks |
| **Access Control** | Supporting | Assinatura, tiers e feature gates |
| **Intelligence** | Supporting | Geração de insights — pautas, planos, currículo, job fit, entrevista |
| **Profile** | Supporting | Importação e armazenamento de dados do LinkedIn |
| **Identity** | Generic | Autenticação JWT |
| **Capture** | Generic | Integração com WhatsApp via ACL |
| **Portfolio** | Generic | Página pública do profissional |

Cada contexto é um módulo em `src/modules/`. A comunicação entre contextos ocorre **exclusivamente** via:
- `AccessPolicyPort` — para verificação de tier (síncrono)
- Domain Events — para reações assíncronas entre contextos
- ACL (Anti-Corruption Layer) — para integrações externas (WhatsApp, LinkedIn, AI)

### Aggregate Roots por Contexto

| Aggregate Root | Contexto | Entidades internas |
|---|---|---|
| `CareerCompany` | Career | `Role` |
| `Achievement` | Career | — |
| `Impediment` | Career | — |
| `Feedback` | Career | `ActionItem` |
| `Subscription` | Access Control | — |
| `MeetingAgenda` | Intelligence | `AgendaItem` (VO) |
| `PromotionPlan` | Intelligence | `Gap` |
| `Resume` | Intelligence | — |
| `JobFit` | Intelligence | — |
| `InterviewPrep` | Intelligence | `InterviewQuestion` |
| `BaseProfile` | Profile | — |
| `PublicPage` | Portfolio | — |
| `Capture` | Capture | — |
| `User` | Identity | — |

> Regra inegociável: toda mutação de estado passa pelo Aggregate Root. Entidades internas só são acessadas através do root. Contextos externos referenciam agregados apenas por ID.

### Linguagem Ubíqua — Termos Obrigatórios

Use estes termos **de forma consistente** no código, testes, commits e comentários. Nunca use os termos da coluna "Evitar".

| ✅ Usar no código | ❌ Evitar | Significado |
|---|---|---|
| `careerCompany` | `company`, `job`, `experience` | Vínculo profissional com uma empresa |
| `role` | `position`, `post`, `job` | Cargo dentro de um vínculo |
| `achievement` | `story`, `memory`, `record` | Conquista no formato STAR |
| `impediment` | `problem`, `issue`, `blocker` | Bloqueio documentado com lição aprendida |
| `feedback` | `comment`, `review` | Elogio (Praise) ou crítica (Criticism) recebida |
| `actionItem` | `task`, `todo` | Ação derivada de um feedback |
| `promotionPlan` | `plan`, `roadmap` | Análise de gaps para promoção |
| `meetingAgenda` | `agenda`, `plan` | Pauta de reunião 1:1 gerada automaticamente |
| `jobFit` | `match`, `score` | Análise de compatibilidade com vaga |
| `gap` | `missing`, `lack` | Lacuna identificada no plano de promoção |
| `userId` | `user` (como objeto) | No domínio, usuário é referenciado apenas por ID |
| `guardNotConcluded()` | `checkStatus()`, `validateStatus()` | Invariante que rejeita mutação em vínculo encerrado |
| `star` | `content`, `fields` | Value Object com situation, task, action, result |
| `conclude()` | `close()`, `finish()`, `end()` | Encerrar um vínculo profissional |
| `deliver()` | `send()`, `complete()` | Marcar pauta de reunião como entregue |
| `register()` | `create()`, `add()` | Factory semântico de Achievement e Impediment |
| `receive()` | `create()`, `add()` | Factory semântico de Feedback |

### Invariantes de Negócio Críticas

O Copilot deve conhecer estas invariantes ao gerar qualquer código que envolva os agregados:

| ID | Regra | Erro | HTTP |
|---|---|---|---|
| CC-003 | `CareerCompany.CONCLUDED` não aceita novos registros | `CAREER_COMPANY_CONCLUDED` | 422 |
| CC-005 | Freemium: máximo 1 empresa `ACTIVE` | `CAREER_COMPANY_LIMIT_REACHED` | 403 |
| CC-006 | `CareerCompany` deve ter pelo menos 1 `Role` | — | 422 |
| ACH-001 | Freemium: máximo 3 conquistas total | `ACHIEVEMENT_LIMIT_REACHED` | 403 |
| ACH-002 | Campos STAR: mínimo 20 caracteres cada | `CO_001` | 400 |
| ROLE-005 | Não pode deletar o único `Role` do vínculo | — | 422 |
| FB-001 | `Feedback` disponível apenas para Pro e Pro+ | `FEATURE_NOT_AVAILABLE` | 403 |
| PUB-002 | `Slug` da página pública deve ser único | `SLUG_ALREADY_TAKEN` | 409 |

### Domain Events — Mapa de Publishers e Consumidores

| Evento | Publisher | Consumidores |
|---|---|---|
| `CareerCompanyCreated` | `CareerCompany` | Access Control (incrementa contador) |
| `CareerCompanyConcluded` | `CareerCompany` | Access Control (decrementa contador) |
| `AchievementRegistered` | `Achievement` | Access Control, Intelligence, Portfolio |
| `AchievementDeleted` | `Achievement` | Access Control (decrementa contador) |
| `ImpedimentRegistered` | `Impediment` | Intelligence |
| `FeedbackReceived` | `Feedback` | — |
| `CaptureProcessed` | `Capture` | Career BC (cria Achievement ou Impediment) |
| `BaseProfileImportCompleted` | `BaseProfile` | Intelligence |

**Regra de publicação:** o UseCase persiste o agregado e depois chama `dispatch(aggregate.pullEvents())`. Handlers não lançam exceção para o publisher.

> ⚠️ **O mecanismo de entrega de cada evento (síncrono in-process ou assíncrono via fila) é uma decisão técnica definida durante o refinamento do caso de uso**, não globalmente. A especificação de cada UseCase deve indicar explicitamente como o evento será entregue e por quê. Não assuma nem implemente um mecanismo sem que esteja especificado.

### Value Objects — Referência Rápida

| Value Object | Contexto | Regra de validação |
|---|---|---|
| `Star` | Career | 4 campos, mínimo 20 chars cada |
| `CompanyName` | Career | Não vazio |
| `RoleTitle` | Career | Não vazio |
| `EvidenceLink` | Career | URL válida |
| `Slug` | Portfolio | Padrão `[a-z0-9-]{3,50}`, único no sistema |
| `UserId` | Shared | UUID v4 válido |
| `Email` | Identity | Formato de e-mail válido |
| `SubscriptionLimits` | Access Control | Valores ≥ 0 ou `null` (ilimitado) |
| `SubscriptionFeatures` | Access Control | Flags booleanas por feature |

Enums do domínio — use **sempre os valores literais** abaixo, nunca strings avulsas:

```typescript
// Career
Complexity:      LOW | MEDIUM | HIGH | CRITICAL
ImpactCategory:  EFFICIENCY | LEADERSHIP | RISK | INNOVATION | QUALITY
FeedbackType:    PRAISE | CRITICISM
CriticismStatus: PENDING | IN_PROGRESS | RESOLVED
ActionItemStatus: PENDING | COMPLETED
CaptureChannel:  WEB | WHATSAPP

// Access Control
SubscriptionTier: FREEMIUM | PRO | PRO_PLUS

// Intelligence
TargetRole:  JUNIOR | PLENO | SENIOR | STAFF | PRINCIPAL | MANAGER | DIRECTOR
GapType:     COMPLEXITY | CATEGORY | EVIDENCE
GapSeverity: LOW | MEDIUM | HIGH | CRITICAL

// Portfolio / Profile
PublicPageStatus: ACTIVE | INACTIVE
ImportStatus:     PENDING | COMPLETED | FAILED
```

---

## Skills Disponíveis

O Copilot deve consultar as skills abaixo antes de implementar qualquer artefato correspondente. Cada skill contém templates, regras inegociáveis e checklists.

| Skill | Quando consultar |
|---|---|
| `value-object` | Criar Value Object ou tipo de domínio tipado |
| `use-case` | Criar caso de uso na camada application |
| `controller` | Criar controller REST na camada presentation |
| `error-handling` | Criar exceções de domínio ou tratar erros HTTP |
| `prisma-migration` | Criar ou alterar schema do banco via Prisma |
| `bdd` | Criar cenários Cucumber ou testes de integração |

---

## Convenções de TypeScript

### Tipagem forte — sem `any`

```typescript
// ❌ Proibido
function processar(dados: any): any { ... }

// ✅ Correto
function processar(dados: RegistrarConquistaInput): RegistrarConquistaOutput { ... }
```

Use `unknown` quando o tipo for genuinamente desconhecido e faça narrowing explícito. Generics, interfaces e union types para expressar contratos corretos.

### Nomenclatura

| Artefato | Padrão | Exemplo |
|---|---|---|
| Aggregate Root | `{Nome}.ts` | `Achievement.ts` |
| Interface de repositório | `{Nome}Repository.ts` | `AchievementRepository.ts` |
| Token de injeção do repositório | `{NOME}_REPOSITORY` | `ACHIEVEMENT_REPOSITORY` |
| Adaptador de repositório | `{Nome}RepositoryAdapter.ts` | `AchievementRepositoryAdapter.ts` |
| Interface de Port | `{Capacidade}Port.ts` | `PasswordHasherPort.ts` |
| Token de injeção do Port | `{CAPACIDADE}_PORT` | `PASSWORD_HASHER_PORT` |
| Adaptador de Port | `{Capacidade}Adapter.ts` | `BcryptPasswordHasherAdapter.ts` |
| Caso de uso | `{Acao}UseCase.ts` | `RegisterAchievementUseCase.ts` |
| Domain event | `{PassadoPerfeito}.ts` | `AchievementRegistered.ts` |
| Exceção de domínio | `{Cenario}Exception.ts` | `AchievementLimitReachedException.ts` |
| Arquivo de enums | `{Nome}Enums.ts` | `AchievementEnums.ts` |
| Prisma Entity | `{Nome}PrismaEntity.ts` | `UserPrismaEntity.ts` |

---

## Convenções de DTOs

### Request (presentation/)

Use `class-validator`, `@ApiProperty` do Swagger, descrições em português:

```typescript
export class RegisterAchievementRequest {
  @ApiProperty({ description: 'Complexidade da conquista', example: 'HIGH' })
  @IsEnum(Complexity)
  complexity!: Complexity;

  @ApiProperty({ description: 'Situação vivenciada (mín. 20 caracteres)', example: 'O sistema de pagamentos estava com latência crítica...' })
  @IsString()
  @MinLength(20)
  situation!: string;
}
```

### Response (presentation/)

Classe simples, campos em **snake_case**, sem decorators de validação:

```typescript
export class AchievementResponse {
  id!: string;
  impact_category!: string;   // snake_case — sempre
  registered_at!: string;
}
```

### Input/Output (application/)

Interfaces TypeScript simples (não classes), camelCase:

```typescript
export interface RegisterAchievementInput {
  userId: string;
  companyId: string;
  complexity: Complexity;
  situation: string;
  traceId?: string;
}
```

---

## Tratamento de Erros (resumo)

| Tipo | HTTP | Origem |
|---|---|---|
| Validação de input inválido | 400 | `class-validator` via `ValidationPipe` |
| Regra de negócio violada | 422 | `DomainException` |
| Recurso inexistente | 404 | `ApplicationException` |
| Sem permissão | 403 | `ApplicationException` |
| Token ausente/expirado | 401 | Guard JWT |

Um `GlobalExceptionFilter` em `src/shared/` é o **único** lugar que constrói respostas de erro. Controllers nunca montam `ErrorResponse` diretamente.

> Consulte a skill `error-handling` para templates completos.

---

## Testes

- **Unitários:** colocalizados como `{arquivo}.spec.ts` — testam `domain/` e `application/`
- **Integração:** em `test/integration/` — arquivos `.feature` em português (Gherkin), step definitions em TypeScript, PostgreSQL real via TestContainers
- **Cobertura mínima:** 90% em linhas, funções, branches e statements
- `infrastructure/` e `presentation/` são excluídas do `collectCoverageFrom`

---

## Convenções de API REST

- Prefixo de todas as rotas: `api/v1` (variável `API_PREFIX`)
- JWT Bearer obrigatório em todas as rotas, exceto as marcadas com `@Public()`
- Extrair o usuário autenticado com `@CurrentUser() userId: string`
- Campos de resposta sempre em **snake_case**
- Swagger disponível em `/api/v1/docs`
- Paginação padrão: `?page=0&size=10` → `{ items, page, size, total }`
- `X-Trace-Id` (UUID) em toda requisição para rastreabilidade

---

## Shared Kernel (`src/shared/`)

Classes base disponíveis — **nunca duplique, sempre importe de shared**:

| Classe/Interface | Descrição |
|---|---|
| `Entity<TId>` | Base para entidades com identidade |
| `ValueObject<T>` | Base para value objects com igualdade por valor |
| `AggregateRoot<TId>` | Estende Entity, adiciona suporte a Domain Events |
| `DomainEvent` | Base para eventos de domínio |
| `DomainException` | Base para exceções de negócio (→ HTTP 422) |
| `ApplicationException` | Exceção de caso de uso com status customizável |
| `UserId` | Value Object do identificador de usuário |

---

## Ambiente e Banco de Dados

```bash
# Suba o banco local
docker-compose up -d

# Configure variáveis
cp .env.example .env
```

Variáveis principais: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `APP_PORT`.

**Schema sempre via migrations Prisma** — nunca altere o banco diretamente.

> Consulte a skill `prisma-migration` para templates e regras de migration.

---

## Documentação de Produto

Toda documentação de negócio e regras de produto está em **`docs/prd/`**. Consulte antes de implementar qualquer nova funcionalidade — especialmente `regras-de-negocio.md` e o módulo correspondente.