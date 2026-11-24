# Frontend — Guia de Implementação (`apps/web`)

SPA em **React + TypeScript (strict) + Vite + Tailwind CSS** para consulta de dashboards, filtros por período/segmento e acompanhamento do status das integrações.

## Comandos

```bash
pnpm --filter web dev          # Vite dev server (http://localhost:5173)
pnpm --filter web build        # Typecheck (tsc -b) + Vite build
pnpm --filter web test         # Testes unitários (Vitest)
pnpm --filter web test:cov     # Cobertura de testes
pnpm --filter web preview      # Preview do build de produção
```

## Fluxo de Dados

```text
Page / Feature Component ──▶ TanStack Query Hook ──▶ API Service ──▶ httpClient ──▶ Backend (/v1/*)
                                                                           │
                                                                 @hidroforte/contracts
```

- **Estado de servidor:** TanStack Query (cache, refetch, estados `isLoading` e `isError`).
- **Estado de filtros de tela:** URL search params (permite compartilhar link filtrado) ou estado local tipado.
- **Regra estrita:** nenhuma fórmula de negócio ou agregação de indicador é calculada no navegador. O Frontend apenas formata números, moedas, percentuais e datas recebidos da API.

## Estrutura de Pastas

```text
apps/web/src/
├── features/
│   ├── dashboards/
│   │   ├── api/               # Chamadas HTTP tipadas com @hidroforte/contracts
│   │   ├── components/        # Cards de KPI, gráficos, barra de filtros
│   │   ├── hooks/             # Hooks TanStack Query
│   │   └── pages/             # Páginas de categoria (Comercial, Operacional, Financeiro)
│   └── integrations/          # Tela de status e histórico de execuções
├── shared/
│   ├── api/                   # Cliente HTTP base + parser RFC 7807 (ProblemDetail)
│   ├── components/            # Componentes de UI reutilizáveis (EmptyState, ErrorBanner, Card)
│   ├── formatters/            # Formatação PT-BR (moeda, m³, percentual, datas)
│   └── layout/                # Shell, navegação lateral e cabeçalho de página
├── router.tsx
└── main.tsx
```

## Convenções de UI e Código

- **Props imutáveis:** sempre declare `Readonly<Props>` nos componentes React.
- **Estados obrigatórios:** toda consulta de dashboard deve tratar explicitamente `loading` (skeleton), `error` (mensagem legível + botão tentar novamente) e `empty` (sem dados para o período selecionado).
- **Sem efeitos colaterais em `throwOnError`:** trate erros de query na UI pelo estado `error` ou via handlers dedicados; nunca dispare toasts dentro de funções puras de configuração.
- **Tokens semânticos:** use variáveis/classes semânticas do tema para cores, superfícies e bordas; evite cores hardcoded soltas nos componentes.
- **Formatação PT-BR centralizada:** use helpers em `shared/formatters/` (`Intl.NumberFormat("pt-BR")`) em vez de concatenar símbolos manualmente nos componentes.
- **Semântica HTML nativa:** prefira `<button>`, `<nav>`, `<main>`, `<output>` e `<fieldset>` a `<div>` com roles ARIA artificiais.

---

## Exemplo Canônico: Nova Tela de Indicadores

### 1. Service (`features/dashboards/api/indicators-api.ts`)

```typescript
import type {
  IndicatorFilterParams,
  IndicatorSeriesResponse,
} from "@hidroforte/contracts";
import { httpClient } from "@/shared/api/http-client";

export function fetchIndicators(
  params: Readonly<IndicatorFilterParams>,
): Promise<IndicatorSeriesResponse> {
  return httpClient.get<IndicatorSeriesResponse>("/indicators", {
    category: params.category,
    startPeriod: params.startPeriod,
    endPeriod: params.endPeriod,
    serviceType: params.serviceType,
  });
}
```

### 2. Query Hook (`features/dashboards/hooks/use-indicators.ts`)

```typescript
import { useQuery } from "@tanstack/react-query";
import type { IndicatorFilterParams } from "@hidroforte/contracts";
import { fetchIndicators } from "../api/indicators-api";

export function useIndicators(params: Readonly<IndicatorFilterParams>) {
  return useQuery({
    queryKey: ["indicators", params],
    queryFn: () => fetchIndicators(params),
    staleTime: 60_000,
  });
}
```

### 3. Componente de KPI (`features/dashboards/components/indicator-card.tsx`)

```typescript
interface IndicatorCardProps {
  title: string;
  formattedValue: string;
  referencePeriod: string;
  sourceLabel: string;
}

export function IndicatorCard({
  title,
  formattedValue,
  referencePeriod,
  sourceLabel,
}: Readonly<IndicatorCardProps>) {
  return (
    <article className="rounded-lg border border-border bg-card p-5">
      <h2 className="text-sm font-medium text-muted-foreground">{title}</h2>
      <p className="mt-2 text-2xl font-semibold tracking-tight text-foreground">
        {formattedValue}
      </p>
      <footer className="mt-3 flex items-center justify-between text-xs text-muted-foreground">
        <span>Período: {referencePeriod}</span>
        <span>Fonte: {sourceLabel}</span>
      </footer>
    </article>
  );
}
```

### 4. Página de Dashboard (`features/dashboards/pages/operational-dashboard-page.tsx`)

```typescript
import { useState } from "react";
import { formatPercentage } from "@/shared/formatters/number";
import { useIndicators } from "../hooks/use-indicators";
import { IndicatorCard } from "../components/indicator-card";

export function OperationalDashboardPage() {
  const [period, setPeriod] = useState({ startPeriod: "2026-01", endPeriod: "2026-11" });
  const { data, isLoading, isError, refetch } = useIndicators({
    category: "operational",
    startPeriod: period.startPeriod,
    endPeriod: period.endPeriod,
  });

  const items = data?.items ?? [];

  if (isLoading) {
    return <output className="block p-6 text-sm text-muted-foreground">Carregando indicadores...</output>;
  }

  if (isError) {
    return (
      <div className="rounded-lg border border-destructive/40 p-6">
        <p className="text-sm text-destructive">Não foi possível carregar os indicadores operacionais.</p>
        <button
          type="button"
          onClick={() => void refetch()}
          className="mt-3 rounded-md border border-border px-3 py-1.5 text-sm"
        >
          Tentar novamente
        </button>
      </div>
    );
  }

  if (items.length === 0) {
    return (
      <p className="p-6 text-sm text-muted-foreground">
        Nenhum dado consolidado encontrado para o período selecionado.
      </p>
    );
  }

  return (
    <main className="space-y-6 p-6">
      <header className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-xl font-semibold text-foreground">Indicadores Operacionais</h1>
        <input
          type="month"
          aria-label="Período final"
          value={period.endPeriod}
          onChange={(event) =>
            setPeriod((prev) => ({ ...prev, endPeriod: event.target.value }))
          }
          className="rounded-md border border-border bg-background px-3 py-1.5 text-sm"
        />
      </header>

      <section className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {items.map((item) => (
          <IndicatorCard !="=" "Sem : ? formatPercentage(item.value) formattedValue="{" item.value key="{item.id}" medição" null referencePeriod="{item.referencePeriod}" sourceLabel="{item.sourceSystem}" title="{item.label}" }/>
        ))}
      </section>
    </main>
  );
}
```
