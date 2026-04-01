---
name: domain-language
description: >-
  Linguagem ubíqua, agregados, Value Objects, prefixos de erro, tabelas e
  fluxos críticos do expense-service. Consulte esta skill sempre que precisar
  de nomes de domínio, códigos de erro, exemplos realistas ou estrutura de
  tabelas específicas deste projeto. Use em combinação com as demais skills
  de arquitetura ao criar qualquer artefato do expense-service.
user-invocable: true
disable-model-invocation: false
---

# Linguagem do Domínio — expense-service

## Visão geral do serviço

Microserviço da plataforma **FinanceFlow** responsável exclusivamente pelo
domínio de **gastos** — registro, categorização e acompanhamento de saídas financeiras.
Receitas e saldo são responsabilidade de serviços futuros.

---

## Agregados e entidades

| Agregado | Responsabilidade |
|---|---|
| `Expense` | Gasto avulso (débito, Pix, dinheiro) |
| `CreditCard` | Cartão de crédito com limite e ciclo de fatura |
| `Invoice` | Fatura mensal de um cartão |
| `Installment` | Parcela de compra parcelada no cartão |
| `Bill` | Conta fixa recorrente (aluguel, internet, etc.) |
| `BillPayment` | Registro de pagamento de uma conta fixa |
| `Category` | Classificação de gastos (padrão ou personalizada) |

---

## Value Objects compartilhados

| Value Object | Tipo base | Regras |
|---|---|---|
| `UserId` | `UUID` | Identificador do usuário autenticado |
| `ExpenseId` | `UUID` | — |
| `CardId` | `UUID` | — |
| `InvoiceId` | `UUID` | — |
| `InstallmentId` | `UUID` | — |
| `BillId` | `UUID` | — |
| `CategoryId` | `UUID` | — |
| `Money` | `Long` | Valor em **centavos** — nunca negativo |
| `ExpenseDescription` | `String` | Não vazio, máx 255 chars |
| `PixKey` | `String` | Opcional — apenas informativo |
| `PaymentMethod` | `enum` | `CREDIT_CARD`, `DEBIT`, `PIX`, `CASH` |

---

## Prefixo de erro e catálogo de codes

**Prefixo deste serviço:** `EX`

| Code | HTTP | Descrição |
|---|---|---|
| `CO_001` | 400 | Dados de entrada inválidos |
| `CO_002` | 400 | Corpo da requisição inválido ou ausente |
| `CO_999` | 500 | Erro interno do servidor |
| `EX_001` | 404 | Categoria não encontrada |
| `EX_002` | 404 | Gasto não encontrado |
| `EX_003` | 404 | Cartão não encontrado |
| `EX_004` | 404 | Conta fixa não encontrada |
| `EX_005` | 403 | Acesso negado ao gasto |
| `EX_006` | 403 | Acesso negado ao cartão |
| `EX_007` | 422 | Limite do cartão insuficiente |
| `EX_008` | 422 | Fatura já paga |

> Ao criar uma nova exceção, adicione o code aqui e documente no Swagger.

---

## Tabelas do banco de dados

| Tabela | Colunas principais |
|---|---|
| `users` | `id UUID`, `name VARCHAR`, `email VARCHAR UNIQUE`, `password_hash VARCHAR` |
| `categories` | `id UUID`, `user_id UUID`, `name VARCHAR`, `icon VARCHAR`, `color VARCHAR`, `parent_id UUID NULL` |
| `credit_cards` | `id UUID`, `user_id UUID`, `name VARCHAR`, `brand VARCHAR`, `limit_cents BIGINT`, `closing_day SMALLINT`, `due_day SMALLINT` |
| `bills` | `id UUID`, `user_id UUID`, `name VARCHAR`, `amount_cents BIGINT`, `due_day SMALLINT`, `recurrence VARCHAR`, `category_id UUID` |
| `bill_payments` | `id UUID`, `bill_id UUID`, `paid_at DATE`, `amount_paid_cents BIGINT`, `payment_method VARCHAR` |
| `expenses` | `id UUID`, `user_id UUID`, `amount_cents BIGINT`, `date DATE`, `description VARCHAR`, `category_id UUID`, `payment_method VARCHAR`, `card_id UUID NULL`, `pix_key VARCHAR NULL` |
| `installments` | `id UUID`, `expense_id UUID`, `installment_number SMALLINT`, `total_installments SMALLINT`, `amount_cents BIGINT`, `invoice_date DATE` |
| `invoices` | `id UUID`, `card_id UUID`, `closing_date DATE`, `due_date DATE`, `total_amount_cents BIGINT`, `paid_at TIMESTAMP NULL` |

Todas as tabelas possuem `created_at`, `updated_at` e `deleted_at`.

---

## Regras de negócio fundamentais

1. **Valores monetários:** sempre em centavos (`BIGINT` / `Long`) — nunca `Float`/`Double`
2. **Isolamento de dados:** todo acesso filtrado por `user_id` do usuário autenticado
3. **Fatura:** agrupa compras entre fechamentos consecutivos; compras após fechamento entram na próxima
4. **Parcelas:** compra em N vezes gera N lançamentos (um por fatura), valor = `total / N`
5. **Limite disponível:** `limite total - compras fatura atual - parcelas futuras`
6. **Contas fixas e feriados:** mantém data original, exibe aviso de pagamento antecipado

---

## Fluxos críticos (BDD)

| Feature | Agregados envolvidos |
|---|---|
| `registro_gasto_cartao.feature` | `CreditCard`, `Invoice`, `Installment`, `Expense` |
| `controle_fatura.feature` | `CreditCard`, `Invoice` |
| `registro_gasto_avulso.feature` | `Expense`, `Category` |
| `alertas_vencimento.feature` | `Bill`, `BillPayment` |

---

## Exemplos realistas para Swagger e testes

```
UUID realista:    a1b2c3d4-e5f6-7890-abcd-ef1234567890
Money (R$49,90):  4990
Money (R$150,00): 15000
Data:             2026-03-15
Email:            joao@financeflow.com.br
Chave Pix:        11999998888
Descrição:        Farmácia Drogasil
Estabelecimento:  Restaurante Fogo de Chão
```

---

## Endpoints da API

Base path: `/api/v1/`

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/auth/register` | Cadastro (público) |
| `POST` | `/auth/login` | Login (público) |
| `POST` | `/auth/forgot-password` | Recuperação de senha (público) |
| `GET/POST` | `/expenses` | Gastos avulsos |
| `PUT/DELETE` | `/expenses/:id` | Edição / exclusão de gasto |
| `GET/POST` | `/cards` | Cartões de crédito |
| `GET` | `/cards/:id/invoices` | Faturas de um cartão |
| `GET/POST` | `/bills` | Contas fixas |
| `POST` | `/bills/:id/payments` | Pagamento de conta fixa |
| `GET/POST` | `/categories` | Categorias |
| `GET` | `/reports/summary` | Resumo por período |
| `GET` | `/reports/by-category` | Gastos por categoria |
| `GET` | `/dashboard` | Dashboard consolidado |