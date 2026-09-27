# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** —
enviaram um pedido formal para serem notificados em tempo real quando o status
dos pedidos deles muda na plataforma ([09:00] Marcos). Hoje eles fazem polling
no `GET /orders` para detectar mudanças, o que deixa a integração lenta e cara
([09:00] Marcos). A feature entrega um **Sistema de Webhooks de Notificação de
Pedidos**: quando o status de um pedido muda, a plataforma envia uma notificação
HTTP para o endpoint do cliente de forma assíncrona, confiável e assinada
([09:06], [09:19]). A comunicação é **somente outbound** — os clientes recebem,
não enviam ([09:02] Marcos).

## 2. Problema e Motivação

Os clientes precisam reagir à mudança de status dos pedidos (expedição,
entrega, cancelamento) em tempo real, e hoje o único mecanismo é polling manual,
que não atende "tempo real" e custa chamadas constantes ([09:00] Marcos). Para
os clientes, **tempo real é qualquer coisa abaixo de 10 segundos** ([09:02]
Marcos). A urgência é concreta: a **Atlas sinalizou que pode migrar para o
concorrente** se a feature não for entregue até o fim do trimestre ([09:00]
Marcos), e o prazo combinado é fim de novembro ([09:45] Marcos).

## 3. Público-Alvo e Cenários de Uso

- **Clientes B2B integrados por API** — os três clientes iniciais (Atlas,
  MaxDistribuição, Nova Cargo) e outros que adotarem o padrão. Cenário: quando
  um pedido deles muda de `PAID` para `SHIPPED` (ou outro status), o endpoint
  deles é notificado em segundos, sem polling ([09:00]–[09:02], [09:02]).
- **Operadores e integradores da plataforma** — cadastram e gerenciam webhooks
  por cliente via API autenticada com JWT da plataforma ([09:31]–[09:33]).
- **Administradores da plataforma** — reprocessam eventos em DLQ e auditam
  entregas via endpoints admin ([09:18], [09:36]).
- **Time interno (dev/ops)** — monitora a entrega via histórico de entregas,
  logs e métricas ([09:34]).

## 4. Objetivos e Métricas de Sucesso

- **O1 — Notificar mudanças de status em tempo real:** pelo menos **95% das
  notificações entregues com latência < 10 segundos** entre a mudança de status
  e a chegada ao endpoint do cliente ([09:02] Marcos, [09:10] Larissa).
  *Método de medição:* percentis de latência calculados a partir dos logs de
  entrega do worker (tempo entre inserção na outbox e resposta `2xx`).
- **O2 — Cobertura transacional:** **100% das mudanças de status** de pedidos de
  clientes com webhook ativo geram evento na outbox na mesma transação
  ([09:06] Diego, [09:41] Diego). *Método de medição:* contador
  `webhook_outbox_pending` + auditoria por `order_id`.
- **O3 — Retenção de clientes:** **zero perdas por ausência de notificação** —
  a Atlas permanece cliente após a entrega no prazo ([09:00] Marcos,
  [09:47] Marcos). *Método de medição:* acompanhamento comercial/comercial; SLA
  com a Atlas confirmado pelo PM ([09:47]).

## 5. Escopo (Incluso / Fora de Escopo)

**Incluso:**
- Cadastro, edição, remoção e listagem de webhooks por cliente ([09:31]–[09:33]).
- Filtro de eventos por status por endpoint ([09:33]–[09:34]).
- Notificação outbound via worker com retry e DLQ ([09:06]–[09:18]).
- Histórico de entregas por webhook ([09:34]).
- Assinatura HMAC-SHA256, secret por endpoint e rotação com grace period de 24h
  ([09:19]–[09:21]).
- Replay admin de DLQ com auditoria ([09:18], [09:36]).

**Fora de Escopo:**
- **Email como fallback de falha** — pedido pelo PM, explicitamente adiado para
  a próxima fase, após medir o impacto ([09:37] Marcos, [09:37] Larissa).
- **Dashboard visual para clientes** — descartado; painel é projeto separado do
  time de frontend ([09:40] Marcos, [09:40] Larissa).
- **Recebimento de webhooks (inbound)** — comunicação é somente outbound
  ([09:02] Marcos).
- **Rate limiting de saída** — a ser observado e decidido depois ([09:39]).
- **Garantia de ordering global** — não é prometida; a garantia é por `order_id`
  e single-worker ([09:13]).

## 6. Requisitos Funcionais

- **FR-01 — Cadastrar webhook:** o cliente cadastra um webhook informando URL e
  os status que quer receber; a plataforma gera a secret e a devolve na criação
  ([09:31] Marcos).
- **FR-02 — Editar webhook:** configuração pode ser alterada (URL, status,
  ativação) ([09:33] Bruno).
- **FR-03 — Remover webhook:** configuração pode ser removida ([09:33] Bruno).
- **FR-04 — Listar webhooks:** listar os webhooks de um cliente ([09:33] Bruno).
- **FR-05 — Filtrar eventos por status:** o cliente escolhe quais status ouvir;
  o filtro é aplicado na **inserção na outbox**, evitando linhas desnecessárias
  ([09:33]–[09:34] Marcos, Bruno).
- **FR-06 — Enfileirar evento na mudança de status:** toda mudança de status
  gera evento na outbox **na mesma transação** da mudança ([09:40]–[09:41]
  Bruno, [09:06] Diego).
- **FR-07 — Entregar via worker:** worker em polling de 2s dispara o HTTP ao
  endpoint do cliente com payload renderizado e headers `X-Event-Id`,
  `X-Signature`, `X-Timestamp`, `X-Webhook-Id` ([09:09]–[09:10], [09:20],
  [09:44]–[09:45]).
- **FR-08 — Retry com backoff e DLQ:** 5 tentativas na progressão 1m/5m/30m/2h/12h;
  após esgotar, evento vai para a dead-letter ([09:15]–[09:18] Diego).
- **FR-09 — Histórico de entregas:** cliente consulta as últimas 100 entregas
  do webhook com sucesso/falha, payload, response e tempo de resposta
  ([09:34] Marcos).
- **FR-10 — Rotacionar secret:** cliente solicita nova secret; a antiga fica
  válida por 24h em paralelo ([09:21] Sofia).
- **FR-11 — Replay admin de DLQ:** endpoint admin recoloca evento da dead-letter
  na outbox; exige role ADMIN e loga quem executou ([09:18] Diego, [09:36] Sofia).
- **FR-12 — `customer_id` explícito:** o cliente é identificado no body/path da
  requisição (o JWT é do usuário operador, não do cliente) ([09:32] Larissa).

## 7. Requisitos Não Funcionais

- **NFR-01 — Latência:** notificação entregue abaixo de 10 segundos no pior
  caso (polling de 2s + entrega) ([09:02] Marcos, [09:10] Larissa).
- **NFR-02 — Segurança — TLS:** URL do webhook obrigatoriamente `https`; URL
  `http` é recusada na validação ([09:23] Sofia).
- **NFR-03 — Segurança — autenticidade:** assinatura HMAC-SHA256 do payload no
  header `X-Signature`, secret única por endpoint ([09:19]–[09:21] Sofia).
- **NFR-04 — Limite de payload:** eventos com payload acima de 64KB **erram** em
  vez de truncar ([09:24] Diego, Larissa).
- **NFR-05 — Entrega:** garantia at-least-once; duplicidade possível e dedup
  pelo `X-Event-Id` do lado do cliente ([09:24]–[09:25] Diego).
- **NFR-06 — Timeout:** 10 segundos por chamada HTTP do worker ([09:42] Diego).
- **NFR-07 — Ordenamento:** ordem por `order_id` enquanto single-worker; sem
  garantia de ordering global ([09:12]–[09:13] Diego, Larissa).
- **NFR-08 — Rastreabilidade:** replay de DLQ registra quem executou
  ([09:36] Sofia).

## 8. Decisões e Trade-offs

- **Outbox transacional em vez de HTTP síncrono:** consistência e não-bloqueio
  vs. latência mínima de ~2s (ADR-001).
- **Polling de 2s em vez de listener/trigger:** simplicidade e ausência de infra
  nova vs. reatividade imediata e latência mínima de 2s (ADR-002).
- **5 retries com backoff em vez de 3:** cobertura de ~15h de indisponibilidade
  vs. espera maior antes da DLQ (ADR-003).
- **At-least-once em vez de exactly-once:** padrão de mercado e simplicidade vs.
  duplicidade que o cliente precisa deduplicar (ADR-005).
- **Secret por endpoint em vez de secret global:** isolamento de comprometimento
  vs. gestão de muitas secrets (ADR-004).
- **Payload snapshot na inserção:** fidelidade histórica do evento vs. payload
  que pode defasar do estado atual do pedido (ADR-007).

## 9. Dependências

- **Clientes:** precisam disponibilizar endpoint HTTPS receptivo e implementar
  verificação HMAC-SHA256 e dedup por `event_id` ([09:19]–[09:20], [09:25]).
- **Documentação:** PM publica no portal de desenvolvedor o padrão at-least-once,
  headers e dedup ([09:26] Marcos, [09:26] Larissa).
- **Infra:** banco MySQL existente; nenhuma infraestrutura nova de mensageria
  ([09:07] Diego).
- **Prazo/time:** três sprints, com revisão de segurança (2 dias úteis) antes do
  deploy ([09:46] Larissa, [09:46] Sofia).

## 10. Riscos e Mitigação

- **R1 — Perda de cliente por prazo:** probabilidade **Média**; impacto **Alto**
  (migração da Atlas). *Mitigação:* prazo de 3 sprints com revisão de segurança
  incluída, atualização dos clientes ao longo da entrega ([09:45]–[09:47]).
- **R2 — Cliente com indisponibilidade longa:** probabilidade **Média**;
  impacto **Médio** (notificações atrasadas até ~15h, DLQ acumulando).
  *Mitigação:* backoff de 5 tentativas + replay admin manual quando o cliente
  voltar ([09:17] Marcos, [09:18] Diego).
- **R3 — Comprometimento de secret:** probabilidade **Baixa**; impacto **Alto**
  (eventos com dados de pedido forjados/adulterados). *Mitigação:* HMAC-SHA256,
  secret por endpoint, rotação com grace period de 24h, TLS obrigatório
  ([09:19]–[09:23] Sofia).
- **R4 — Confusão por duplicidade (at-least-once):** probabilidade **Média**;
  impacto **Médio** (integração incorreta do cliente). *Mitigação:* `X-Event-Id`
  + documentação destacada no portal de desenvolvedor ([09:25]–[09:26]).

## 11. Critérios de Aceitação

- Cliente consegue cadastrar, listar, editar e remover webhooks por API, com a
  secret devolvida na criação (FR-01 a FR-04, FR-12).
- Ao mudar o status de um pedido, o endpoint do cliente é notificado em menos de
  10 segundos com payload e headers corretos (FR-06, FR-07, NFR-01).
- Cliente que escolheu só `SHIPPED`/`DELIVERED` não recebe outros eventos
  (FR-05).
- Em caso de falha, o evento é reentregue até 5 vezes com backoff e, ao esgotar,
  aparece na DLQ para replay admin com auditoria (FR-08, FR-11, NFR-08).
- Cliente consegue consultar as últimas 100 entregas com sucesso/falha, payload,
  response e tempo de resposta (FR-09).
- Cliente consegue rotacionar a secret, com a antiga válida por 24h em paralelo
  (FR-10).
- Nenhuma mudança de status fica sem evento na outbox (O2).
- Documentação de integração publicada no portal (at-least-once, headers,
  dedup) ([09:26]).

## 12. Estratégia de Testes

- **Unitários:** validação dos schemas Zod (URL https, payload > 64KB); geração
  e verificação da assinatura HMAC-SHA256; cálculo da progressão de backoff;
  lógica de filtro de status por endpoint.
- **Integração (transação):** teste do `changeStatus` confirma que o evento é
  gravado na outbox na mesma transação e **some no rollback** ([09:41] Diego).
- **Ponta a ponta (worker):** mudança de status → entrega no endpoint de teste
  com headers corretos; simulação de falha → retry → DLQ → replay admin.
- **Segurança (revisão da Sofia):** revisão de HMAC, geração de secret e schemas
  antes do deploy ([09:46] Sofia).
- **Conformidade com padrões do projeto:** testes seguem o padrão Vitest já
  existente no repositório (`tests/`).
