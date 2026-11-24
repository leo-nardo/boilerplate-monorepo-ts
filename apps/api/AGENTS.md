# Backend — Guia de Implementação (`apps/api`)

API REST modular em **NestJS + TypeScript + PostgreSQL** responsável por ingestão de fontes externas, validação, padronização, cálculo de indicadores e exposição de dados para dashboards.

## Comandos

```bash
pnpm --filter api dev             # Watch mode (http://localhost:3000)
pnpm --filter api build           # Compilação TypeScript
pnpm --filter api test            # Testes unitários (Jest)
pnpm --filter api test:watch      # TDD watch mode
pnpm --filter api test:e2e        # Testes de integração/E2E
pnpm --filter api migration:run   # Aplica migrations no PostgreSQL
```

## Estrutura de Pastas

```text
apps/api/src/
├── modules/
│   ├── integrations/
│   │   ├── adapters/                 # Uma classe isolada por fonte externa (SOLID)
│   │   ├── contracts/                # Interface SourceAdapter e tipos de ingestão
│   │   ├── integrations.controller.ts
│   │   ├── integrations.service.ts
│   │   └── integrations.module.ts
│   ├── indicators/
│   │   ├── calculators/              # Funções puras testadas via TDD (zero NestJS/ORM)
│   │   ├── indicators.controller.ts
│   │   ├── indicators.service.ts
│   │   └── indicators.module.ts
│   ├── analytics/                    # Consolidação e agregações entre fontes
│   └── dashboards/                   # Consultas otimizadas e filtros para a UI
├── common/
│   ├── filters/                      # GlobalExceptionFilter (RFC 7807 ProblemDetail)
│   ├── interceptors/                 # Logging estruturado com executionId
│   └── decorators/                   # Permissões e validações reutilizáveis
├── config/                           # Validação de variáveis de ambiente
└── main.ts
```

## Convenções de API

- Prefixo global de versão: `/v1` (ex.: `GET /v1/dashboards/operational`, `POST /v1/integrations/:source/sync`).
- Paginação **0-indexed** (`page=0`, `size=20`).
- Todos os endpoints devem possuir decorators OpenAPI (`@ApiTags`, `@ApiOperation`) e validação de entrada com DTOs.
- Datas trafegam em ISO-8601 (`YYYY-MM-DD` para períodos de referência e UTC para timestamps).

---

## Exemplo Canônico 1: Novo Adaptador de Fonte Externa (SOLID)

Cada nova fonte de dados entra como um adaptador isolado sem alterar o orquestrador ou outras fontes (`RNF-001`).

### 1. Contrato (`modules/integrations/contracts/source-adapter.interface.ts`)

```typescript
export interface StandardizedRecord<TPayload> {
  readonly sourceSystem: string;
  readonly externalId: string;
  readonly extractedAt: Date;
  readonly referencePeriod: string;
  readonly payload: TPayload;
}

export interface SourceAdapter<TPayload unknown> {
  readonly sourceName: string;
  extract(referencePeriod: string): Promise<readonly StandardizedRecord<TPayload>[]>;
}

export const SOURCE_ADAPTERS = Symbol("SOURCE_ADAPTERS");
```

### 2. Implementação de Adaptador (`modules/integrations/adapters/operational-volume.adapter.ts`)

```typescript
import { Injectable } from "@nestjs/common";
import type {
  SourceAdapter,
  StandardizedRecord,
} from "../contracts/source-adapter.interface";

export interface OperationalVolumePayload {
  readonly producedVolumeM3: number;
  readonly networkDischargeM3: number;
  readonly micrometeredVolumeM3: number;
}

@Injectable()
export class OperationalVolumeAdapter implements SourceAdapter<OperationalVolumePayload> {
  readonly sourceName = "operational-volume";

  async extract(
    referencePeriod: string,
  ): Promise<readonly StandardizedRecord<OperationalVolumePayload>[]> {
    const rawRows = await this.fetchFromOrigin(referencePeriod);
    const extractedAt = new Date();

    return rawRows.map((row) => ({
      sourceSystem: this.sourceName,
      externalId: row.id,
      extractedAt,
      referencePeriod,
      payload: {
        producedVolumeM3: Number(row.volume_produzido),
        networkDischargeM3: Number(row.volume_descarga),
        micrometeredVolumeM3: Number(row.volume_micromedido),
      },
    }));
  }

  private async fetchFromOrigin(referencePeriod: string): Promise<
    Array<{
      id: string;
      volume_produzido: number;
      volume_descarga: number;
      volume_micromedido: number;
    }>
  > {
    throw new Error(
      `Origin client not configured for period ${referencePeriod}`,
    );
  }
}
```

### 3. Registro no Módulo via DI (`modules/integrations/integrations.module.ts`)

```typescript
import { Module } from "@nestjs/common";
import { SOURCE_ADAPTERS } from "./contracts/source-adapter.interface";
import { OperationalVolumeAdapter } from "./adapters/operational-volume.adapter";
import { IntegrationsService } from "./integrations.service";

@Module({
  providers: [
    OperationalVolumeAdapter,
    {
      provide: SOURCE_ADAPTERS,
      useFactory: (operationalVolume: OperationalVolumeAdapter) => [
        operationalVolume,
      ],
      inject: [OperationalVolumeAdapter],
    },
    IntegrationsService,
  ],
  exports: [IntegrationsService],
})
export class IntegrationsModule {}
```

---

## Exemplo Canônico 2: Novo Indicador com TDD (Função Pura)

Fórmulas nunca ficam acopladas a Controllers, SQL bruto ou services do NestJS. Todo indicador tem sua função pura em `modules/indicators/calculators/` e seu teste unitário em `.spec.ts`.

### 1. Teste Unitário Primeiro (`modules/indicators/calculators/ipd.calculator.spec.ts`)

```typescript
import { calculateIpd } from "./ipd.calculator";

describe("calculateIpd (IND-OPE-08)", () => {
  it("calculates water loss ratio from produced, discharge and micrometered volumes", () => {
    const result = calculateIpd({
      producedVolumeM3: 1000,
      networkDischargeM3: 50,
      micrometeredVolumeM3: 650,
    });

    expect(result).toBeCloseTo(0.3, 6);
  });

  it("returns null when produced volume is zero to avoid division by zero", () => {
    const result = calculateIpd({
      producedVolumeM3: 0,
      networkDischargeM3: 0,
      micrometeredVolumeM3: 0,
    });

    expect(result).toBeNull();
  });

  it("throws when any volume input is negative", () => {
    expect(() =>
      calculateIpd({
        producedVolumeM3: 1000,
        networkDischargeM3: -10,
        micrometeredVolumeM3: 500,
      }),
    ).toThrow("Volumes must be non-negative");
  });
});
```

### 2. Implementação Pura (`modules/indicators/calculators/ipd.calculator.ts`)

```typescript
export interface IpdInput {
  readonly producedVolumeM3: number;
  readonly networkDischargeM3: number;
  readonly micrometeredVolumeM3: number;
}

export function calculateIpd(input: IpdInput): number | null {
  const { producedVolumeM3, networkDischargeM3, micrometeredVolumeM3 } = input;

  if (
    producedVolumeM3 < 0 ||
    networkDischargeM3 < 0 ||
    micrometeredVolumeM3 < 0
  ) {
    throw new Error("Volumes must be non-negative");
  }

  if (producedVolumeM3 === 0) {
    return null;
  }

  return (
    (producedVolumeM3 - networkDischargeM3 - micrometeredVolumeM3) /
    producedVolumeM3
  );
}
```

### 3. Controller Enxuto (`modules/indicators/indicators.controller.ts`)

```typescript
import { Controller, Get, Query } from "@nestjs/common";
import { ApiOperation, ApiTags } from "@nestjs/swagger";
import type { IndicatorSeriesResponse } from "@hidroforte/contracts";
import { IndicatorsService } from "./indicators.service";
import { QueryIndicatorDto } from "./dto/query-indicator.dto";

@ApiTags("Indicators")
@Controller("v1/indicators")
export class IndicatorsController {
  constructor(private readonly indicatorsService: IndicatorsService) {}

  @Get()
  @ApiOperation({
    summary: "Consulta indicadores consolidados por período e filtros",
  })
  findByFilters(
    @Query() query: QueryIndicatorDto,
  ): Promise<IndicatorSeriesResponse> {
    return this.indicatorsService.findByFilters(query);
  }
}
```
