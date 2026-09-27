# Tracker — Rastreabilidade

Mapa de cada item registrado nos documentos à origem na transcrição
(`TRANSCRICAO`) ou no código (`CODIGO`). Onde a origem é a transcrição, a
localização usa o formato `[hh:mm] NomeFalante`; onde é o código, o caminho do
arquivo relativo à raiz do repositório.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (URL, status desejados, secret gerada e devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar configuração de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtrar eventos por status por endpoint, na inserção na outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Enfileirar evento na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Worker entrega HTTP com payload e headers X-Event-Id / X-Signature / X-Timestamp / X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry com backoff 1m/5m/30m/2h/12h e DLQ após a 5ª falha | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimas 100) com payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay admin de DLQ com role ADMIN e auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | customer_id passado no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação < 10s no pior caso | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório (URL https; http recusado) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 e secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload acima de 64KB erra (não trunca) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10s no worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id, single-worker, sem garantia global | TRANSCRICAO | [09:13] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Auditoria registra quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| PRD-O-01 | docs/PRD.md | Requisito Não Funcional | Meta: ≥95% das entregas com latência < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-O-02 | docs/PRD.md | Requisito Não Funcional | Meta: 100% das mudanças de status geram evento na outbox | TRANSCRICAO | [09:41] Diego |
| PRD-O-03 | docs/PRD.md | Requisito Não Funcional | Meta: zero perda de clientes por ausência de notificação (Atlas) | TRANSCRICAO | [09:00] Marcos |
| PRD-R-01 | docs/PRD.md | Risco | Perda de cliente por prazo (Atlas pode migrar) | TRANSCRICAO | [09:00] Marcos |
| PRD-R-02 | docs/PRD.md | Risco | Cliente com indisponibilidade longa atrasa notificações | TRANSCRICAO | [09:17] Marcos |
| PRD-R-03 | docs/PRD.md | Risco | Comprometimento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-R-04 | docs/PRD.md | Risco | Confusão do cliente com duplicidade (at-least-once) | TRANSCRICAO | [09:25] Sofia |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono no service descartado (cliente lento trava mudanças; sem rollback) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado (overengineering / infra nova) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartado (MySQL sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado (complexidade de coordenação) | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Escala futura do worker (particionamento por order_id / lock) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Email como fallback de falha — próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Arquivamento de entregas após 30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /api/v1/webhooks — cadastrar webhook (secret só na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | GET /api/v1/webhooks — listar por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | PATCH /api/v1/webhooks/:id — editar configuração | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | DELETE /api/v1/webhooks/:id — remover | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | POST /api/v1/webhooks/:id/rotate-secret — rotação com grace 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | POST /api/v1/admin/webhooks/dead-letter/:id/replay — admin | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Código WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Restrição | Código WEBHOOK_INVALID_URL (422) — TLS obrigatório | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Código WEBHOOK_SECRET_REQUIRED (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Código WEBHOOK_PAYLOAD_TOO_LARGE (422, >64KB) | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-05 | docs/FDD.md | Restrição | Código WEBHOOK_DELIVERY_TIMEOUT (worker, timeout 10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-06 | docs/FDD.md | Restrição | Código WEBHOOK_DELIVERY_FAILED (worker, DLQ após 5ª falha) | TRANSCRICAO | [09:15] Diego |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Retry/backoff 5 tentativas + DLQ em tabela separada | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Timeout 10s tratado como falha e reagendado | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Observabilidade com Pino, logs estruturados e métricas | TRANSCRICAO | [09:29] Bruno |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL, transação atômica com a mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling-2s.md | Decisão | Worker em processo separado, polling de 2s | TRANSCRICAO | [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-politica-retry-backoff-dlq.md | Decisão | Retry 5x com backoff e DLQ em tabela separada | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Decisão | Entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso dos padrões existentes e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-renderizado-na-insercao.md | Decisão | Payload snapshot renderizado na inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Transação do changeStatus é o ponto de inserção do evento na outbox | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Padrão de rotas (authenticate + validate) replicado no webhook.routes | CODIGO | src/modules/orders/order.routes.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | requireRole('ADMIN') usado no replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | Novas exceções do módulo estendem AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-05 | docs/FDD.md | Decisão | Error middleware centralizado serializa AppError/Zod/Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | Logger Pino reusado com redação de secrets | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Worker cria instância própria de PrismaClient | CODIGO | src/config/database.ts |
| FDD-INTEG-08 | docs/FDD.md | Decisão | WebhookController e router adicionados ao build da app | CODIGO | src/app.ts |
| FDD-INTEG-09 | docs/FDD.md | Decisão | src/server.ts é o modelo da nova entry src/worker.ts | CODIGO | src/server.ts |
| FDD-INTEG-10 | docs/FDD.md | Decisão | Rotas de webhooks montadas no router principal | CODIGO | src/routes/index.ts |
