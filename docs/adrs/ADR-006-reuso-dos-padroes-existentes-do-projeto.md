# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

Accepted

## Contexto

A codebase tem padrões claros: cada domínio é um módulo em `src/modules` com
controller, service, repository, routes e schemas ([09:27] Bruno); erros usam a
classe `AppError` com códigos tipo `INSUFFICIENT_STOCK`,
`INVALID_STATUS_TRANSITION` ([09:28] Bruno); o logger é Pino no projeto inteiro
([09:29] Bruno); e o middleware de erro centralizado já trata `AppError`, Zod e
Prisma ([09:29] Bruno). A decisão é reusar tudo isso em vez de criar camadas
novas.

## Decisão

- O webhook vira um **módulo `src/modules/webhooks`** seguindo a estrutura
  padrão (controller, service, repository, routes, schemas) ([09:27]–[09:28]
  Bruno).
- A lógica do worker fica numa **entry point separada `src/worker.ts`** com o
  processamento em arquivo do módulo (ex.: `webhook.processor.ts`)
  ([09:28] Bruno, [09:11] Larissa).
- **Prefixo `WEBHOOK_`** em todos os códigos de erro do módulo, ex.:
  `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`
  ([09:28]–[09:29] Bruno, [09:29] Larissa).
- **Reuso máximo do que já existe**: `AppError`, Pino, error middleware,
  `validate` middleware com schemas Zod, `authenticate`/`requireRole`, padrão de
  códigos de erro ([09:30] Larissa).
- O worker abre uma **instância própria de `PrismaClient`** (PrismaClient é por
  processo; mesmo banco, mesma `DATABASE_URL`) ([09:29]–[09:30] Bruno).

## Alternativas Consideradas

- **Criar camadas próprias para o módulo (erros, logger, validação)** —
  descartada implicitamente em favor do reuso máximo: "reuso máximo do que já
  existe" ([09:30] Larissa). Bruno é explícito: "não vamos botar nada novo"
  ([09:29]).
- **Novo mecanismo de mensageria (Redis Streams) para o transporte** —
  descartada por overengineering e infra nova (ver ADR-001, [09:07] Diego).

## Consequências

**Positivas:**
- Consistência e baixo custo de manutenção; a equipe já conhece os padrões
  ([09:27] Bruno, [09:30] Larissa).
- O error middleware centralizado captura os erros do módulo sem mudança de
  infraestrutura ([09:29] Bruno).
- Código mais fácil de revisar (Sofia revisa segurança antes do deploy)
  ([09:46] Sofia).

**Negativas:**
- Acoplamento ao middleware centralizado e aos schemas Zod do projeto: mudanças
  nesses componentes afetam o módulo.
- Dois processos Node (API e worker) compartilhando o mesmo banco — instância
  própria de PrismaClient por processo ([09:29]–[09:30] Diego).
