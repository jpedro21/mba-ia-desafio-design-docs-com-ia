# ADR-001 — Padrão Outbox no MySQL para notificação de pedidos

## Status

Accepted

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser
notificados em tempo real — definido como "qualquer coisa abaixo de 10 segundos"
([09:02] Marcos) — quando o status dos pedidos deles muda na plataforma. Hoje
eles fazem polling no `GET /orders`, o que é lento e caro para a integração
([09:00] Marcos).

A mudança de status de um pedido já é uma transação pesada: atualiza a `order`,
insere em `order_status_history` e ajusta `stock_quantity` dos produtos do pedido
([09:04] Bruno). Disparar um HTTP call dentro dessa transação travaria mudanças
de status de outros pedidos se o cliente estivesse lento, e seria impossível
fazer rollback da mudança de status se o cliente estivesse fora do ar
([09:04] Bruno).

## Decisão

Adotar o padrão **outbox no MySQL existente**. Quando o status do pedido muda,
dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`,
inserir uma linha numa tabela `webhook_outbox` com o evento. Um worker separado
lê essa tabela e dispara as chamadas HTTP ([09:06] Diego, [09:08] Larissa).

A garantia central é transacional: se a transação principal fez commit, o evento
foi registrado; se ela deu rollback, o evento some junto. Não há como existir
mudança de status sem evento correspondente ([09:06] Diego, [09:40]–[09:41]
Bruno). A tabela recebe índice nos campos de status (`pendente`, `processando`,
`falhou`, `entregue`) e em `created_at` para o worker ler apenas os pendentes em
batch pequeno ([09:08] Diego).

## Alternativas Consideradas

- **Disparo síncrono dentro do service de orders** — levantada por Larissa
  ([09:03]); refutada por Bruno ([09:04]): a transação de mudança de status já
  é pesada e um cliente lento travaria mudanças de status de outros pedidos;
  além disso, não há como dar rollback na mudança de status se o cliente estiver
  offline. Larissa concordou ([09:04]) e Diego reforçou ([09:06]).
- **Redis Streams (ou equivalente) para fila** — proposta por Diego ([09:07]):
  descartada porque exigiria subir infraestrutura nova (Redis Cluster), o que é
  overengineering para um time pequeno; o outbox no MySQL existente resolve o
  problema ([09:07] Diego).

## Consequências

**Positivas:**
- Consistência transacional garantida entre mudança de status e registro do
  evento — nunca há status alterado sem evento ([09:06] Diego).
- Nenhuma infraestrutura nova de mensageria; reuso do MySQL e do Prisma já
  existentes ([09:07] Diego).
- O ponto de integração no código é único e testável: a transação do
  `changeStatus` em `src/modules/orders/order.service.ts` ([09:40] Bruno).

**Negativas:**
- A latência mínima de notificação é o intervalo de polling do worker (~2s)
  ([09:10] Larissa).
- Nova tabela para operar; linhas entregues precisam de arquivamento depois de
  30 dias, fora do escopo desta feature ([09:08] Diego).
- Se o volume de eventos crescer muito, o worker single pode virar gargalo —
  escalabilidade é adiada (ver ADR-002).
