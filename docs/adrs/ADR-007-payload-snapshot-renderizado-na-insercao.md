# ADR-007 — Payload do evento como snapshot renderizado na inserção na outbox

## Status

Accepted

## Contexto

Ao enfileirar o evento na outbox, restava uma decisão de modelagem: a linha da
outbox deveria guardar o **payload renderizado** já, ou apenas o `order_id`,
renderizando o payload **na hora do envio** pelo worker ([09:51] Bruno).

## Decisão

O evento guarda o **payload renderizado como snapshot, no momento da inserção**
na outbox ([09:52] Larissa). O payload JSON contém: `event_id`, `event_type`
(`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`,
`from_status`, `to_status`, `customer_id` e `total_cents`. Não inclui os itens,
para manter o payload enxuto; se o cliente quiser detalhes, consulta o
`GET /orders/:id` depois ([09:43] Diego).

## Alternativas Consideradas

- **Guardar só o `order_id` e renderizar o payload na hora do envio** —
  perguntada por Bruno ([09:51]); descartada por Larissa ([09:52]): se o pedido
  mudar depois, o evento refletiria o estado atual em vez do estado no momento
  da mudança de status, criando caso esquisito. Diego concorda: "snapshot na
  inserção" ([09:52] Diego).

## Consequências

**Positivas:**
- Fidelidade histórica: o evento reflete o estado do pedido no instante em que o
  status mudou, mesmo que o pedido evolua depois ([09:52] Larissa).
- Payload enxuto e estável, previsível para integração dos clientes
  ([09:43] Diego).
- Consistente com o padrão outbox: o snapshot preserva a semântica do momento
  ([09:52] Diego).

**Negativas:**
- O payload pode ficar defasado em relação ao estado atual do pedido
  (intencional, mas exige documentação).
- Reproduzir/re-renderizar um evento antigo exige reprocessamento, não basta
  ler a linha da outbox.
