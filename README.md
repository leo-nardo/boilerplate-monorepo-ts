# Plataforma de Integração e Indicadores — Hidroforte

Monorepo da plataforma centralizada de integração, padronização, cálculo e visualização de indicadores comerciais, financeiros, operacionais, de produtividade e de energia.

## Stack

- **Monorepo:** pnpm Workspaces + TypeScript (strict)
- **Backend (`apps/api`):** NestJS + PostgreSQL
- **Frontend (`apps/web`):** React + Vite + Tailwind CSS + TanStack Query
- **Infraestrutura local:** Docker + Docker Compose

## Estrutura do Repositório

```text
.
├── apps/
│   ├── api/                  # API REST, ingestão, padronização e cálculo de indicadores
│   └── web/                  # Interface web de dashboards e filtros
├── packages/
│   ├── contracts/            # Tipos e contratos compartilhados entre API e Web
│   └── config/               # Configurações compartilhadas (tsconfig, lint, format)
├── infrastructure/
│   └── docker/               # Dockerfiles e docker-compose local
├── docs/                     # Especificação, arquitetura (arc42), ADRs e catálogo
├── .ai/documents/            # Documentos vivos por entrega
├── AGENTS.md                 # Guia operacional global do monorepo
└── package.json
```

## Pré-requisitos

- Node.js 22 LTS
- pnpm 9+
- Docker e Docker Compose

## Configuração de Ambiente

Copie os arquivos `.env.example` para `.env` em `apps/api` e `apps/web`. Credenciais reais nunca devem ser versionadas.

### Backend (`apps/api/.env`)

| Variável          | Exemplo                                                    | Descrição                                                  |
| ----------------- | ---------------------------------------------------------- | ---------------------------------------------------------- |
| `PORT`            | `3000`                                                     | Porta HTTP da API                                          |
| `NODE_ENV`        | `development`                                              | Ambiente de execução (`development`, `test`, `production`) |
| `DATABASE_URL`    | `postgresql://postgres:postgres@localhost:5432/indicators` | String de conexão PostgreSQL                               |
| `CORS_ORIGIN`     | `http://localhost:5173`                                    | Origem permitida para o Frontend                           |
| `AUTH_ISSUER_URL` | `A definir`                                                | Provedor de autenticação conforme política da TI           |

### Frontend (`apps/web/.env`)

| Variável            | Exemplo                    | Descrição       |
| ------------------- | -------------------------- | --------------- |
| `VITE_API_BASE_URL` | `http://localhost:3000/v1` | URL base da API |

## Como Executar Localmente

```bash
pnpm install
docker compose -f infrastructure/docker/docker-compose.yml up -d
pnpm --filter api migration:run
pnpm dev
```

- **Frontend:** `http://localhost:5173`
- **API:** `http://localhost:3000/v1`
- **Swagger / OpenAPI:** `http://localhost:3000/docs`

## Comandos Principais

```bash
pnpm dev                  # Sobe API e Web em modo desenvolvimento
pnpm build                # Typecheck e build de todos os pacotes e apps
pnpm lint                 # Lint em todo o monorepo
pnpm format               # Formatação em todo o monorepo
pnpm test                 # Executa testes unitários em todo o monorepo
pnpm test:cov             # Executa testes com relatório de cobertura
```

## Documentação de Engenharia

- `docs/01. PROJECT-SPEC.md` — Requisitos funcionais e não funcionais
- `docs/02. ARCHITECTURE.md` — Arquitetura (arc42)
- `docs/03. DEVELOPMENT-STANDARDS.md` — Padrões de código e testes
- `docs/04. ADR.md` — Decisões arquiteturais
- `docs/05. INDICATORS-CATALOG.md` — Catálogo oficial de indicadores e fórmulas
- `docs/06. DISCOVERY-BACKLOG.md` — Pendências de levantamento com TI e negócio
- `docs/07. GLOSSARY.md` — Glossário de negócio e técnico
