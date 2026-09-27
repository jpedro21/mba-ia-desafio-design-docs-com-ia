# Da Reunião ao Documento: Design Docs Gerados por IA

Este repositório entrega o **pacote completo de design docs** de uma feature de
**Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da
transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) e do código-fonte
da aplicação existente. O enunciado original do desafio está disponível no
[repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

O desafio consiste em transformar uma gravação de reunião técnica em cinco
documentos coerentes e rastreáveis — PRD, RFC, FDD, ADRs e Tracker — usando IA
como ferramenta principal de produção e o código existente como contexto. O
ponto de partida é uma empresa que opera um Order Management System (OMS) e
precisa construir, do zero, notificações em tempo real para três clientes B2B
via webhooks outbound. Toda informação registrada nos documentos precisa ser
rastreável a uma linha da transcrição (com timestamp e falante) ou a um arquivo
real do repositório — nada de requisitos, decisões ou restrições inventados.

## Ferramentas de IA utilizadas

- **Claude Code (CLI)** com o modelo Claude — ferramenta principal. Executou a
  leitura da transcrição e do código, o mapeamento dos padrões do projeto e a
  redação de todos os documentos.
- **Skill `doc-generator` (Claude Code)** — metodologia estruturada que
  organizou a produção em fases (ADRs → RFC → FDD → PRD → Tracker), definiu os
  contratos de cada documento e os portões de qualidade a checar antes de
  finalizar.
- **Verificação manual (revisão crítica)** — o papel de maestro: revisar cada
  item contra a transcrição, conferir os timestamps, confirmar os caminhos de
  arquivo citados e corrigir o que não tinha origem identificável.

## Workflow adotado

A produção seguiu a ordem do skill `doc-generator`, que espelha a natureza dos
documentos (do esqueleto de decisões para o detalhe de implementação e, por fim,
a síntese de produto):

1. **Grounding (Fase 0):** leitura integral da transcrição, catalogando
   decisões fechadas, itens descartados/adiados, questões em aberto, requisitos
   e os participantes com seus papéis. Em paralelo, `find src -type f` mapeou
   os arquivos reais (módulos, erros, middleware, logger, banco, entry points)
   e confirmou a estrutura de `docs/` e `docs/adrs/`.
2. **ADRs primeiro (7 arquivos):** cada decisão arquitetural isolada em um ADR
   (`ADR-001` a `ADR-007`), todos com Status, Contexto, Decisão, Alternativas
   Consideradas e Consequências. Cobrem as 6 decisões principais da reunião
   (outbox, worker/polling, retry+DLQ, HMAC, at-least-once, reuso de padrões)
   mais a de payload snapshot.
3. **RFC:** a proposta técnica em nível de arquitetura, consolidada sobre os
   ADRs, com alternativas descartadas e questões em aberto citando quem falou e
   quando, e links para os ADRs.
4. **FDD:** o documento de implementação, com fluxos detalhados, contratos HTTP
   com payloads de exemplo, matriz de erros `WEBHOOK_*`, estratégias de
   resiliência, observabilidade e a seção obrigatória de integração com o
   sistema existente (caminhos reais do código).
5. **PRD por último:** síntese em nível de produto/negócio, puxando dos
   documentos técnicos já prontos.
6. **Tracker:** varredura transversal de todos os documentos, mapeando cada item
   para `[hh:mm] NomeFalante` (transcrição) ou caminho de arquivo (código).
7. **README do processo** (este arquivo), por fim, documentando o percurso.
8. **Revisão final** contra a checklist de critérios de aceite, item a item.

## Prompts customizados

A qualidade do prompt determinou a qualidade do documento. Dois prompts
relevantes usados/adaptados durante o processo:

**1. Grounding da transcrição (Fase 0) — extração estruturada:**
```text
Leia TRANSCRICAO.md em detalhe e catalogue, com timestamps e nome do falante:
(a) decisões fechadas; (b) itens descartados ou adiados (NÃO são requisitos);
(c) questões em aberto; (d) requisitos funcionais e não funcionais explícitos;
(e) restrições de segurança; (f) participantes e papéis. Separe claramente o
que foi decidido do que foi apenas levantado.
```

**2. Controle de alucinação na matriz de erros do FDD:**
```text
Na Matriz de Erros do FDD, use apenas códigos WEBHOOK_ que tenham origem na
transcrição (ex.: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL,
WEBHOOK_SECRET_REQUIRED). Para cada código, diga em qual linha da transcrição
ele está ancorado. Se um código não tiver ancoragem, remova ou marque como
derivado da condição de disparo discutida, explicitando isso.
```

## Iterações e ajustes

O processo exigiu ajustes concretos de revisão crítica, principalmente para
combater a tendência de a IA preencher lacunas com conteúdo genérico:

- **Iteração 1 — delimitar escopo negativo:** o primeiro rascunho tendia a
  incluir email de fallback e dashboard visual como requisitos. Correção: como a
  reunião os descartou explicitamente, eles foram movidos para "Fora de Escopo"
  no PRD e registrados como questões em aberto no RFC, com a citação correta
  ([09:37] Larissa, [09:40] Larissa).
- **Iteração 2 — ancorar códigos de erro:** a matriz de erros precisou ser
  restrita ao prefixo `WEBHOOK_` acordado e aos códigos citados na reunião;
  os códigos de entrega derivados (timeout, falha após 5 tentativas) foram
  marcados como internos do worker (sem HTTP status), porque a transcrição
  define apenas a condição de disparo.
- **Iteração 3 — rigor nos timestamps:** cada linha `TRANSCRICAO` do Tracker
  foi conferida contra o arquivo original; timestamps que não batiam foram
  corrigidos ou a linha removida.
- **Iteração 4 — convenção de nomenclatura:** o README da pasta `docs/adrs/`
  sugeria `0001-titulo.md`; alinhei a convenção ao formato exigido
  `ADR-NNN-titulo-em-kebab-case.md` e atualizei o README da pasta para
  corresponder aos 7 ADRs produzidos.

Ao todo, o documento final é resultado de **4 ciclos principais** de geração +
revisão + correção sobre os rascunhos iniciais.

## Como navegar a entrega

Todos os arquivos entregues estão na raiz e em `docs/`. Ordem sugerida de
leitura:

1. `docs/PRD.md` — o *porquê e o quê*: problema, público, métricas, escopo.
2. `docs/RFC.md` — a proposta técnica em nível de arquitetura e o que ficou em
   aberto.
3. `docs/adrs/ADR-001-*.md` a `ADR-007-*.md` — cada decisão fechada, com
   alternativas e consequências (leia o `docs/adrs/README.md` para o índice).
4. `docs/FDD.md` — o *como construir*: fluxos, contratos, erros, resiliência e
   integração com o código existente.
5. `docs/TRACKER.md` — a referência cruzada de tudo, ligando cada item à sua
   origem na transcrição ou no código.

**Estrutura do repositório:**

```
.
├── README.md                (este arquivo — processo)
├── TRANSCRICAO.md           (fonte de origem — não alterado)
├── docs/
│   ├── PRD.md               (requisitos de produto)
│   ├── RFC.md               (proposta técnica em arquitetura)
│   ├── FDD.md               (especificação de implementação)
│   ├── TRACKER.md           (rastreabilidade)
│   └── adrs/
│       ├── README.md        (índice dos ADRs)
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-processo-separado-polling-2s.md
│       ├── ADR-003-politica-retry-backoff-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-entrega-at-least-once-com-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
│       └── ADR-007-payload-snapshot-renderizado-na-insercao.md
├── src/                    (não alterado — contexto e referência)
├── prisma/                 (não alterado)
└── tests/                  (não alterado)
```
