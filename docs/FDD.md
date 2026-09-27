# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e Motivação Técnica

A plataforma (OMS) precisa notificar clientes B2B em tempo real sobre mudanças
de status dos pedidos. O requisito de negócio é latência **abaixo de 10
segundos** ([09:02] Marcos). A mudança de status hoje é uma transação que já
atualiza a `order`, insere em `order_status_history` e ajusta `stock_quantity` —
adicionar um HTTP call síncrono no meio tornaria a mudança de status refém da
disponibilidade e latência do cliente ([09:04] Bruno). Por isso a feature usa o
padrão **outbox transacional**: o evento é gravado na mesma transação da mudança
de status e entregue de forma assíncrona por um **worker em processo separado**
([09:06] Diego, [09:11] Diego). Nenhum mecanismo de notificação, fila ou evento
existe hoje na aplicação — este módulo preenche esse vácuo reusando os padrões
já consolidados no projeto ([09:27]–[09:30]).

## 2. Objetivos Técnicos

- Gravar o evento na outbox **atomicamente** com a mudança de status: se a
  transação commita, o evento existe; se dá rollback, o evento some
  ([09:06] Diego, [09:41] Diego).
- Entregar o evento ao endpoint do cliente com latência **≤ 10s** no pior caso
  (polling de 2s + tempo de entrega) ([09:02] Marcos, [09:10] Larissa).
- Garantir entrega **at-least-once** com `X-Event-Id` único para dedup
  ([09:24]–[09:25] Diego).
- Garantir retry com backoff em 5 tentativas e destino final em **DLQ** com
  replay admin auditável ([09:15]–[09:18] Diego, [09:36] Sofia).
- Garantir autenticidade e integridade via **HMAC-SHA256** com secret por
  endpoint e rotação com grace period de 24h ([09:19]–[09:21] Sofia).
- Reusar os padrões existentes: módulos, `AppError`, Pino, error middleware,
  schemas Zod, `authenticate`/`requireRole`, prefixo `WEBHOOK_` ([09:27]–[09:30]).

## 3. Escopo e Exclusões

**Incluído:**
- CRUD de configuração de webhooks por cliente (cadastrar, editar, remover,
  listar) ([09:31]–[09:33]).
- Filtro de eventos por lista de status por endpoint, aplicado na inserção na
  outbox ([09:33]–[09:34]).
- Registro do evento na outbox dentro da transação de `changeStatus`
  ([09:40]–[09:41]).
- Worker de entrega com polling de 2s, retry, backoff e DLQ ([09:09]–[09:18]).
- Histórico de entregas por webhook ([09:34]).
- Rotação de secret com grace period de 24h ([09:21]).
- Replay admin de DLQ com role ADMIN e auditoria ([09:18], [09:36]).
- Assinatura HMAC-SHA256, headers `X-Event-Id`, `X-Signature`, `X-Timestamp`,
  `X-Webhook-Id` ([09:19]–[09:45]).

**Excluído / fora do escopo desta feature:**
- Recebimento de webhooks dos clientes (inbound) — comunicação é somente
  outbound ([09:02] Marcos).
- Email como fallback de falha — próxima fase ([09:37] Larissa).
- Dashboard visual para clientes — projeto separado do time de frontend
  ([09:40] Larissa).
- Rate limiting de saída — observação e decisão posterior ([09:39]).
- Arquivamento de linhas entregues após 30 dias — fora do escopo ([09:08]).
- Garantia de ordering global — garantia é por `order_id`, single-worker
  ([09:12]–[09:13]).

## 4. Fluxos Detalhados

### Fluxo 1 — Mudança de status enfileira o evento na outbox (happy path)

1. `PATCH /api/v1/orders/:id/status` chega ao `OrderController.changeStatus`
   e aciona o `OrderService.changeStatus` ([09:40]).
2. A transação abre. Dentro dela: busca a order, valida transição de status
   (`canTransition`), debita/recompõe estoque quando aplicável, atualiza a
   `order` e insere em `order_status_history` — comportamento já existente.
3. No mesmo `tx`, o serviço resolve os webhooks ativos do `customerId` do
   pedido e filtra pelos status que cada endpoint quer ouvir; se nenhum webhook
   do cliente quiser aquele status, **não insere** evento ([09:33]–[09:34]).
4. Para cada webhook que quer o evento, insere uma linha em `webhook_outbox`
   com o payload **já renderizado (snapshot)** ([09:43], [09:52]) e um
   `event_id` UUID gerado na inserção ([09:25], [09:51]).
5. Commit da transação. Evento e mudança de status persistem juntos
   ([09:06], [09:41]).
6. O worker, no próximo ciclo de polling, encontra o evento pendente.

### Fluxo 2 — Worker entrega o evento (happy path)

1. O worker (processo separado) faz polling a cada **2 segundos** dos eventos
   pendentes mais antigos, em batch pequeno, via índice em `status` e
   `created_at` ([09:09]–[09:10], [09:08]).
2. Para cada evento: monta o request HTTP para a URL do webhook com o payload
   snapshot, `Content-Type: application/json` e os headers `X-Event-Id`
   (UUID), `X-Signature` (HMAC-SHA256 do corpo com a secret do endpoint),
   `X-Timestamp` (timestamp do envio) e `X-Webhook-Id` (id do endpoint)
   ([09:20], [09:44]–[09:45]).
3. Dispara o HTTP com **timeout de 10 segundos** ([09:42]).
4. Resposta `2xx` ⇒ marca o evento como **entregue** e registra a entrega no
   histórico (payload, response, tempo de resposta) ([09:34]).
5. A linha entregue segue na outbox até o arquivamento pós-30 dias (fora de
   escopo) ([09:08]).

### Fluxo 3 — Falha de entrega, retry com backoff e DLQ (failure path)

1. Timeout de 10s, erro de rede ou resposta não-2xx ⇒ entrega **falhou** e é
   registrada como tal ([09:42], [09:15]).
2. O evento é reagendado com backoff exponencial na progressão
   **1m / 5m / 30m / 2h / 12h**, num total de **5 tentativas** (~15h)
   ([09:15]–[09:17]).
3. Se uma tentativa seguinte tiver sucesso, o evento vira **entregue**.
4. Esgotadas as 5 tentativas, o evento é movido para **`webhook_dead_letter`**
   com a payload, o motivo da última falha e timestamp; a linha da outbox deixa
   de ser pendente ([09:18]).
5. Um operador/ADMIN monitora a DLQ e, se o cliente voltou, faz o replay
   manual (Fluxo 4).

### Fluxo 4 — Replay manual da DLQ (admin)

1. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige token com role
   **ADMIN** via `requireRole` ([09:36], [09:12]).
2. O evento é recolocado na outbox como **pendente**, com novo ciclo de retry
   ([09:18]).
3. O replay é **logado com o id de quem executou**, para auditoria
   ([09:36] Sofia).

### Fluxo 5 — Rotação de secret (segurança)

1. O cliente solicita nova secret pelo endpoint de rotação ([09:21]).
2. Uma nova secret é gerada para o endpoint; a **antiga permanece válida por
   24 horas em paralelo** (grace period), depois morre ([09:21]).
3. O worker passa a assinar com a nova secret; durante o grace period, o
   cliente pode migrar os sistemas sem janela ([09:21]).

## 5. Contratos Públicos

Todos os endpoints vivem sob o prefixo `/api/v1`, seguindo a montagem do router
atual. Requisições usam `Authorization: Bearer <JWT>` (exceto a entrega do
worker, que é HTTP puro assinado por HMAC).

### 5.1 `POST /api/v1/webhooks` — Cadastrar webhook

- **Headers:** `Authorization: Bearer <jwt>` · `Content-Type: application/json`
- **Body (request):**
  ```json
  {
    "customerId": "5f1c1a4f-...",
    "url": "https://atlas.example.com/webhooks/orders",
    "eventStatuses": ["SHIPPED", "DELIVERED"]
  }
  ```
- **Response 201 (created):**
  ```json
  {
    "id": "a3d9...",
    "customerId": "5f1c1a4f-...",
    "url": "https://atlas.example.com/webhooks/orders",
    "eventStatuses": ["SHIPPED", "DELIVERED"],
    "secret": "wbsec_9f2c...",
    "active": true,
    "createdAt": "2026-09-27T14:03:00Z"
  }
  ```
  A secret é **gerada pela plataforma e devolvida somente na criação**
  ([09:31] Marcos).
- **Status codes:** `201` sucesso · `400` validação de schema (Zod) ·
  `401` não autenticado · `422` URL inválida/não-https (`WEBHOOK_INVALID_URL`).

### 5.2 `GET /api/v1/webhooks?customerId=<uuid>` — Listar webhooks de um cliente

- **Headers:** `Authorization: Bearer <jwt>`
- **Response 200:**
  ```json
  {
    "items": [
      {
        "id": "a3d9...",
        "customerId": "5f1c1a4f-...",
        "url": "https://atlas.example.com/webhooks/orders",
        "eventStatuses": ["SHIPPED", "DELIVERED"],
        "active": true,
        "createdAt": "2026-09-27T14:03:00Z"
      }
    ]
  }
  ```
  (a secret **não** é devolvida na listagem — só na criação.)
- **Status codes:** `200` sucesso · `401` não autenticado · `400` query
  inválida.

### 5.3 `PATCH /api/v1/webhooks/:id` — Editar webhook

- **Headers:** `Authorization: Bearer <jwt>` · `Content-Type: application/json`
- **Body (request):**
  ```json
  { "url": "https://atlas.example.com/webhooks/orders/v2", "eventStatuses": ["DELIVERED"] }
  ```
- **Response 200:** configuração atualizada (mesma forma da 5.1, sem secret).
- **Status codes:** `200` sucesso · `400` validação · `401` não autenticado ·
  `404` não encontrada (`WEBHOOK_NOT_FOUND`) · `422` URL inválida.

### 5.4 `DELETE /api/v1/webhooks/:id` — Remover webhook

- **Headers:** `Authorization: Bearer <jwt>`
- **Response 204** (sem corpo).
- **Status codes:** `204` sucesso · `401` não autenticado · `404` não encontrada.

### 5.5 `GET /api/v1/webhooks/:id/deliveries` — Histórico de entregas

- **Headers:** `Authorization: Bearer <jwt>`
- **Response 200** (últimos 100 registros; cada entrega registra payload,
  response e tempo de resposta — [09:34] Marcos):
  ```json
  {
    "items": [
      {
        "id": "d7e1...",
        "eventId": "11bf5b37-...",
        "status": "delivered",
        "payload": { "event_id": "11bf5b37-...", "event_type": "order.status_changed", "order_id": "f9c2...", "order_number": "ORD-000123", "from_status": "PAID", "to_status": "SHIPPED", "customer_id": "5f1c1a4f-...", "total_cents": 8990, "timestamp": "2026-09-27T14:03:00Z" },
        "response": { "status": 200, "body": "ok" },
        "latencyMs": 142,
        "attemptedAt": "2026-09-27T14:03:02Z"
      }
    ]
  }
  ```
- **Status codes:** `200` sucesso · `401` não autenticado · `404` não encontrada.

### 5.6 `POST /api/v1/webhooks/:id/rotate-secret` — Rotacionar secret

- **Headers:** `Authorization: Bearer <jwt>`
- **Response 200:**
  ```json
  {
    "secret": "wbsec_7a1e...",
    "previousSecretValidUntil": "2026-09-28T14:03:00Z"
  }
  ```
  A secret anterior permanece válida por **24h em paralelo** ([09:21] Sofia).
- **Status codes:** `200` sucesso · `401` não autenticado · `404` não encontrada.

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — Replay de DLQ

- **Headers:** `Authorization: Bearer <jwt>` com role **ADMIN** ([09:36]).
- **Response 202:**
  ```json
  { "replayed": true, "eventId": "11bf5b37-..." }
  ```
- **Status codes:** `202` reenfileirado · `401` não autenticado · `403` role
  não ADMIN · `404` registro de dead-letter não encontrado.

## 6. Matriz de Erros

Prefixo acordado: **`WEBHOOK_`** ([09:29] Larissa). O middleware central de
erros já converte `AppError`/Zod/Prisma no formato `{ error: { code, message, details } }`
(ver Integração, §12).

| Código | HTTP | Condição de disparo |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook/registro solicitado não existe ([09:28] Bruno). |
| `WEBHOOK_INVALID_URL` | 422 | URL inválida ou não-`https`; TLS é obrigatório ([09:28], [09:23] Sofia). |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret ausente onde é obrigatória (validação de configuração) ([09:28] Bruno). |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede **64KB**; a plataforma erra em vez de truncar ([09:24] Diego, Larissa). |
| `WEBHOOK_DELIVERY_TIMEOUT` | N/A (worker) | Timeout de **10s** no HTTP call; tratado como falha e reagendado ([09:42] Diego). |
| `WEBHOOK_DELIVERY_FAILED` | N/A (worker) | Falha após as **5 tentativas** de retry; evento movido para DLQ ([09:15]–[09:18]). |

## 7. Estratégias de Resiliência

- **Timeout:** 10 segundos por chamada HTTP; cliente lento que não responde em
  10s é tratado como falha e marcado para retry ([09:42] Diego).
- **Retry/backoff:** 5 tentativas no total, progressão 1m/5m/30m/2h/12h (~15h)
  ([09:15]–[09:17] Diego).
- **DLQ:** após esgotar tentativas, evento vai para tabela separada com payload,
  motivo e timestamp; outbox principal permanece limpa para leitura
  ([09:18] Diego).
- **Replay:** manual, via endpoint admin, recoloca o evento como pendente
  ([09:18] Diego).
- **Idempotência/dedup:** garantia at-least-once; cada evento carrega
  `X-Event-Id` único; o cliente deduplica ([09:24]–[09:25] Diego).
- **Ordenamento:** single-worker processa em ordem de `created_at`, preservando
  ordem por `order_id`; sem garantia de ordering global ([09:12]–[09:13]).
- **Polling atômico:** o worker marca o evento como `processando` antes de
  disparar, evitando entrega dupla concorrente no mesmo processo.

## 8. Observabilidade

- **Logger:** Pino, o logger já usado no projeto inteiro; nenhuma biblioteca nova
  ([09:29] Bruno). A secret nunca é logada (Pino já redige
  `*.secret`/`*.token`/headers de autorização no logger atual).
- **Campos estruturados por evento de entrega** (≥3 por evento):
  `event_id`, `webhook_id`, `customer_id`, `delivery_status`, `attempt`,
  `status_code`, `latency_ms`.
- **Métricas a expor (≥3):**
  `webhook_deliveries_total` (contador por status),
  `webhook_delivery_latency_ms` (histograma p50/p95/p99),
  `webhook_outbox_pending` (gauge da fila),
  `webhook_dlq_total` (gauge/acumulador da dead-letter).

## 9. Dependências e Compatibilidade

- **Banco:** MySQL existente via Prisma; nova tabela `webhook_outbox`, de
  configuração de webhooks, de entregas e de dead-letter seguem o padrão de IDs
  UUID do projeto ([09:51] Larissa, `prisma/schema.prisma`).
- **Node:** ≥ 20 (engines do projeto).
- **Bibliotecas:** reuso de `@prisma/client`, `pino`, `uuid`, `zod`, `express`,
  `jsonwebtoken`. Nenhuma dependência nova de mensageria (ADR-001).
- **Processos:** API (`src/server.ts`) e worker (`src/worker.ts`) são processos
  separados, cada um com instância própria de `PrismaClient` ([09:11], [09:29]–[09:30]).

## 10. Critérios de Aceite Técnicos

- [ ] Mudança de status e inserção na outbox acontecem na **mesma transação**;
      rollback remove o evento junto ([09:06], [09:41]).
- [ ] Evento não é inserido quando nenhum webhook do cliente quer aquele status
      ([09:34]).
- [ ] Payload entregue é o snapshot renderizado na inserção, no formato do
      Fluxo 5.5 ([09:43], [09:52]).
- [ ] Worker entrega em até 2s de polling; pior caso < 10s ([09:10]).
- [ ] Entrega tem retry em 1m/5m/30m/2h/12h e move para DLQ após a 5ª falha
      ([09:15]–[09:17]).
- [ ] Request de entrega carrega `X-Event-Id`, `X-Signature` (HMAC-SHA256),
      `X-Timestamp` e `X-Webhook-Id` ([09:20], [09:44]–[09:45]).
- [ ] Replay de DLQ exige role ADMIN e registra quem executou ([09:36]).
- [ ] URL não-https é recusada na validação ([09:23]).
- [ ] Todos os códigos de erro do módulo usam prefixo `WEBHOOK_`
      ([09:28]–[09:29]).
- [ ] Nenhuma secret aparece em logs (ver §8).

## 11. Riscos e Mitigação

- **Risco:** quebrar a atomicidade (evento fora da transação de status) ⇒
  notificações inconsistentes. **Mitigação:** `publishWebhookEvent(tx, ...)`
  recebe o `tx` da transação atual; teste ponta a ponta valida rollback
  ([09:41] Diego).
- **Risco:** cliente com indisponibilidade longa sobrecarrega a DLQ. **Mitigação:**
  janela de retry de ~15h + replay manual controlado por ADMIN ([09:17], [09:18]).
- **Risco:** comprometimento de secret. **Mitigação:** secret por endpoint,
  rotação com grace period de 24h, TLS obrigatório, redação de secrets nos logs
  ([09:21] Sofia, [09:23] Sofia).
- **Risco:** volume alto de eventos em período curto (ex.: 50 mudanças/min).
  **Mitigação:** observação; rate limiting de saída é questão em aberto para
  decidir depois ([09:39]).
- **Risco:** duplicidade confunde clientes (at-least-once). **Mitigação:**
  `X-Event-Id` + documentação no portal de desenvolvedor ([09:25]–[09:26]).

## 12. Integração com o Sistema Existente

Caminhos reais do código-base e como o módulo de webhooks se integra com cada um:

- **`src/modules/orders/order.service.ts`** — o ponto crítico. O método
  `changeStatus` (linhas 126–179) executa a transação que atualiza a order, o
  histórico e o estoque. É aqui que entra a chamada à função
  `publishWebhookEvent(tx, order, fromStatus, toStatus)`, dentro do mesmo `tx`,
  após `tx.order.update`/`tx.orderStatusHistory.create` e antes do commit
  ([09:40]–[09:41] Bruno). Uma função pura recebendo o `tx` evita injetar um
  repository inteiro no `OrderService` ([09:41] Diego).

- **`src/modules/orders/order.routes.ts`** — serve de modelo para o novo
  `webhook.routes.ts`: o padrão `router.use(authenticate)` + `validate({ body/query/params })`
  + delegar ao controller é o que o módulo de webhooks reproduz para o CRUD, o
  histórico de entregas e a rotação de secret ([09:27]–[09:28] Bruno). O padrão
  de schemas em `src/modules/orders/order.schemas.ts` (Zod + `z.infer`) define
  os schemas `createWebhookSchema`, `updateWebhookSchema`, `webhookIdParamSchema`.

- **`src/middlewares/auth.middleware.ts`** — o endpoint de replay de DLQ usa o
  `requireRole('ADMIN')` já implementado aqui, que lança `ForbiddenError` quando
  o role do JWT não é ADMIN. A auditoria do replay (`quem fez`) se apoia no
  `req.user.id` populado pelo `authenticate` ([09:36] Sofia, [09:12] Larissa).

- **`src/shared/errors/app-error.ts`** — as novas exceções do módulo
  (ex.: `WebhookNotFoundError`, `InvalidWebhookUrlError`, lançando códigos
  `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`)
  estendem `AppError` e são exportadas via `src/shared/errors/index.ts`, seguindo
  exatamente o padrão de `http-errors.ts` ([09:28]–[09:29] Bruno). Isso faz o
  `src/middlewares/error.middleware.ts` capturá-los sem nenhuma mudança, pois ele
  já serializa qualquer `AppError` para `{ error: { code, message, details } }`
  ([09:29] Bruno).

- **`src/shared/logger/index.ts`** — o logger Pino exportado aqui é reusado pelo
  módulo e pelo worker para os logs estruturados de entrega (§8). Os paths de
  redação já cobrem `*.secret` e `*.token`, então a secret nunca é gravada
  ([09:29] Bruno).

- **`src/config/database.ts`** — `createPrismaClient()` é o factory usado pela
  API; o worker chama o mesmo factory para criar **sua própria instância** de
  `PrismaClient` (PrismaClient é por processo), apontando para a mesma
  `DATABASE_URL` ([09:29]–[09:30] Bruno, Diego).

- **`src/app.ts` / `src/routes/index.ts`** — `buildControllers` (src/app.ts) e
  `buildApiRouter` (src/routes/index.ts) ganham o novo `WebhookController`/router;
  o router é montado em `/webhooks` (e `/admin/webhooks` para o replay), no mesmo
  esquema dos módulos `orders`, `products`, `customers` ([09:27] Bruno).

- **`src/server.ts`** — é o modelo de entry point para o novo `src/worker.ts`:
  bootstrap com env, instância de Prisma, log estruturado e shutdown gracioso em
  SIGINT/SIGTERM ([09:11] Larissa). A lógica de processamento fica em
  `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno).
