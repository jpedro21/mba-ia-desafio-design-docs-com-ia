# Quality Checklist

Run these gates before declaring a document complete. Each item is binary:
pass it or fix the document before moving on.

---

## PRD

- [ ] File `docs/PRD.md` exists and is valid Markdown.
- [ ] All 12 required sections are present, in order.
- [ ] `Requisitos Funcionais` lists ≥ 8 items, each with an ID.
- [ ] `Objetivos e Métricas` has ≥ 1 objective with a numeric target and a
      measurement method.
- [ ] `Fora de Escopo` lists ≥ 2 items explicitly discarded or deferred in the
      transcript.
- [ ] `Riscos` has ≥ 2 entries, each with probability (Alta/Média/Baixa),
      impact, and mitigation.

## RFC

- [ ] File `docs/RFC.md` exists and is valid Markdown.
- [ ] Metadata block present with `Autor`, `Status`, `Data`, and all meeting
      reviewers listed.
- [ ] All 8 required sections are present, in order.
- [ ] `Alternativas Consideradas` has ≥ 2 entries, each with a transcript
      citation (`[hh:mm] NomeFalante`).
- [ ] `Questões em Aberto` has ≥ 2 entries with transcript citations.
- [ ] `Decisões Relacionadas` links to ≥ 2 ADR files via relative paths.
- [ ] No SQL DDL, no code snippets, no error codes (FDD-level detail).

## FDD

- [ ] File `docs/FDD.md` exists and is valid Markdown.
- [ ] All 12 required sections are present, in order.
- [ ] `Contratos Públicos` has ≥ 4 HTTP endpoints, each with method + path,
      request headers, request body, response body, and status codes.
- [ ] Every code in `Matriz de Erros` uses the agreed prefix (e.g. `WEBHOOK_`)
      and has an HTTP status code + trigger condition.
- [ ] `Integração com o Sistema Existente` names ≥ 4 real file paths, each
      verified against the Phase 0 directory listing, each with a paragraph.
- [ ] `Observabilidade` names the logging library, ≥ 3 structured log fields,
      and ≥ 3 metrics.

## ADRs

- [ ] `docs/adrs/` contains 5–8 files named `ADR-NNN-kebab-case.md`.
- [ ] Each ADR has all five sections: `Status`, `Contexto`, `Decisão`,
      `Alternativas Consideradas`, `Consequências`.
- [ ] Every major decision from the transcript is covered by at least one ADR.
- [ ] ≥ 1 ADR references ≥ 1 real source file path.
- [ ] Every rejected alternative cites a real discussion point from the
      transcript (who proposed it and why it lost).

## Tracker

- [ ] File `docs/TRACKER.md` exists and is a valid Markdown table.
- [ ] Columns are exactly `ID | Documento | Tipo | Conteúdo (resumo) | Fonte |
      Localização`.
- [ ] Coverage ≥ 80% of the identifiable items across all documents.
- [ ] ≥ 70% of rows are `TRANSCRICAO` with valid `[hh:mm] NomeFalante` format.
- [ ] ≥ 5 rows are `CODIGO` with real, verified file paths.

---

## Consistency (global)

- [ ] Every file path cited anywhere exists in the source tree (verified in
      Phase 0).
- [ ] No document states a decision that contradicts the transcript.
- [ ] No requirement, constraint, or risk is present without a traceable source
      (transcript line or source file).
