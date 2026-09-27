# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Accepted

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints fora da infra da
plataforma. O cliente precisa conseguir validar que a requisição veio realmente
da plataforma e que ninguém adulterou o payload no meio ([09:19] Sofia). Já
houve caso de cliente que vazou secret em log de aplicação, então a gestão das
secrets é uma preocupação real ([09:22] Diego).

## Decisão

- Assinar o payload com **HMAC-SHA256**, usando uma secret compartilhada entre
  a plataforma e o cliente, enviada no header **`X-Signature`**. O cliente
  verifica do lado dele ([09:19]–[09:20] Sofia). SHA-256 é o padrão de mercado,
  todo cliente sério tem biblioteca para isso ([09:20] Sofia).
- **Uma secret única por endpoint de webhook** do cliente — não uma secret
  global da plataforma — porque, se uma vazar, não compromete os demais
  ([09:21] Sofia). A tabela de configuração do webhook armazena
  `url + secret + customer_id + estado ativo` ([09:21] Bruno).
- **Rotação de secret suportada via API.** Quando o cliente rotaciona, a secret
  antiga permanece válida por **24 horas em paralelo** (grace period), para dar
  tempo de migrar os sistemas; depois disso, a antiga morre ([09:21] Sofia).
- **TLS obrigatório**: a URL cadastrada precisa ser `https`; URL `http` é
  recusada com erro de validação no schema Zod ([09:23] Sofia).

## Alternativas Consideradas

- **Secret global única da plataforma** — descartada por Sofia ([09:21]): se
  uma secret global vaza, compromete todos os clientes; secret por endpoint
  isola o comprometimento a um único endpoint.
- (Algoritmo de assinatura) HMAC-SHA256 foi definido por Sofia ([09:20]) como o
  padrão de mercado; a alternativa de algoritmo mais fraco não sobreviveu à
  discussão de segurança.

## Consequências

**Positivas:**
- Autenticidade e integridade verificáveis pelo cliente ([09:19] Sofia).
- Comprometimento de uma secret afeta apenas um endpoint ([09:21] Sofia).
- Grace period de 24h permite rotação sem janela de indisponibilidade
  ([09:21] Sofia).

**Negativas:**
- Responsabilidade nova de gestão de secrets por endpoint: geração,
  armazenamento seguro e rotação ([09:21] Sofia).
- O cliente precisa implementar a verificação do lado dele; a responsabilidade
  de deduplicação/verificação é compartilhada com o cliente (ver ADR-005).
- Rotação exige coordenação com o cliente dentro da janela de 24 horas.
