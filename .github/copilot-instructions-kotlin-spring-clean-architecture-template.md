# Instruções do Copilot — {{SERVICE_NAME}}

<!--
  TEMPLATE — Clean Architecture + DDD + Kotlin + Spring Boot
  =============================================================
  Preencha todos os campos marcados com {{...}} antes de usar.
  Remova este bloco de comentário após preencher.

  CHECKLIST de preenchimento:
  [ ] {{SERVICE_NAME}}         → nome do serviço (ex: expense-service)
  [ ] {{PLATFORM_NAME}}        → nome da plataforma (ex: FinanceFlow)
  [ ] {{SERVICE_RESPONSIBILITY}} → responsabilidade em uma linha
  [ ] {{OUT_OF_SCOPE}}         → o que este serviço NÃO faz
  [ ] {{OTHER_SERVICES}}       → tabela dos outros serviços do ecossistema
  [ ] {{ERROR_PREFIX}}         → prefixo de 2 letras para códigos de erro (ex: EX, WA, AU)
  [ ] {{VERSION_SERIES}}       → série de versão (ex: 1.x.y)
  [ ] Revise a seção "Stack" se alguma tecnologia for diferente
-->

---

## Visão Geral

O **{{SERVICE_NAME}}** é um microserviço da plataforma **{{PLATFORM_NAME}}**
responsável por: {{SERVICE_RESPONSIBILITY}}.

**Fora do escopo deste serviço:** {{OUT_OF_SCOPE}}.

### Ecossistema de serviços

{{OTHER_SERVICES}}
<!--
  Exemplo:
  | Serviço | Responsabilidade | Status |
  |---|---|---|
  | `expense-service` | Gastos | v1.0 — em desenvolvimento |
  | `income-service` | Receitas | Planejado |
  | `report-service` | Relatórios consolidados | Planejado |
-->

- Cada serviço possui **seu próprio banco de dados** — sem compartilhamento
- Comunicação entre serviços: **HTTP/REST** síncrono
- Autenticação **própria** por serviço na v1.0

---

## Stack

- **Linguagem:** Kotlin — nunca `Any`, nunca `!!`
- **Framework:** Spring Boot
- **Build:** Gradle (Kotlin DSL)
- **Banco:** PostgreSQL — migrations via **Flyway** exclusivamente
- **ORM:** JPA / Hibernate — `FetchType.LAZY` obrigatório, `EAGER` proibido
- **Auth:** JWT Bearer Token (24h) + refresh token — bcrypt mín. 12 rounds
- **Testes:** JUnit 5, Testcontainers, Cucumber (fluxos críticos)
- **Qualidade:** ktlint (CI bloqueia merge), ArchUnit obrigatório
- **Infra local:** Docker + Docker Compose
- **Documentação:** OpenAPI / Swagger

> Para código que envolva bibliotecas ou frameworks, usar **Context7 (MCP)**.

---

## Arquitetura: Clean Architecture + DDD

| Camada | Responsabilidade | Dependências permitidas |
|---|---|---|
| `domain` | Entidades, agregados, Value Objects, regras de negócio | **Nenhuma** |
| `application` | Casos de uso, orquestração, portas outbound | `domain` |
| `presentation` | Controllers REST, DTOs, OpenAPI | `application` |
| `infrastructure` | JPA, adapters, integrações externas | `application`, `domain` |

### Regras invioláveis

- `domain` nunca importa framework (Spring, JPA, etc.)
- Regras de negócio residem **exclusivamente** no `domain`
- `@Service` e `@Component` proibidos no `domain` — use `@DomainService` (registrado via `DomainServiceScanConfig` na infra)
- Tipos nativos (`String`, `UUID`, `Int`, `Long`) **nunca** como campos de negócio em entidades, VOs ou agregados — sempre `value class`
- Módulos nunca importam classes internas uns dos outros — apenas contratos (portas, eventos, comandos)

---

## Skills disponíveis

As skills complementam estas instruções com receitas detalhadas de implementação.
O Copilot as carrega automaticamente quando relevante, ou invoque com `/nome`:

| Skill | Quando é carregada |
|---|---|
| `/domain-language` | **Consulte sempre** — nomes de domínio, codes de erro, tabelas, exemplos |
| `/use-case` | Criar Use Cases na camada `application` |
| `/controller` | Criar Controllers REST e DTOs na camada `presentation` |
| `/value-object` | Criar `value class` e tipos de domínio |
| `/migration` | Criar migrations Flyway |
| `/test-builder` | Criar testes unitários, de integração e builders |
| `/error-handling` | Criar exceções de domínio e `GlobalExceptionHandler` |
| `/openapi` | Documentar endpoints com OpenAPI/Swagger |
| `/bdd` | Criar cenários Cucumber para fluxos críticos |

---

## API

- **Base path:** `/api/v1/`
- **Auth:** `Authorization: Bearer <token>`
- **Respostas:** JSON — campos **sempre em inglês** camelCase
- **Prefixo de erro deste serviço:** `{{ERROR_PREFIX}}` (ex: `{{ERROR_PREFIX}}_001`)
- **Códigos comuns:** `CO_001` (400 entrada inválida), `CO_002` (400 body inválido), `CO_999` (500)

### Endpoints públicos (sem autenticação)

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/forgot-password`

### Padrão de erros HTTP (NÃO NEGOCIÁVEL)

| Código | Quando usar |
|---|---|
| `400` | Validação de **entrada** — campo inválido, ausente ou formato errado |
| `422` | Violação de **regra de negócio** |
| `403` | Recurso existe mas usuário sem permissão — **nunca `404`** para mascarar existência |
| `404` | Recurso genuinamente inexistente |

- `GlobalExceptionHandler` é o **único** responsável por construir `ErrorResponse`
- Respostas **nunca** expõem stack trace ou mensagem de exceção Java/Kotlin

---

## Segurança

- Todo endpoint requer JWT, exceto os 3 endpoints públicos de auth
- Autorização verificada **no use case**, antes de qualquer operação de domínio
- Controller **nunca** verifica permissão
- Isolamento de dados: todo acesso filtrado por `user_id` do usuário autenticado

---

## Banco de dados

- Migrations exclusivamente via **Flyway**
- **Soft delete obrigatório** — hard delete proibido — coluna `deleted_at TIMESTAMP NULL` em todas as tabelas de negócio
- Filtro `deleted_at IS NULL` é responsabilidade exclusiva da camada `infrastructure`
- Valores monetários sempre em **centavos** (`BIGINT`) — nunca `FLOAT` ou `DECIMAL`

---

## Qualidade e testes

- **Cobertura mínima:** `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`
- **Builders de teste obrigatórios** — padrão `object` Kotlin, em `src/testFixtures/kotlin/testbuilders/`
- Testes de unidade em `src/test/kotlin/`, integração em `src/integrationTest/kotlin/`
- **Testcontainers** para testes com banco real
- **BDD com Cucumber** apenas para fluxos críticos — cenários `.feature` em PT-BR
- **ArchUnit** obrigatório: `domain` sem Spring, `infrastructure` sem importar `presentation`
- **ktlint** obrigatório — CI bloqueia merge se lint falhar
- Prevenção de N+1: `FetchType.EAGER` proibido, sem carregar associações em loop

---

## Documentação

- OpenAPI em **português do Brasil**
- Diagramas **Mermaid** atualizados quando o domínio mudar
- KDoc apenas quando o "porquê" não é óbvio

---

## Versionamento

Série `{{VERSION_SERIES}}` — MAJOR fixo durante o MVP.

| Mudança | Versão |
|---|---|
| Nova Spec ou conjunto de funcionalidades de negócio | MINOR |
| Correções e ajustes de specs já entregues | PATCH |

**Formato de commit obrigatório:** `tipo(módulo): descrição — SPEC-XX`
Tipos: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`

### Comando `/release`

1. Listar commits desde a última tag
2. Calcular versão (`feat` → MINOR, só `fix`/`chore` → PATCH)
3. Atualizar `CHANGELOG.md`
4. Criar commit `chore(release): bump version to X.Y.Z` + tag anotada `vX.Y.Z`
5. Exibir resumo e **aguardar confirmação** antes de `git push --follow-tags`