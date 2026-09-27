# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadata

- **Autor:** Larissa (Tech Lead)
- **Status:** Proposed
- **Data:** 2026-09-27
- **Revisores:** Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)

---

## TL;DR

A plataforma passará a notificar clientes B2B em tempo real (abaixo de 10
segundos) sobre mudanças de status dos seus pedidos, por meio de **webhooks
outbound** entregues de forma assíncrona. A solução usa o **padrão outbox no
MySQL existente**: quando o status muda, um evento é gravado na mesma transação
da mudança; um **worker em processo separado** faz polling e dispara as chamadas
HTTP com assinatura **HMAC-SHA256**, secret **por endpoint**, retry com backoff
e **DLQ** para falhas permanentes. A garantia de entrega é **at-least-once**,
com `X-Event-Id` para deduplicação do lado do cliente. A feature segue os
padrões já existentes do projeto (módulos, `AppError`, Pino, schemas Zod, error
middleware centralizado).

---

## Contexto e Problema

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** —
fizeram um pedido formal para serem notificados em tempo real quando o status
dos pedidos deles muda na plataforma ([09:00] Marcos). Hoje eles fazem polling
no `GET /orders`, integração lenta e cara ([09:00] Marcos). Para eles, "tempo
real" é qualquer coisa abaixo de 10 segundos; o essencial é não ficar
pendurado em atualização manual ([09:02] Marcos). A **Atlas** chegou a sinalizar
que, sem a feature até o fim do trimestre, pode migrar para o concorrente
([09:00] Marcos); o prazo alvo é o fim de novembro ([09:45] Marcos).

A comunicação é **somente outbound** (da plataforma para os clientes), o que
simplifica o escopo ([09:02] Marcos, [09:03] Sofia). A aplicação atual não tem
nenhum mecanismo de notificação externa, eventos ou filas — este vácuo é
exatamente o que esta proposta preenche. A mudança de status de pedido já é uma
transação pesada (pedido + histórico + estoque), então a notificação não pode
entrar no caminho síncrono ([09:04] Bruno).

---

## Proposta Técnica

A proposta tem quatro componentes, todos dentro da infraestrutura já existente:

1. **Outbox transacional.** Quando `changeStatus` altera o pedido, o mesmo
   cliente de transação também grava o evento em `webhook_outbox`. Commit da
   transação ⇒ evento gravado; rollback ⇒ evento descartado. Não existe mudança
   de status sem evento ([09:06] Diego, [09:40]–[09:41] Bruno). O evento já sai
   renderizado (snapshot) e filtrado pelos status que cada endpoint quer ouvir
   ([09:34] Bruno, [09:52] Larissa).
2. **Worker de entrega.** Um processo Node separado faz polling a cada 2
   segundos dos eventos pendentes, dispara o HTTP para o endpoint do cliente e
   marca o resultado. A latência mínima é ~2s, dentro do requisito de 10s
   ([09:09]–[09:10] Diego, Larissa).
3. **Confiabilidade de entrega.** Retry com backoff exponencial em 5 tentativas
   (1m/5m/30m/2h/12h); após esgotar, o evento vai para uma **DLQ** em tabela
   separada, com replay manual por endpoint admin restrito a role ADMIN
   ([09:15]–[09:18] Diego, [09:36] Sofia).
4. **Segurança e idempotência.** Assinatura HMAC-SHA256 do corpo no header
   `X-Signature`, secret única por endpoint com rotação e grace period de 24h,
   TLS obrigatório; cada evento carrega `X-Event-Id` único para dedup do lado
   do cliente, com garantia at-least-once ([09:19]–[09:26] Sofia, Diego).

O webhook é implementado como um módulo padrão do projeto (`src/modules/webhooks`)
com controller, service, repository, routes e schemas, reusando `AppError`,
Pino, o middleware de erro centralizado, o `validate` middleware e o
`requireRole` ([09:27]–[09:30] Bruno, Larissa). O FDD detalha cada contrato e
ponto de integração.

---

## Alternativas Consideradas

- **Disparo síncrono no service de orders.** Levantada por Larissa ([09:03]),
  refutada por Bruno ([09:04]): um HTTP call no meio da transação de mudança de
  status faria cliente lento travar mudanças de outros pedidos, e não há rollback
  possível se o cliente estiver fora do ar. **Trade-off:** simplicidade imediata
  vs. acoplamento e fragilidade sob falha do cliente. *[09:04] Bruno*

- **Redis Streams como fila.** Proposta por Diego ([09:07]): exigiria subir
  Redis Cluster, overengineering para um time pequeno; o outbox no MySQL
  existente resolve sem infra nova. **Trade-off:** flexibilidade de broker
  dedicado vs. custo operacional e complexidade desproporcionais ao volume.
  *[09:07] Diego*

- **Trigger de banco para reatividade.** Perguntada por Bruno ([09:09]):
  MySQL não tem listener nativo (NOTIFY/LISTEN); trigger só executa SQL e não
  notifica um processo externo. **Trade-off:** reatividade sem polling vs.
  improvisação frágil (arquivo/endpoint) para preencher a lacuna. *[09:09] Diego*

- **Exactly-once.** Descartada por Diego ([09:25]): exigiria coordenação dos
  dois lados e complexidade muito maior; at-least-once com `event_id` resolve
  99% dos casos e é o padrão do mercado (Stripe, GitHub). **Trade-off:**
  perfeição de entrega vs. complexidade; o requisito real dos clientes é apenas
  saber se o pedido deles mudou. *[09:25] Diego, [09:14] Marcos*

---

## Questões em Aberto

- **Rate limiting de saída para clientes.** Se um cliente tem 50 pedidos
  mudando em um minuto, a plataforma bombardeia o endpoint dele com 50 chamadas?
  Diego propôs **observar e decidir depois**, fora do escopo atual ([09:39] Diego,
  [09:39] Larissa).
- **Escala futura do worker.** Garantia de ordering é por `order_id` e apenas
  enquanto single-worker; múltiplos workers em paralelo no futuro exigiriam
  particionamento por `order_id` ou lock pessimista. Adiado, não decidido
  ([09:13] Diego).
- **Fallback por email em caso de falha.** Avisar o cliente por email quando o
  webhook dele falhar 3 vezes seguidas foi pedido pelo PM, mas ficou para a
  **próxima fase**, depois de medir o impacto ([09:37] Marcos, [09:37] Larissa).
- **Arquivamento de eventos entregues.** Linhas entregues da outbox são
  arquivadas depois de 30 dias — fora do escopo da feature, mas decisão
  operacional pendente ([09:08] Diego).

---

## Impacto e Riscos

- **Correção:** o ponto crítico é a atomicidade — evento e mudança de status
  precisam commitar juntos. Fora da transação, a garantia toda se perde
  ([09:41] Diego).
- **Segurança:** exposição de dados de pedido fora da infra; mitigado por
  HMAC-SHA256, secret por endpoint, rotação com grace period e TLS obrigatório
  ([09:19]–[09:23] Sofia).
- **Operação:** dois processos Node a monitorar (API + worker); DLQ precisa de
  alerta e política de reprocessamento; eventos entregues acumulam até arquivar
  ([09:08] Diego).
- **Clientes:** duplicidade esperada (at-least-once) — exige documentação clara
  no portal de desenvolvedor ([09:26] Marcos).
- **Prazo:** três sprints, com revisão de segurança da Sofia reservada antes do
  deploy ([09:46] Larissa, [09:46] Sofia).

---

## Decisões Relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2s](adrs/ADR-002-worker-processo-separado-polling-2s.md)
- [ADR-003 — Política de retry com backoff e DLQ](adrs/ADR-003-politica-retry-backoff-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Entrega at-least-once com `X-Event-Id`](adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Payload snapshot renderizado na inserção](adrs/ADR-007-payload-snapshot-renderizado-na-insercao.md)
