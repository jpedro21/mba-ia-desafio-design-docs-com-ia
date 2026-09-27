# ADR-005 — Garantia de entrega at-least-once com `X-Event-Id` para dedup

## Status

Accepted

## Contexto

Em entrega distribuída, é preciso definir o nível de garantia. A plataforma
garante at-least-once: é possível que o cliente receba o **mesmo evento mais de
uma vez**, e ele precisa estar preparado para isso ([09:24] Diego).

## Decisão

- A entrega é **at-least-once** ([09:24] Diego).
- Cada evento carrega um **`X-Event-Id`** (UUID gerado quando o evento entra na
  outbox) único por evento; se o cliente receber duas vezes, ele deduplica pelo
  `event_id` do lado dele ([09:25] Diego).
- Isso é o padrão de mercado — Stripe e GitHub fazem assim — e at-least-once com
  `event_id` resolve 99% dos casos ([09:25] Diego). O PM documenta isso de forma
  destacada no portal de desenvolvedor ([09:26] Marcos, [09:26] Larissa).

## Alternativas Consideradas

- **Exactly-once** — descartada por Diego ([09:25]): exigiria coordenação dos
  dois lados e fica muito mais complexa; at-least-once com `event_id` atende o
  caso real dos clientes. Sofia observa que isso joga responsabilidade para o
  cliente ([09:25] Sofia).

## Consequências

**Positivas:**
- Implementação simples e robusta, alinhada ao padrão de mercado ([09:25] Diego).
- Deduplicação viável do lado do cliente via `X-Event-Id` ([09:25] Diego).
- Requisito de negócio atendido: clientes só querem saber se o pedido deles
  mudou, não exigem exactly-once ([09:14] Marcos).

**Negativas:**
- O cliente pode receber eventos duplicados e precisa implementar dedup do lado
  dele ([09:25] Sofia).
- Exige documentação clara no portal de desenvolvedor para evitar fricção de
  integração ([09:26] Marcos).
