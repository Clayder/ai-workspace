---
name: domain-language
description: >-
  Ubiquitous language, aggregates, Value Objects, error prefixes, tables and
  critical flows of the expense-service. Consult this skill whenever you need
  domain names, error codes, realistic examples, or the specific table structure
  of this project. Use in combination with the other architecture skills when
  creating any artifact for the expense-service.
user-invocable: true
disable-model-invocation: false
---

# Domain Language — expense-service

## Service overview

Microservice of the **FinanceFlow** platform responsible exclusively for the
**expense** domain — recording, categorizing, and tracking financial outflows.
Income and balance tracking are the responsibility of future services.

---

## Aggregates and entities

| Aggregate | Responsibility |
|---|---|
| `Expense` | One-off expense (debit, Pix, cash) |
| `CreditCard` | Credit card with limit and billing cycle |
| `Invoice` | Monthly credit card invoice |
| `Installment` | Installment of a credit card purchase |
| `Bill` | Recurring fixed bill (rent, internet, etc.) |
| `BillPayment` | Payment record for a fixed bill |
| `Category` | Expense category (default or custom) |

---

## Shared Value Objects

| Value Object | Base type | Rules |
|---|---|---|
| `UserId` | `UUID` | Authenticated user identifier |
| `ExpenseId` | `UUID` | — |
| `CardId` | `UUID` | — |
| `InvoiceId` | `UUID` | — |
| `InstallmentId` | `UUID` | — |
| `BillId` | `UUID` | — |
| `CategoryId` | `UUID` | — |
| `Money` | `Long` | Amount in **cents** — never negative |
| `ExpenseDescription` | `String` | Non-empty, max 255 chars |
| `PixKey` | `String` | Optional — informational only |
| `PaymentMethod` | `enum` | `CREDIT_CARD`, `DEBIT`, `PIX`, `CASH` |

---

## Error prefix and code catalog

**This service's prefix:** `EX`

| Code | HTTP | Description |
|---|---|---|
| `CO_001` | 400 | Invalid input data |
| `CO_002` | 400 | Invalid or missing request body |
| `CO_999` | 500 | Internal server error |
| `EX_001` | 404 | Category not found |
| `EX_002` | 404 | Expense not found |
| `EX_003` | 404 | Card not found |
| `EX_004` | 404 | Fixed bill not found |
| `EX_005` | 403 | Access denied to expense |
| `EX_006` | 403 | Access denied to card |
| `EX_007` | 422 | Insufficient card limit |
| `EX_008` | 422 | Invoice already paid |

> When creating a new exception, add the code here and document it in Swagger.

---

## Database tables

| Table | Key columns |
|---|---|
| `users` | `id UUID`, `name VARCHAR`, `email VARCHAR UNIQUE`, `password_hash VARCHAR` |
| `categories` | `id UUID`, `user_id UUID`, `name VARCHAR`, `icon VARCHAR`, `color VARCHAR`, `parent_id UUID NULL` |
| `credit_cards` | `id UUID`, `user_id UUID`, `name VARCHAR`, `brand VARCHAR`, `limit_cents BIGINT`, `closing_day SMALLINT`, `due_day SMALLINT` |
| `bills` | `id UUID`, `user_id UUID`, `name VARCHAR`, `amount_cents BIGINT`, `due_day SMALLINT`, `recurrence VARCHAR`, `category_id UUID` |
| `bill_payments` | `id UUID`, `bill_id UUID`, `paid_at DATE`, `amount_paid_cents BIGINT`, `payment_method VARCHAR` |
| `expenses` | `id UUID`, `user_id UUID`, `amount_cents BIGINT`, `date DATE`, `description VARCHAR`, `category_id UUID`, `payment_method VARCHAR`, `card_id UUID NULL`, `pix_key VARCHAR NULL` |
| `installments` | `id UUID`, `expense_id UUID`, `installment_number SMALLINT`, `total_installments SMALLINT`, `amount_cents BIGINT`, `invoice_date DATE` |
| `invoices` | `id UUID`, `card_id UUID`, `closing_date DATE`, `due_date DATE`, `total_amount_cents BIGINT`, `paid_at TIMESTAMP NULL` |

All tables have `created_at`, `updated_at`, and `deleted_at`.

---

## Core business rules

1. **Monetary values:** always in cents (`BIGINT` / `Long`) — never `Float`/`Double`
2. **Data isolation:** all access filtered by `user_id` of the authenticated user
3. **Invoice:** groups purchases between consecutive closing dates; purchases after closing go into the next invoice
4. **Installments:** a purchase in N installments generates N records (one per invoice), amount = `total / N`
5. **Available limit:** `total limit - current invoice purchases - future installments`
6. **Fixed bills and holidays:** keeps the original due date, displays an early payment warning

---

## Critical flows (BDD)

| Feature | Aggregates involved |
|---|---|
| `credit_card_expense.feature` | `CreditCard`, `Invoice`, `Installment`, `Expense` |
| `invoice_management.feature` | `CreditCard`, `Invoice` |
| `one_off_expense.feature` | `Expense`, `Category` |
| `due_date_alerts.feature` | `Bill`, `BillPayment` |

---

## Realistic examples for Swagger and tests

```
Realistic UUID:   a1b2c3d4-e5f6-7890-abcd-ef1234567890
Money (R$49.90):  4990
Money (R$150.00): 15000
Date:             2026-03-15
Email:            joao@financeflow.com.br
Pix key:          11999998888
Description:      Farmácia Drogasil
Merchant:         Restaurante Fogo de Chão
```

---

## API endpoints

Base path: `/api/v1/`

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/register` | Registration (public) |
| `POST` | `/auth/login` | Login (public) |
| `POST` | `/auth/forgot-password` | Password recovery (public) |
| `GET/POST` | `/expenses` | One-off expenses |
| `PUT/DELETE` | `/expenses/:id` | Update / delete expense |
| `GET/POST` | `/cards` | Credit cards |
| `GET` | `/cards/:id/invoices` | Card invoices |
| `GET/POST` | `/bills` | Fixed bills |
| `POST` | `/bills/:id/payments` | Fixed bill payment |
| `GET/POST` | `/categories` | Categories |
| `GET` | `/reports/summary` | Summary by period |
| `GET` | `/reports/by-category` | Expenses by category |
| `GET` | `/dashboard` | Consolidated dashboard |