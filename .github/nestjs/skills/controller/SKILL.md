---
name: controller
description: >-
  Use ao criar um Controller REST na camada presentation seguindo Arquitetura Hexagonal.
  Ative para termos como: "criar controller", "novo endpoint", "criar rota",
  "expor API", "criar Request", "criar Response", "documentar endpoint",
  "adicionar endpoint", "criar DTO", ou qualquer tarefa que envolva receber
  requisições HTTP e delegar para um Caso de Uso.
  Stack: TypeScript + NestJS + Fastify + Swagger.
argument-hint: "[recurso] [operação]"
user-invocable: true
disable-model-invocation: false
---

# Como criar um Controller REST — Arquitetura Hexagonal (TypeScript/NestJS)

> Consulte `docs/prd/api-endpoints.md` para nomes de recursos, prefixos de rota
> e contratos já definidos para o projeto.

## Localização dos arquivos

```
presentation/controllers/{recurso}/
├── {Recurso}Controller.ts      # Decorators NestJS + Swagger + delegação ao UseCase
├── {Acao}Request.ts            # DTO de entrada com class-validator
└── {Acao}Response.ts           # Classe de resposta em snake_case
```

> **Diferença do Kotlin:** No NestJS, Swagger e Spring MVC ficam no **mesmo arquivo** do controller.
> Não há interface separada `ControllerDoc`.

---

## Regra inegociável

O controller **apenas** recebe, valida e delega. Nunca contém regra de negócio,
verificação de autorização ou montagem de `ErrorResponse`.

---

## Template: {Acao}Request.ts

```typescript
// presentation/controllers/{recurso}/{Acao}Request.ts

import { ApiProperty } from '@nestjs/swagger';
import { IsString, IsEnum, MinLength, IsOptional, IsArray, IsUrl } from 'class-validator';

export class {Acao}Request {
  @ApiProperty({
    description: 'Descrição do campo em português — significado de negócio',
    example: 'valor realista do domínio',
  })
  @IsString()
  @MinLength(/* mínimo */)
  campo!: string;
}
```

**Regras dos Request DTOs:**
- Campos em **camelCase** (padrão JSON do NestJS)
- `@ApiProperty` com `description` em português e `example` com valor realista de domínio
- Nunca `example: 'string'`, `example: 0` ou `example: 'uuid'`
- Sufixo de validação `!` (non-null assertion) pois `class-transformer` garante o valor

---

## Template: {Acao}Response.ts

```typescript
// presentation/controllers/{recurso}/{Acao}Response.ts

import { ApiProperty } from '@nestjs/swagger';
import { {Acao}Output } from '../../../application/use-cases/{agregado}/{acao}/{Acao}Output';

export class {Acao}Response {
  @ApiProperty({ description: 'Identificador único', example: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890' })
  id!: string;

  @ApiProperty({ description: 'Data de criação no formato ISO 8601', example: '2024-01-15T10:30:00Z' })
  created_at!: string;     // snake_case — sempre

  static fromOutput(output: {Acao}Output): {Acao}Response {
    const response = new {Acao}Response();
    response.id = output.id;
    response.created_at = output.created_at;
    return response;
  }
}
```

**Regras dos Response DTOs:**
- Campos em **snake_case** — regra de contrato da API
- Factory estático `fromOutput()` para mapear do Output do UseCase
- `@ApiProperty` em todos os campos com `description` e `example` realistas

---

## Template: {Recurso}Controller.ts

```typescript
// presentation/controllers/{recurso}/{Recurso}Controller.ts

import {
  Controller, Post, Get, Put, Delete, Body, Param, Query,
  HttpCode, HttpStatus, UseGuards,
} from '@nestjs/common';
import {
  ApiTags, ApiOperation, ApiResponse, ApiParam, ApiQuery, ApiBearerAuth,
} from '@nestjs/swagger';
import { JwtAuthGuard } from '../../../shared/guards/JwtAuthGuard';
import { CurrentUser } from '../../../shared/decorators/CurrentUser';
import { {Acao}UseCase } from '../../../application/use-cases/{agregado}/{acao}/{Acao}UseCase';
import { {Acao}Request } from './{Acao}Request';
import { {Acao}Response } from './{Acao}Response';

@ApiTags('{Recurso}')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('api/v1/{recursos}')
export class {Recurso}Controller {
  constructor(private readonly {acao}UseCase: {Acao}UseCase) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({
    summary: 'Ação curta em português',
    description: 'Objetivo da operação. Pré-condições. Papéis autorizados.',
  })
  @ApiResponse({ status: 201, description: 'Criado com sucesso', type: {Acao}Response })
  @ApiResponse({ status: 400, description: 'Dados de entrada inválidos (CO_001)' })
  @ApiResponse({ status: 401, description: 'Token JWT ausente ou expirado' })
  @ApiResponse({ status: 403, description: 'Sem permissão ou limite de plano atingido' })
  @ApiResponse({ status: 422, description: 'Regra de negócio violada' })
  async create(
    @Body() request: {Acao}Request,
    @CurrentUser() userId: string,
  ): Promise<{Acao}Response> {
    const output = await this.{acao}UseCase.execute({
      userId,
      // ...campos do request mapeados para o Input
    });
    return {Acao}Response.fromOutput(output);
  }
}
```

---

## Exemplo real — AchievementController (POST)

```typescript
@ApiTags('Conquistas')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('api/v1/career-companies/:companyId/roles/:roleId/achievements')
export class AchievementController {
  constructor(private readonly registerAchievementUseCase: RegisterAchievementUseCase) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({
    summary: 'Registrar conquista STAR',
    description: 'Registra uma nova conquista profissional no formato STAR para o cargo informado.',
  })
  @ApiParam({ name: 'companyId', description: 'ID do vínculo profissional' })
  @ApiParam({ name: 'roleId', description: 'ID do cargo' })
  @ApiResponse({ status: 201, description: 'Conquista registrada com sucesso', type: RegisterAchievementResponse })
  @ApiResponse({ status: 400, description: 'Dados de entrada inválidos — campos STAR abaixo de 20 caracteres (CO_001)' })
  @ApiResponse({ status: 401, description: 'Token JWT ausente ou expirado' })
  @ApiResponse({ status: 403, description: 'Limite de conquistas do plano gratuito atingido (ACH_001)' })
  @ApiResponse({ status: 422, description: 'Empresa encerrada — não aceita novos registros (ACH_002)' })
  async register(
    @Param('companyId') companyId: string,
    @Param('roleId') roleId: string,
    @Body() request: RegisterAchievementRequest,
    @CurrentUser() userId: string,
  ): Promise<RegisterAchievementResponse> {
    const output = await this.registerAchievementUseCase.execute({
      userId,
      companyId,
      roleId,
      complexity: request.complexity,
      impactCategory: request.impact_category,
      situation: request.star.situation,
      task: request.star.task,
      action: request.star.action,
      result: request.star.result,
      evidenceLinks: request.evidence_links ?? [],
    });
    return RegisterAchievementResponse.fromOutput(output);
  }
}
```

---

## Proibições

| Proibido | Correto |
|---|---|
| Regra de negócio no controller | Delegar ao UseCase |
| `if (user.tier !== 'PRO')` no controller | Verificar tier no UseCase |
| Montar `ErrorResponse` no controller | Responsabilidade do `GlobalExceptionFilter` |
| `example: 'string'` no Swagger | Valor realista do domínio |
| Campos em camelCase na Response | **snake_case** sempre |
| Campos em snake_case no Request | **camelCase** (padrão NestJS) |

---

## Checklist

- [ ] Arquivos em `presentation/controllers/{recurso}/`
- [ ] Controller com `@ApiTags`, `@ApiBearerAuth`, `@UseGuards(JwtAuthGuard)`
- [ ] `@ApiOperation` com `summary` e `description` em português
- [ ] `@ApiResponse` para todos os status possíveis: 2xx, 400, 401, 403, 422
- [ ] `@ApiParam` para todos os parâmetros de rota
- [ ] Request com `@ApiProperty` em todos os campos — sem exemplos genéricos
- [ ] Response com campos em `snake_case` e factory `fromOutput()`
- [ ] Controller não contém nenhuma lógica de negócio
- [ ] Controller não monta `ErrorResponse` diretamente
- [ ] `@CurrentUser()` para extrair userId do token