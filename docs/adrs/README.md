# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual,
nomeado sequencialmente no formato `ADR-NNN-titulo-em-kebab-case.md`
(ex.: `ADR-001-outbox-no-mysql.md`).

Conjunto atual (7 ADRs):

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-processo-separado-polling-2s.md) | Worker em processo separado com polling de 2s |
| [ADR-003](ADR-003-politica-retry-backoff-dlq.md) | Política de retry com backoff e DLQ |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-entrega-at-least-once-com-event-id.md) | Entrega at-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-payload-snapshot-renderizado-na-insercao.md) | Payload snapshot renderizado na inserção |

Cada ADR segue as seções: `Status`, `Contexto`, `Decisão`, `Alternativas
Consideradas` e `Consequências`.
