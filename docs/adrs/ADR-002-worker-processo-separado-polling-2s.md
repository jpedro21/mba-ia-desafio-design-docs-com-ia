# ADR-002 — Worker em processo separado com polling de 2 segundos

## Status

Accepted

## Contexto

Decidido o padrão outbox (ADR-001), falta definir **como** o worker lê a tabela
e **onde** ele roda. O MySQL não possui um listener nativo como o
NOTIFY/LISTEN do Postgres para notificar um processo externo ([09:09] Diego).
Além disso, se o worker rodasse dentro da mesma instância da API, um restart da
API derrubaria o worker junto ([09:11] Diego).

O requisito de negócio é latência abaixo de 10 segundos ([09:02] Marcos).

## Decisão

O worker roda como **processo Node separado**, com entry point próprio
(`src/worker.ts`) e um script `npm run worker`, no mesmo padrão do
`src/server.ts` existente ([09:11] Diego, [09:11] Larissa, [09:28] Bruno). Ele
faz **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais
antigos, processando e marcando como entregues ([09:09] Diego). O requisito de
"abaixo de 10 segundos" é atendido folgado, com latência mínima de 2 segundos
no pior caso, decisão aceita explicitamente ([09:10] Larissa, [09:10] Marcos).

O worker usa o **mesmo banco e a mesma stack** (mesma `DATABASE_URL`), mas com
**instância própria de PrismaClient**, porque PrismaClient é por processo
([09:11] Bruno, [09:29]–[09:30] Diego).

**Ordenamento:** enquanto houver um único worker, os eventos são processados em
ordem de `created_at`, preservando a ordem por `order_id`. Não há garantia de
ordering **global**; a garantia é por `order_id` e apenas enquanto single-worker
([09:12]–[09:13] Diego, [09:13] Larissa). Escala futura (particionamento por
`order_id` ou lock pessimista) é problema do futuro, não do escopo atual
([09:13] Diego).

## Alternativas Consideradas

- **Trigger de banco para reatividade** — perguntada por Bruno ([09:09]);
  descartada por Diego ([09:09]): o MySQL não tem listener nativo tipo
  NOTIFY/LISTEN; um trigger executa apenas SQL e não notifica um processo
  externo; improvisar (escrever em arquivo ou bater em endpoint) fica esquisito.
- **Worker dentro do mesmo processo da API** — implícito na discussão e
  descartado por Diego ([09:11]): se a API reinicia, o worker é perdido.

## Consequências

**Positivas:**
- Ciclo de vida independente da API: restart da API não derruba o worker
  ([09:11] Diego).
- Implementação simples (polling com Prisma), sem dependência de infra nova.
- Ordenamento correto por `order_id` no cenário single-worker, suficiente para
  os clientes, que nunca pediram ordering global ([09:14] Marcos).

**Negativas:**
- Latência mínima de 2 segundos em qualquer cenário ([09:10] Larissa).
- Garantia de ordering limitada: múltiplos workers em paralelo no futuro
  perderiam a garantia sem particionamento ([09:12] Diego).
- Operação: dois processos Node para monitorar em vez de um.
