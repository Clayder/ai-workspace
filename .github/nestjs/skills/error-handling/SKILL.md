---
name: error-handling
description: >-
  Use ao implementar tratamento de erros, exceções de domínio ou respostas HTTP de erro
  seguindo Arquitetura Hexagonal com TypeScript e NestJS. Ative para termos como:
  "criar exceção de domínio", "erro de negócio", "GlobalExceptionFilter",
  "erro 400", "erro 422", "erro 403", "DomainException", "tratar exceção",
  "resposta de erro", "código de erro", "violations", "padrão de erro da API",
  ou qualquer tarefa relacionada a tratamento de erros HTTP.
  Stack: TypeScript + NestJS + Fastify.
argument-hint: "[tipo de erro ou exceção a implementar]"
user-invocable: true
disable-model-invocation: false
---

# Como implementar tratamento de erros — Arquitetura Hexagonal (TypeScript/NestJS)

> Consulte `docs/prd/regras-de-negocio.md` para a tabela completa de códigos de erro
> e os prefixos por módulo já definidos no projeto.

## Padrão de erros HTTP (inegociável)

| Código | Quando usar | Origem | Estrutura |
|---|---|---|---|
| `400` | Validação de **input** (campo inválido, ausente, formato errado) | `ValidationPipe` na camada presentation | `{ message, error_code, violations[] }` |
| `422` | Violação de **regra de negócio** | `DomainException` na camada domain | `{ message, error_code }` |
| `403` | Recurso existe mas usuário não tem permissão ou limite de tier atingido | `ApplicationException` na camada application | `{ message, error_code }` |
| `404` | Recurso genuinamente não existe | `ApplicationException` na camada application | `{ message, error_code }` |
| `401` | Token ausente ou expirado | Guard JWT | `{ message, error_code }` |

**Proibido:**
- `400` para regras de negócio
- `422` para campos de input inválidos
- `404` para mascarar existência quando é problema de autorização (use `403`)
- Stack trace ou mensagem interna de exceção na resposta HTTP

---

## Hierarquia de exceções

```typescript
// src/shared/exceptions/DomainException.ts
// Para violações de regra de negócio → HTTP 422
export class DomainException extends Error {
  constructor(
    override readonly message: string,
    readonly errorCode: string,
  ) {
    super(message);
    this.name = 'DomainException';
  }
}

// src/shared/exceptions/ApplicationException.ts
// Para erros de caso de uso com status customizável → HTTP 403, 404, 409
export class ApplicationException extends Error {
  constructor(
    override readonly message: string,
    readonly errorCode: string,
    readonly statusCode: number,
  ) {
    super(message);
    this.name = 'ApplicationException';
  }
}
```

---

## Templates de exceções por módulo

```typescript
// domain/aggregates/{Agregado}/exceptions/{Entidade}NotFoundException.ts
// → HTTP 404
export class {Entidade}NotFoundException extends ApplicationException {
  constructor(id: string) {
    super(
      `{Entidade} não encontrada: ${id}`,
      '{PREFIXO}_NNN',
      404,
    );
  }
}

// domain/aggregates/{Agregado}/exceptions/{Entidade}AccessDeniedException.ts
// → HTTP 403
export class {Entidade}AccessDeniedException extends ApplicationException {
  constructor(id: string) {
    super(
      `Acesso negado ao recurso: ${id}`,
      '{PREFIXO}_NNN',
      403,
    );
  }
}

// domain/aggregates/{Agregado}/exceptions/{Regra}Exception.ts
// → HTTP 422
export class {Regra}Exception extends DomainException {
  constructor() {
    super(
      'Descrição clara da regra de negócio violada',
      '{PREFIXO}_NNN',
    );
  }
}
```

---

## Exemplos reais do uMeritor

```typescript
// → HTTP 403
export class AchievementLimitReachedException extends ApplicationException {
  constructor() {
    super(
      'Limite de conquistas do plano gratuito atingido',
      'ACHIEVEMENT_LIMIT_REACHED',
      403,
    );
  }
}

// → HTTP 422
export class CareerCompanyConcludedException extends DomainException {
  constructor(companyId: string) {
    super(
      `Não é possível adicionar registros a um vínculo encerrado: ${companyId}`,
      'CAREER_COMPANY_CONCLUDED',
    );
  }
}

// → HTTP 403
export class FeatureNotAvailableException extends ApplicationException {
  constructor(feature: string) {
    super(
      `Feature '${feature}' não disponível no plano atual`,
      'FEATURE_NOT_AVAILABLE',
      403,
    );
  }
}
```

---

## DTOs de resposta de erro (shared/)

```typescript
// src/shared/filters/ErrorResponse.ts

export class ViolationResponse {
  property!: string;
  message!: string;
  value?: unknown;
}

export class ErrorResponse {
  message!: string;
  error_code!: string;       // snake_case — padrão da API
  violations?: ViolationResponse[];
}
```

---

## GlobalExceptionFilter (único responsável por montar ErrorResponse)

```typescript
// src/shared/filters/GlobalExceptionFilter.ts

import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { FastifyReply } from 'fastify';
import { DomainException } from '../exceptions/DomainException';
import { ApplicationException } from '../exceptions/ApplicationException';
import { ErrorResponse } from './ErrorResponse';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const reply = ctx.getResponse<FastifyReply>();

    if (exception instanceof ApplicationException) {
      reply.status(exception.statusCode).send({
        message: exception.message,
        error_code: exception.errorCode,
      } satisfies ErrorResponse);
      return;
    }

    if (exception instanceof DomainException) {
      reply.status(HttpStatus.UNPROCESSABLE_ENTITY).send({
        message: exception.message,
        error_code: exception.errorCode,
      } satisfies ErrorResponse);
      return;
    }

    if (exception instanceof HttpException) {
      const response = exception.getResponse();
      // ValidationPipe lança HttpException com campo `message` array
      if (typeof response === 'object' && 'message' in response) {
        const messages = Array.isArray((response as { message: unknown }).message)
          ? (response as { message: string[] }).message
          : [(response as { message: string }).message];

        reply.status(HttpStatus.BAD_REQUEST).send({
          message: 'Dados de entrada inválidos',
          error_code: 'CO_001',
          violations: messages.map(msg => ({
            property: '',
            message: msg,
          })),
        } satisfies ErrorResponse);
        return;
      }
      reply.status(exception.getStatus()).send({
        message: exception.message,
        error_code: 'CO_000',
      } satisfies ErrorResponse);
      return;
    }

    // Fallback — nunca expõe stack trace
    reply.status(HttpStatus.INTERNAL_SERVER_ERROR).send({
      message: 'Erro interno do servidor',
      error_code: 'CO_999',
    } satisfies ErrorResponse);
  }
}
```

---

## Registro do filtro global (main.ts)

```typescript
// src/main.ts
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
app.useGlobalFilters(new GlobalExceptionFilter());
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

---

## Tabela de códigos de erro do projeto

| Código | HTTP | Descrição |
|---|---|---|
| `CO_001` | 400 | Dados de entrada inválidos (ValidationPipe) |
| `CO_999` | 500 | Erro interno não tratado |
| `INVALID_CREDENTIALS` | 401 | E-mail ou senha incorretos |
| `INVALID_OR_EXPIRED_TOKEN` | 401 | Token JWT expirado ou inválido |
| `ACHIEVEMENT_LIMIT_REACHED` | 403 | Limite de conquistas do plano gratuito |
| `CAREER_COMPANY_LIMIT_REACHED` | 403 | Limite de empresas ativas do plano gratuito |
| `FEATURE_NOT_AVAILABLE` | 403 | Feature não disponível no plano atual |
| `CAREER_COMPANY_CONCLUDED` | 422 | Vínculo encerrado não aceita novos registros |
| `SLUG_ALREADY_TAKEN` | 409 | Slug da página pública já em uso |
| `BASE_PROFILE_REQUIRED` | 422 | Perfil LinkedIn necessário para o recurso |

> Novos códigos devem ser adicionados a esta tabela e documentados no `@ApiResponse` do controller.

---

## Checklist

- [ ] Exceção estende `DomainException` (422) ou `ApplicationException` (código customizável)
- [ ] `400` apenas para input — `422` apenas para regra de negócio
- [ ] `403` para recurso existente sem permissão ou limite atingido — nunca `404`
- [ ] `GlobalExceptionFilter` é o único lugar que monta `ErrorResponse`
- [ ] Resposta nunca expõe stack trace
- [ ] Código registrado na tabela acima e documentado no `@ApiResponse` do controller