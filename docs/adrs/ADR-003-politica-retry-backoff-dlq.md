# ADR-003 — Política de retry com backoff exponencial e DLQ em tabela separada

## Status

Accepted

## Contexto

O cliente de um webhook pode ficar temporariamente fora do ar. O worker precisa
de uma política de retry com backoff, de um teto de tentativas e de um destino
final para eventos que nunca puderam ser entregues, com evidência suficiente
para debug e reprocessamento ([09:15] Diego, [09:18] Diego).

## Decisão

- **Retry com backoff exponencial de 5 tentativas no total**, na progressão
  **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** — cerca de 15 horas
  entre a primeira falha e a última tentativa ([09:15]–[09:17] Diego).
- Após esgotar as tentativas, o evento é considerado **falha permanente** e
  movido para uma tabela separada **`webhook_dead_letter`**, que guarda a
  payload, o motivo da falha e o timestamp. Isso mantém a leitura da outbox
  principal limpa e serve como evidência para debug ([09:18] Diego).
- **Replay manual** via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`,
  que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint
  exige **role ADMIN** e registra quem fez o replay para auditoria
  ([09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

- **3 tentativas (mais agressivo)** — proposta por Bruno ([09:16]); descartada
  por Diego ([09:16]): três tentativas em cerca de 30 minutos não cobririam uma
  indisponibilidade de duas horas em manutenção planejada, cenário já vivido com
  cliente real.
- **Retry indefinido com backoff** — mencionada por Diego ([09:15]) como
  posição que existe no mercado; descartada porque o evento ficaria pendurado
  para sempre se o cliente sumisse, sem encerramento do ciclo de vida.

## Consequências

**Positivas:**
- Cobre indisponibilidades de até ~15 horas, aceitável para o negócio
  ([09:17] Marcos).
- DLQ em tabela própria dá leitura limpa da outbox e evidência para
  reprocessamento e auditoria ([09:18] Diego).
- Replay controlado por ADMIN com log de auditoria ([09:36] Sofia).

**Negativas:**
- Janela longa de retry pode atrasar notificações por várias horas quando o
  cliente está fora do ar ([09:17] Marcos).
- Como a entrega é at-least-once (ADR-005), um replay na DLQ pode gerar
  duplicidade adicional que o cliente precisa deduplicar.
- Tabela de dead-letter acumula dados que precisam de política de retenção.
