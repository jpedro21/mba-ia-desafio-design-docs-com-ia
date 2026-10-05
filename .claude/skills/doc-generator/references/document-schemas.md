# Document Schemas

Section-by-section contracts for each document type. Sections must appear in
the order given. A "≥ N" constraint is a minimum; exceed it only when the
transcript genuinely supports more.

---

## PRD (`docs/PRD.md`)

Sections, in order:

1. **Resumo e Contexto**
2. **Problema e Motivação**
3. **Público-Alvo e Cenários de Uso**
4. **Objetivos e Métricas de Sucesso** — ≥ 1 objective with a numeric target
   and a stated measurement method.
5. **Escopo (Incluso / Fora de Escopo)** — `Fora de Escopo` lists ≥ 2 items
   explicitly discarded or deferred in the transcript.
6. **Requisitos Funcionais** — ≥ 8 items, each with an ID (`FR-01`, `FR-02`, …).
7. **Requisitos Não Funcionais**
8. **Decisões e Trade-offs**
9. **Dependências**
10. **Riscos e Mitigação** — ≥ 2 entries, each with probability
    (Alta/Média/Baixa), impact, and mitigation.
11. **Critérios de Aceitação**
12. **Estratégia de Testes**

Constraints:

- Every requirement must trace to a line in the transcript.
- `Riscos` uses the probability scale Alta / Média / Baixa.

---

## RFC (`docs/RFC.md`)

Sections, in order:

1. **Metadata** — block with:
   - `Autor`: the tech lead from the transcript.
   - `Status`: `Proposed` (default).
   - `Data`: current date.
   - `Revisores`: every other participant from the meeting.
2. **TL;DR**
3. **Contexto e Problema**
4. **Proposta Técnica**
5. **Alternativas Consideradas** — ≥ 2, each with the rejection trade-off and a
   `[hh:mm] NomeFalante` citation.
6. **Questões em Aberto** — ≥ 2, each citing where it was left open.
7. **Impacto e Riscos**
8. **Decisões Relacionadas** — relative Markdown links to ≥ 2 ADR files.

Constraints:

- Architecture level only: **no** SQL DDL, no code snippets, no error codes.
- Target 2–4 pages.

---

## FDD (`docs/FDD.md`)

Sections, in order:

1. **Contexto e Motivação Técnica**
2. **Objetivos Técnicos**
3. **Escopo e Exclusões**
4. **Fluxos Detalhados** — ≥ 3 numbered sequential flows (happy path + failure
   paths named in the transcript).
5. **Contratos Públicos** — ≥ 4 HTTP endpoints. Each endpoint must state:
   - method + path
   - request headers
   - request body example (JSON)
   - response body example (JSON)
   - HTTP status codes for success and for each error case
6. **Matriz de Erros** — every code prefixed with the agreed prefix (e.g.
   `WEBHOOK_`). For each: HTTP status code + trigger condition.
7. **Estratégias de Resiliência**
8. **Observabilidade** — logging library named (e.g. Pino), ≥ 3 structured log
   fields per event, ≥ 3 metrics.
9. **Dependências e Compatibilidade**
10. **Critérios de Aceite Técnicos**
11. **Riscos e Mitigação**
12. **Integração com o Sistema Existente** — ≥ 4 real file paths, each with a
    paragraph describing the integration point.

Constraints:

- All file paths must be confirmed in Phase 0 (`find src -type f`).
- The error-code prefix is derived from the transcript, not invented.

---

## ADRs (`docs/adrs/ADR-NNN-kebab-case.md`)

Sections, in order:

1. **Status** — one of `Accepted`, `Proposed`, `Deprecated`, `Superseded`.
2. **Contexto**
3. **Decisão**
4. **Alternativas Consideradas** — ≥ 1 real rejected alternative, with reason
   and proposer.
5. **Consequências** — positive AND negative consequences, trade-off stated.

Constraints:

- One decision per file. 5–8 files total.
- Sequential numbering from `001`.
- ≥ 1 ADR cites ≥ 1 real source file path.

---

## Tracker (`docs/TRACKER.md`)

Table columns, in order:

```
ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização
```

Allowed values:

- `Fonte`: `TRANSCRICAO` or `CODIGO`.
- `Tipo`: `Requisito Funcional`, `Requisito Não Funcional`, `Decisão`,
  `Restrição`, `Trade-off`, `Questão em Aberto`, `Risco`.
- `Localização` for `TRANSCRICAO`: `[hh:mm] NomeFalante`.
- `Localização` for `CODIGO`: relative file path from repo root.

Constraints:

- Coverage ≥ 80% of identifiable items across all documents.
- ≥ 70% of rows are `TRANSCRICAO` with valid timestamps.
- ≥ 5 rows are `CODIGO` with verified paths.
