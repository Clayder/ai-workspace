---
name: migration
description: >-
  Use when creating or modifying database migrations with Flyway and PostgreSQL.
  Activate for terms like: "create migration", "new table", "alter schema",
  "add column", "create index", "Flyway", "DDL", "SQL migration",
  "modify database", "create table", or any task involving
  a change to the database schema.
argument-hint: "[description of the schema change]"
user-invocable: true
disable-model-invocation: false
---

# How to create Flyway Migrations — PostgreSQL

> Consult `/domain-language` to see the tables already defined in the project
> and the domain column naming conventions.

## Core rule (NON-NEGOTIABLE)

Schema changed **exclusively** via Flyway migrations.
Never use `spring.jpa.hibernate.ddl-auto=create` or `update`.

## Naming

```
V{version}__{description_in_english_snake_case}.sql
```

- Two underscores (`__`) between version and description
- Sequential versions — never reuse or skip
- Location: `src/main/resources/db/migration/`

## Schema rules (NON-NEGOTIABLE)

### Mandatory soft delete

```sql
deleted_at TIMESTAMP WITH TIME ZONE NULL
```

Hard delete is **PROHIBITED**. The `deleted_at IS NULL` filter is the exclusive responsibility of `infrastructure`.

### Standard timestamps on every business table

```sql
created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
deleted_at  TIMESTAMP WITH TIME ZONE NULL
```

### Monetary values

Always `BIGINT` (cents) — never `DECIMAL`, `FLOAT`, or `NUMERIC`.

### Indexes with soft-delete filter

```sql
CREATE INDEX idx_{table}_{column}
    ON {table} ({column})
    WHERE deleted_at IS NULL;
```

## Full migration template

```sql
-- V{N}__{description}.sql

CREATE TABLE {table} (
    id          UUID                        NOT NULL,

    -- business fields here

    created_at  TIMESTAMP WITH TIME ZONE    NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE    NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMP WITH TIME ZONE    NULL,

    CONSTRAINT pk_{table} PRIMARY KEY (id),
    CONSTRAINT fk_{table}_{ref} FOREIGN KEY ({fk_col}) REFERENCES {ref_table}(id),
    CONSTRAINT chk_{table}_{rule} CHECK ({condition})
);

CREATE INDEX idx_{table}_{column}
    ON {table} ({column})
    WHERE deleted_at IS NULL;
```

## Template: add column

```sql
-- V{N}__add_{column}_to_{table}.sql

ALTER TABLE {table}
    ADD COLUMN {column} {TYPE} {NULL|NOT NULL};

COMMENT ON COLUMN {table}.{column} IS 'Field description';
```

## Checklist

- [ ] Name: `V{N}__{description}.sql` with two underscores
- [ ] Sequential version with no conflicts
- [ ] `deleted_at TIMESTAMP WITH TIME ZONE NULL` present
- [ ] `created_at` and `updated_at` with `DEFAULT NOW()`
- [ ] Monetary values as `BIGINT`
- [ ] Indexes with `WHERE deleted_at IS NULL` on frequently filtered columns
- [ ] Integrity constraints declared
- [ ] Migration tested locally