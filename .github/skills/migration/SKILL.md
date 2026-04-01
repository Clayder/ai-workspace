---
name: migration
description: >-
  Use ao criar ou modificar migrations de banco de dados com Flyway e PostgreSQL.
  Ative para termos como: "criar migration", "nova tabela", "alterar schema",
  "adicionar coluna", "criar índice", "Flyway", "DDL", "SQL migration",
  "modificar banco", "criar tabela", ou qualquer tarefa que envolva
  mudança no schema do banco de dados.
argument-hint: "[descrição da mudança no schema]"
user-invocable: true
disable-model-invocation: false
---

# Como criar Migrations Flyway — PostgreSQL

> Consulte `/domain-language` para ver as tabelas já existentes no projeto
> e as convenções de nomenclatura de colunas do domínio.

## Regra fundamental (NÃO NEGOCIÁVEL)

Schema alterado **exclusivamente** via Flyway migrations.
Nunca usar `spring.jpa.hibernate.ddl-auto=create` ou `update`.

## Nomenclatura

```
V{versão}__{descricao_em_ingles_snake_case}.sql
```

- Dois underscores (`__`) entre versão e descrição
- Versões sequenciais — sem reutilizar ou pular
- Localização: `src/main/resources/db/migration/`

## Regras de schema (NÃO NEGOCIÁVEL)

### Soft delete obrigatório

```sql
deleted_at TIMESTAMP WITH TIME ZONE NULL
```

Hard delete é **PROIBIDO**. Filtro `deleted_at IS NULL` é responsabilidade exclusiva da `infrastructure`.

### Timestamps padrão em toda tabela de negócio

```sql
created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
deleted_at  TIMESTAMP WITH TIME ZONE NULL
```

### Valores monetários

Sempre `BIGINT` (centavos) — nunca `DECIMAL`, `FLOAT` ou `NUMERIC`.

### Índices com filtro de soft delete

```sql
CREATE INDEX idx_{tabela}_{coluna}
    ON {tabela} ({coluna})
    WHERE deleted_at IS NULL;
```

## Template de migration completa

```sql
-- V{N}__{descricao}.sql

CREATE TABLE {tabela} (
    id          UUID                        NOT NULL,

    -- campos de negócio aqui

    created_at  TIMESTAMP WITH TIME ZONE    NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE    NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMP WITH TIME ZONE    NULL,

    CONSTRAINT pk_{tabela} PRIMARY KEY (id),
    CONSTRAINT fk_{tabela}_{ref} FOREIGN KEY ({col_fk}) REFERENCES {tabela_ref}(id),
    CONSTRAINT chk_{tabela}_{regra} CHECK ({condicao})
);

CREATE INDEX idx_{tabela}_{coluna}
    ON {tabela} ({coluna})
    WHERE deleted_at IS NULL;
```

## Template: adicionar coluna

```sql
-- V{N}__add_{coluna}_to_{tabela}.sql

ALTER TABLE {tabela}
    ADD COLUMN {coluna} {TIPO} {NULL|NOT NULL};

COMMENT ON COLUMN {tabela}.{coluna} IS 'Descrição do campo';
```

## Checklist

- [ ] Nome: `V{N}__{descricao}.sql` com dois underscores
- [ ] Versão sequencial sem conflito
- [ ] `deleted_at TIMESTAMP WITH TIME ZONE NULL` presente
- [ ] `created_at` e `updated_at` com `DEFAULT NOW()`
- [ ] Valores monetários como `BIGINT`
- [ ] Índices com `WHERE deleted_at IS NULL` nas colunas de filtragem frequente
- [ ] Constraints de integridade declaradas
- [ ] Migration testada localmente