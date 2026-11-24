# AGENTS.md — Guia Global do Monorepo

Fonte única de verdade para desenvolvedores e agentes de IA neste repositório. Para detalhes específicos de implementação, leia também `apps/api/AGENTS.md` e `apps/web/AGENTS.md`.

## Comandos do Monorepo

Sempre use **`pnpm`** (nunca `npm`, `yarn` ou `bun`).

```bash
pnpm dev                         # Roda API e Web em paralelo
pnpm build                       # Compila contratos + apps com checagem de tipos
pnpm lint                        # Lint em todos os workspaces
pnpm format                      # Formatação em todos os workspaces
pnpm test                        # Todos os testes unitários
pnpm --filter api test           # Testes unitários apenas do Backend
pnpm --filter api test:e2e       # Testes de integração/E2E do Backend
pnpm --filter web test           # Testes unitários apenas do Frontend
```

## Regras Arquiteturais Inegociáveis

1. **Monólito Modular no padrão nativo do NestJS:** sem camadas artificiais ou overengineering. Aplique inversão de dependência (SOLID) onde há variação real (adaptadores de fontes externas) e funções puras onde há regra matemática crítica (cálculo de indicadores).
2. **Nenhuma regra de negócio no Frontend:** o Frontend apenas envia filtros, consome JSON da API e formata a exibição.
3. **Nenhuma fórmula sem validação:** toda regra implementada em produção deve corresponder a um indicador aprovado em `docs/05. INDICATORS-CATALOG.md` (`IND-*`). Fórmulas ainda marcadas como _A validar_ só podem existir atrás de fixtures/testes de protótipo claramente sinalizados.
4. **Rastreabilidade obrigatória (Data Lineage):** todo registro ingerido de fonte externa deve preservar `sourceSystem`, `externalId`, `extractedAt` e `executionId`.
5. **Falha segura na ingestão:** erros em uma carga externa nunca podem sobrescrever ou corromper dados consolidados válidos de execuções anteriores.
6. **Sistemas de origem são somente leitura:** a plataforma nunca escreve nem altera dados em bancos ou APIs externas.

## Fronteiras do Monorepo

```text
apps/web ──▶ packages/contracts
apps/api ──▶ packages/contracts
```

- `packages/*` **nunca** importam código de `apps/*`.
- `apps/web` **nunca** importa código interno de `apps/api`.
- `packages/contracts` contém apenas tipos/DTOs de fronteira (requisição, resposta e paginação), sem dependências de runtime de banco ou framework.

## Padrões de Código (Bloqueantes em PR)

- **TypeScript `strict: true`:** proibido `any` e type assertions (`as`) desnecessárias. Use `import type` para tipos.
- **Fallback seguro:** use `??` (nullish coalescing) para valores padrão; reserve `||` apenas para expressões booleanas.
- **Legibilidade:** sem ternários aninhados; prefira `if/else` explícito ou funções auxiliares pequenas.
- **Código limpo:** evite comentários inline óbvios que apenas repetem o que o código já expressa. Nomes de variáveis, funções e tipos devem carregar a intenção.
- **Erros HTTP padronizados:** a API retorna erros no formato RFC 7807 (`ProblemDetail`) sem expor stack traces ou segredos em produção.
- **Segredos:** jamais hardcode senhas, tokens ou URLs internas. Toda configuração específica de ambiente deve ser validada na inicialização via variáveis de ambiente.

## Fluxo Obrigatório de TDD para Indicadores

1. Escreva primeiro o teste unitário da função pura de cálculo usando os valores de referência validados pela área de negócio (incluindo divisão por zero, valores negativos e dados ausentes).
2. Execute o teste e confirme a falha.
3. Implemente o cálculo mínimo para passar.
4. Refatore mantendo os testes verdes.

## Checklist Antes de Concluir Qualquer Tarefa

- [ ] `pnpm lint` passou sem avisos ou erros.
- [ ] `pnpm build` compilou todos os workspaces sem erros de tipo.
- [ ] `pnpm test` passou integralmente (incluindo novos testes para cada regra ou adaptador alterado).
- [ ] Alterações arquiteturais foram refletidas em `docs/02. ARCHITECTURE.md` ou `docs/04. ADR.md`.
- [ ] Commits seguem Conventional Commits (`feat:`, `fix:`, `test:`, `refactor:`, `docs:`, `chore:`).
