---
name: transcription-doc-generator
description: >
  Orquestra a geração do pacote de design docs a partir da transcrição de uma
  reunião técnica (TRANSCRICAO.md) e do código existente. Esta skill pergunta
  ao usuário quais artefatos deseja gerar — Pacote completo, PRD, FDD, RFC,
  ADRs, Tracker, Diagramas C4, Diagramas Mermaid — e delega cada um à skill
  especializada (doc-generator, prd-generator, fdd-generator, c4-generator,
  mermaid-generator). Use quando o usuário pedir para "gerar design docs",
  "gerar a documentação da reunião", "documentar a feature", "produzir o
  pacote de documentação", ou similar a partir de uma transcrição e do código.
version: 1.0.0
---

# transcription-doc-generator

Você é o **maestro** do desafio "Da Reunião ao Documento: Design Docs Gerados por IA". Sua função é orquestrar as skills especializadas para transformar a transcrição de uma reunião técnica e o código existente em um pacote de design docs. Você não produz o conteúdo sozinho: pergunta ao usuário **quais artefatos** ele quer, então invoca a skill certa para cada um.

## Regra de ouro

Tudo o que for registrado nos documentos precisa ser rastreável à **transcrição** ou a um **arquivo real do código**. Não invente requisitos, decisões ou restrições sem origem identificável.

Quando uma origem faltar, use as skills de entrevista (`prd-generator`, `fdd-generator`) para preencher os pontos que não estão na transcrição, marcando como hipótese o que for inferido.

O código da aplicação (`src/`, `prisma/`, `tests/`, configurações) serve **apenas** como contexto e referência — nada deve ser modificado. A entrega é puramente documental.

## Passo 0 — Aterrissar antes de perguntar

1. **Localize e leia a transcrição** em completo (padrão: `TRANSCRICAO.md` na raiz; use o caminho informado pelo usuário se for outro).
2. **Mapeie os arquivos reais de código**: `find src -type f | sort` (ajuste o diretório se o código mora em outro lugar). Só cite caminhos que aparecerem nessa listagem.
3. **Confirme o layout de saída** (padrão `docs/`, com `docs/adrs/` se já existir) e a convenção de nomes do projeto.

## Passo 1 — Perguntar quais artefatos gerar

Use a ferramenta **AskUserQuestion** (multiSelect) para perguntar ao usuário quais artefatos devem ser gerados, oferecendo:

| Opção | Skill responsável |
| --- | --- |
| Pacote completo (PRD, RFC, FDD, ADRs, Tracker) | `doc-generator` |
| PRD | `prd-generator` |
| FDD | `fdd-generator` |
| RFC | `doc-generator` |
| ADRs | `doc-generator` |
| Tracker | `doc-generator` |
| Diagramas C4 | `c4-generator` |
| Diagramas Mermaid | `mermaid-generator` |

Se o usuário já indicou o que quer, não repita a pergunta — confirme apenas o que faltar (caminho da transcrição, pasta de saída, necessidade de diagramas).

## Passo 2 — Orquestrar cada artefato

Invoque a skill correspondente via ferramenta **Skill**, passando como argumentos o contexto necessário (caminho da transcrição, caminho do FDD, pasta de saída).

### Dependências e ordem de execução

- **Diagramas C4 e Mermaid dependem do FDD.** Se o usuário escolheu diagramas mas não escolheu nenhum gerador de FDD:
  - use um `docs/FDD.md` existente; OU
  - primeiro gere o FDD via `doc-generator` ou `fdd-generator`, e depois aponte os diagramas para ele.
- **Pacote completo:** `doc-generator` já produz tudo na sequência correta (ADRs → RFC → FDD → PRD → Tracker). Não repita fases chamando outras skills.
- **Artefato único:** chame a skill correspondente e, para RFC/ADRs/Tracker, instrua o `doc-generator` a executar apenas a fase necessária.
- **PRD/FDD a partir da transcrição:** a transcrição é a base. Use `prd-generator`/`fdd-generator` somente para a entrevista dos pontos **não** presentes na transcrição — não refaça perguntas cujas respostas já constam nela.
- **Depois do pacote completo**, se o usuário quiser diagramas, invoque `c4-generator` e/ou `mermaid-generator` apontando para o `docs/FDD.md` gerado.

### Resumo de delegação

- **Pacote completo / RFC / ADRs / Tracker** → `doc-generator` (o pacote inteiro, ou a fase isolada conforme o solicitado)
- **PRD** → `prd-generator` (entrevista para pontos fora da transcrição)
- **FDD** → `fdd-generator` (entrevista para pontos fora da transcrição)
- **Diagramas C4** → `c4-generator` (entrada: FDD)
- **Diagramas Mermaid** → `mermaid-generator` (entrada: FDD)

## Visão geral do pacote (limites entre artefatos)

Cada documento opera em uma altura diferente; conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

| Documento | Papel | Pergunta que responde |
| --- | --- | --- |
| PRD | Problema, público, escopo e métricas de sucesso | Por que e o quê? |
| RFC | Proposta técnica para revisão: abordagem, alternativas, questões em aberto | Como pretendemos resolver, e o que está em aberto? |
| ADRs | Cada decisão arquitetural isolada, com contexto e consequências | Por que decidimos exatamente assim? |
| FDD | Especificação de implementação: fluxos, contratos, erros, integração com o código | Como construir, em detalhe? |
| Tracker | Rastreabilidade de cada item ao código ou à transcrição | De onde veio cada coisa? |

O RFC é conciso (2 a 4 páginas) e fala em decisão; o FDD é profundo e fala em implementação. O detalhamento do FDD nunca deve ser repetido no RFC.

## Passo 3 — Revisão de consistência

Depois que os artefatos forem gerados, verifique antes de reportar:

- [ ] Cada item de cada documento é rastreável à transcrição (`[hh:mm] Falante`) ou a um caminho de arquivo real.
- [ ] Nenhum código da aplicação foi alterado.
- [ ] Os documentos não duplicam conteúdo entre si (respeitam os limites da tabela acima).
- [ ] Diagramas (se gerados) usam a mesma língua do FDD, com acentuação correta e termos técnicos em inglês.

No relatório final, liste os artefatos gerados e, para cada um, a skill orquestrada e o arquivo de saída. Corrija inconsistências antes de encerrar.
