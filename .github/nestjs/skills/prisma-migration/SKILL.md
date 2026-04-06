---
name: prisma-migration
description: >-
  Use ao criar ou modificar migrations de banco de dados com Prisma e PostgreSQL.
  Ative para termos como: "criar migration", "nova tabela", "alterar schema",
  "adicionar coluna", "criar índice", "Prisma", "DDL", "migration SQL",
  "modificar banco", "criar tabela", "atualizar modelo do banco",
  ou qualquer tarefa que envolva mudança no schema do banco de dados.
  Stack: TypeScript + NestJS + Prisma + PostgreSQL.
argument-hint: "[descrição da mudança no schema]"
user-invocable: true
disable-model-invocation: false
---

# Como criar Migrations com Prisma — PostgreSQL

> Consulte `prisma/schema.prisma` para ver os modelos já definidos no projeto
> e as convenções de nomenclatura de campos.

## Regra inegociável

Schema alterado **exclusivamente** via migrations Prisma.
Nunca use `db push` em produção. Nunca altere o banco diretamente.

```bash
# Desenvolvimento — cria e aplica (interativo, gera o arquivo SQL)
npx prisma migrate dev --name descricao_em_snake_case

# CI/produção — aplica migrations pendentes
npx prisma migrate deploy

# Após alterar o schema, sempre regenere o cliente
npx prisma generate
```

---

## Convenções de nomenclatura

### Modelos Prisma (schema.prisma)
- Nome do modelo: **PascalCase** → `CareerCompany`, `Achievement`
- Nome da tabela (via `@@map`): **snake_case no plural** → `career_companies`, `achievements`
- Campos do modelo: **camelCase** → `startedAt`, `impactCategory`
- Colunas no banco (via `@map`): **snake_case** → `started_at`, `impact_category`

---

## Campos obrigatórios em toda tabela de negócio (inegociável)

```prisma
createdAt   DateTime   @default(now()) @map("created_at")
updatedAt   DateTime   @updatedAt      @map("updated_at")
deletedAt   DateTime?                  @map("deleted_at")  // soft delete obrigatório
```

Hard delete é **proibido**. O filtro `deletedAt == null` é responsabilidade exclusiva da `infrastructure`.

---

## Template: novo modelo no schema.prisma

```prisma
model {Entidade} {
  id        String    @id @default(uuid())
  userId    String    @map("user_id")

  // campos de negócio aqui

  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt      @map("updated_at")
  deletedAt DateTime?                 @map("deleted_at")

  // relações
  user      User      @relation(fields: [userId], references: [id])

  @@map("{entidades}")
  @@index([userId], map: "idx_{entidades}_user_id")
}
```

---

## Exemplo real — modelo Achievement

```prisma
model Achievement {
  id              String    @id @default(uuid())
  careerCompanyId String    @map("career_company_id")
  roleId          String    @map("role_id")

  complexity      String    // enum: LOW | MEDIUM | HIGH | CRITICAL
  impactCategory  String    @map("impact_category")  // enum: EFFICIENCY | LEADERSHIP | RISK | INNOVATION | QUALITY
  captureChannel  String    @default("WEB") @map("capture_channel")

  starSituation   String    @map("star_situation")
  starTask        String    @map("star_task")
  starAction      String    @map("star_action")
  starResult      String    @map("star_result")

  evidenceLinks   String[]  @default([]) @map("evidence_links")

  registeredAt    DateTime  @default(now()) @map("registered_at")
  revisedAt       DateTime?                 @map("revised_at")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt      @map("updated_at")
  deletedAt       DateTime?                 @map("deleted_at")

  careerCompany   CareerCompany @relation(fields: [careerCompanyId], references: [id])
  role            Role          @relation(fields: [roleId], references: [id])

  @@map("achievements")
  @@index([careerCompanyId, deletedAt], map: "idx_achievements_company_soft_delete")
  @@index([roleId, deletedAt],          map: "idx_achievements_role_soft_delete")
}
```

---

## Convenções de índices

```prisma
// Índice simples com soft delete
@@index([campo, deletedAt], map: "idx_{tabela}_{campo}_soft_delete")

// Índice único (ex: slug de página pública)
@@unique([slug], map: "uq_{tabelas}_slug")

// Índice composto para queries frequentes
@@index([userId, status, deletedAt], map: "idx_{tabelas}_user_status")
```

---

## Valores monetários

Sempre `BigInt` (centavos) — nunca `Float`, `Decimal` ou `String` para moeda:

```prisma
amount  BigInt  @map("amount")   // representa centavos
```

---

## Enums: Prisma vs. TypeScript

O projeto **não usa enum nativo do Prisma** para manter flexibilidade de migração.
Enums são armazenados como `String` no banco e validados no domínio:

```prisma
// No schema.prisma — String, não enum Prisma
complexity  String  // validado como Complexity enum no domínio
```

```typescript
// No domínio TypeScript — enum explícito
export enum Complexity {
  LOW = 'LOW',
  MEDIUM = 'MEDIUM',
  HIGH = 'HIGH',
  CRITICAL = 'CRITICAL',
}
```

---

## Fluxo completo de uma mudança de schema

```bash
# 1. Altere o schema.prisma
# 2. Crie a migration
npx prisma migrate dev --name add_revised_at_to_achievements

# 3. Revise o SQL gerado em prisma/migrations/{timestamp}_{name}/migration.sql
# 4. Regenere o cliente Prisma
npx prisma generate

# 5. Atualize o adaptador de repositório correspondente se necessário
```

---

## O que verificar no SQL gerado

Após `prisma migrate dev`, revise o arquivo `migration.sql` gerado e confirme:

- `NOT NULL` apenas em campos realmente obrigatórios
- `DEFAULT` correto nos campos com valor padrão
- Índices criados para colunas usadas em filtros de queries frequentes
- `REFERENCES` para chaves estrangeiras
- Nenhuma coluna sensível sem restrição adequada

---

## Checklist

- [ ] Schema alterado via `prisma migrate dev` — nunca `db push` em produção
- [ ] Modelo com `@@map` em snake_case plural
- [ ] Campos com `@map` em snake_case
- [ ] `createdAt`, `updatedAt` e `deletedAt` presentes em toda tabela de negócio
- [ ] Valores monetários como `BigInt`
- [ ] Enums como `String` no Prisma + enum TypeScript no domínio
- [ ] Índices com `deletedAt` nas queries filtradas por soft delete
- [ ] `npx prisma generate` executado após a migration
- [ ] SQL gerado revisado antes de commitar