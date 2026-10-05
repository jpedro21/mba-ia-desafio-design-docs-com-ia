---
name: doc-generator
description: >
  This skill should be used when the user asks to "generate design docs",
  "produce documentation from a transcript", "generate the PRD",
  "generate the RFC", "generate the FDD", "generate ADRs", "create ADRs",
  "build the tracker", "generate the full doc package", "document this
  feature from the transcript", or any similar request involving the
  production or refinement of structured technical documents (PRD, RFC, FDD,
  ADRs, Tracker) from a meeting transcript file and an existing codebase.
version: 1.0.0
---

# doc-generator

Generate a complete, traceable package of technical design documents from a
meeting transcript and the existing codebase. The five document types you
produce — PRD, RFC, FDD, ADRs, and Tracker — together form a self-consistent
set where every requirement, decision, and risk can be traced back to either a
line in the transcript or a real file in the repository.

**Ground rule:** never invent requirements, decisions, constraints, or file
paths. Everything you write must be attributable to the transcript or to a
confirmed source file. When in doubt, remove the item rather than fabricate a
source.

---

## Phase 0 — Ground yourself before writing anything

Execute these reads in order and do not skip them.

1. **Locate and read the transcript in full.** The default is `TRANSCRICAO.md`
   at the project root, but if the user names a different file, use that one.
   As you read, extract and catalogue:
   - Every **closed decision** — a point a participant restates as final, or
     that the tech lead sums up as settled.
   - Every **discarded or deferred item** — things explicitly ruled out or
     postponed.
   - Every **open question** left unresolved by the end of the meeting.
   - Every **functional and non-functional requirement** raised.
   - The **participant list** with their roles (these become the RFC reviewers).
   - Any **time anchors** (`[hh:mm]`) that let you cite where each item came from.

2. **Map the real source files.** Run `find src -type f | sort` (adjust the
   directory if the code lives elsewhere). Identify the key integration points
   you may need to cite: the main modules, the error/exception layer, the
   middleware, the logger, the database configuration, and the entry points.
   Only cite paths that appear in this listing.

3. **Confirm the output layout.** Check where documents should land (default
   `docs/`), whether an `adrs/` subdirectory already exists, and follow any
   naming convention already present in the project.

---

## Phase 1 — Produce ADRs first (5–8 files)

ADRs are the skeleton the other documents hang from. Create them before the
RFC or FDD.

- Each ADR covers **exactly one** architectural decision raised in the
  transcript. One decision per file, one file per decision.
- File naming: `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`, numbered
  sequentially starting at `001`.
- Each ADR must contain, in order, the sections: `Status`, `Contexto`,
  `Decisão`, `Alternativas Consideradas`, `Consequências`.
  - `Status`: one of `Accepted`, `Proposed`, `Deprecated`, `Superseded`.
  - `Contexto`: the forces that made the decision necessary.
  - `Decisão`: the chosen approach, stated plainly.
  - `Alternativas Consideradas`: at least one real alternative that was raised
    and rejected in the transcript, with the reason it lost and who proposed it.
  - `Consequências`: the positive **and** negative consequences; state the
    trade-off explicitly.
- At least one ADR must reference one or more real file paths to show where the
  decision lands in the codebase.

Target 5–8 ADRs covering the major decisions the group reached. If the
transcript surfaces more than eight, pick the eight most consequential.

---

## Phase 2 — Produce the RFC (architecture level only)

Write `docs/RFC.md`. The RFC operates one level of abstraction above
implementation detail. **Do not** include SQL DDL, code snippets, or error
codes — those belong in the FDD.

Required sections, in order:

1. **Metadata** — a small block: `Autor` (the tech lead from the transcript),
   `Status` (usually `Proposed`), `Data` (current date), and `Revisores`
   (every other participant from the meeting).
2. **TL;DR** — one paragraph: what the system will do and the single headline
   approach.
3. **Contexto e Problema** — the business/technical problem motivating the work,
   with the stakeholders and constraints named in the transcript.
4. **Proposta Técnica** — the architecture at a high level: the components and
   how they relate. No implementation detail.
5. **Alternativas Consideradas** — at least two discarded alternatives, each
   with the trade-off that drove the rejection and a citation (timestamp +
   speaker).
6. **Questões em Aberto** — at least two unresolved questions, each citing where
   in the transcript it was left open.
7. **Impacto e Riscos** — the operational and correctness implications.
8. **Decisões Relacionadas** — relative Markdown links to at least two ADRs
   produced in Phase 1.

---

## Phase 3 — Produce the FDD (implementation specification)

Write `docs/FDD.md`. This is the most detailed document — a developer should be
able to start coding from it.

Required sections, in order:

1. **Contexto e Motivação Técnica**
2. **Objetivos Técnicos**
3. **Escopo e Exclusões**
4. **Fluxos Detalhados** — at least three numbered, sequential flows covering
   the happy path and the failure paths mentioned in the transcript.
5. **Contratos Públicos** — at least four HTTP endpoints, each with: method +
   path, request headers, request body example (JSON), response body example
   (JSON), and the HTTP status codes for success and each error case.
6. **Matriz de Erros** — every error code prefixed with the prefix agreed in
   the transcript (e.g. `WEBHOOK_`). For each, give the HTTP status code and
   the trigger condition.
7. **Estratégias de Resiliência** — timeouts, retries, backoff, and dead-letter
   behaviour as decided.
8. **Observabilidade** — name the project's logging library (e.g. Pino from the
   codebase), list at least three structured log fields per event, and name at
   least three metrics to expose.
9. **Dependências e Compatibilidade**
10. **Critérios de Aceite Técnicos**
11. **Riscos e Mitigação**
12. **Integração com o Sistema Existente** — **mandatory**: name at least four
    real file paths (from `src/`, or the equivalent source directory), each with
    a paragraph explaining exactly how the new component integrates with the
    code already present in that file. Every path must be confirmed in Phase 0.

---

## Phase 4 — Produce the PRD (product/business view)

Write `docs/PRD.md` after the RFC and FDD. At this point the PRD is a synthesis
at product/business level, pulling from the technical decisions you have already
captured.

Required sections, in order:

1. **Resumo e Contexto**
2. **Problema e Motivação**
3. **Público-Alvo e Cenários de Uso**
4. **Objetivos e Métricas de Sucesso** — at least one objective with a numeric
   target and a measurement method.
5. **Escopo (Incluso / Fora de Escopo)** — the "Fora de Escopo" list must name
   at least two items explicitly discarded or deferred in the transcript.
6. **Requisitos Funcionais** — at least eight items, each with an ID
   (`FR-01`, `FR-02`, …).
7. **Requisitos Não Funcionais**
8. **Decisões e Trade-offs**
9. **Dependências**
10. **Riscos e Mitigação** — at least two entries, each with probability
    (Alta/Média/Baixa), impact, and a mitigation strategy.
11. **Critérios de Aceitação**
12. **Estratégia de Testes**

Every item in the PRD must be traceable to the transcript.

---

## Phase 5 — Produce the Tracker (cross-cutting traceability)

Write `docs/TRACKER.md` as a Markdown table with these exact columns:

```
ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização
```

Rules:

- `Fonte` is either `TRANSCRICAO` or `CODIGO` — nothing else.
- For `TRANSCRICAO` rows, `Localização` is `[hh:mm] NomeFalante`.
- For `CODIGO` rows, `Localização` is a relative file path from the repo root.
- `Tipo` is one of: `Requisito Funcional`, `Requisito Não Funcional`, `Decisão`,
  `Restrição`, `Trade-off`, `Questão em Aberto`, `Risco`.
- `ID` follows a `DOC-TIPO-NN` pattern (e.g. `PRD-FR-01`, `RFC-ALT-02`,
  `FDD-CONTRATO-03`, `ADR-001`).
- Coverage: at least 80% of the identifiable items across all documents must
  have a corresponding row.
- At least 70% of rows must be `TRANSCRICAO` with a valid timestamp.
- At least 5 rows must be `CODIGO` with real, verified file paths.

---

## Quality gates — run before declaring any document complete

Before you mark a document done, check it against
`references/quality-checklist.md` and fix every failed item. The minimum bars:

- **PRD**: 8+ functional requirements with IDs, 1+ numeric metric, 2+
  out-of-scope items from the transcript, 2+ risks.
- **RFC**: complete metadata with all reviewers, 2+ discarded alternatives with
  citations, 2+ open questions, 2+ relative ADR links, no DDL/code/error codes.
- **FDD**: 4+ endpoints with full payload examples, all error codes with the
  agreed prefix, "Integração com o Sistema Existente" naming 4+ real paths,
  observability with the logging library named.
- **ADRs**: 5–8 files, each with all five sections, covering the major decisions,
  at least one citing a real file path.
- **Tracker**: 80%+ coverage, 70%+ TRANSCRICAO rows with timestamps, 5+ CODIGO
  rows with verified paths.

Any item you cannot attribute to the transcript or to a confirmed source file
must be rewritten or removed.

---

## Reference files

- `references/document-schemas.md` — section-by-section schema for all five
  document types with content constraints and allowed values.
- `references/quality-checklist.md` — binary checklist for each document type,
  aligned with the acceptance criteria above.
