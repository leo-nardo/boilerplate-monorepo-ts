# {{Título da Entrega}}

> Documento vivo de uma demanda. Salvar em `.ai/documents/AAAA-MM-DD-slug-da-entrega.md`.

## Contexto

- **Projeto:** Plataforma de Integração e Indicadores — Hidroforte
- **Demanda / Issue:** #{{ID}} — {{título}}
- **Branch:** `{{feat|fix|refactor}}/{{slug}}`
- **Branch de destino:** `main`
- **PR:** {{URL ou pendente}}

## Rastreabilidade

- **Requisitos relacionados:** {{ex.: RF-001, RF-011, RNF-005}}
- **Indicadores impactados:** {{ex.: IND-OPE-08 ou N/A}}
- **Itens de Discovery fechados:** {{ex.: DSC-NEG-04 ou N/A}}

## Alinhamento e Entrevista de Requisitos

Registrar dúvidas levantadas com negócio ou TI antes de codificar:

- **Pergunta:** {{dúvida técnica ou de regra de negócio}}
- **Resposta:** {{retorno recebido e responsável}}
- **Decisão:** {{o que será implementado}}

## Definições Confirmadas

- {{regra de cálculo, fonte de dados, formato ou critério de aceite validado}}
- {{massa de referência fornecida pela área de negócio, se aplicável}}

## Decisões Técnicas da Entrega

- `AAAA-MM-DD`: {{decisão tomada e justificativa curta}}

## Progresso

- [ ] Escopo e critérios de aceite confirmados
- [ ] Regras de negócio validadas em `docs/05. INDICATORS-CATALOG.md` (quando houver cálculo)
- [ ] Contratos atualizados em `packages/contracts` (se aplicável)
- [ ] Testes unitários escritos primeiro (TDD) para fórmulas e regras críticas
- [ ] Implementação concluída seguindo `AGENTS.md`
- [ ] `pnpm lint`, `pnpm build` e `pnpm test` executados e verdes
- [ ] Documentação arquitetural/ADR atualizada (se houve decisão estrutural)
- [ ] PR aberto com contexto de entrega e evidências de validação

## Encerramento

- **PR publicado:** {{pendente / URL}}
- **Pendências deixadas para próximas entregas:** {{nenhuma ou listar com ID}}
