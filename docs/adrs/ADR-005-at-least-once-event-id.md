# ADR-005: Garantia At-Least-Once com Deduplicação por Event ID

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Semântica de entrega de webhooks  
**Participantes:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Sofia (Engenheira de Segurança)

## Resumo

O sistema fornece semântica **at-least-once**: cada evento de webhook é garantido ser entregue pelo menos uma vez, mas pode ser entregue mais de uma vez. Cliente é responsável por deduplicação no seu lado, usando o header `X-Event-Id` que contém um UUID único por evento.

Padrão de indústria: Stripe, GitHub, Twilio, AWS SNS adotam at-least-once, não exactly-once.

## Problema

Webhooks são executados em ambiente distribuído (network pode falhar, procesos podem crashar). Garantir **exatamente uma** entrega (exactly-once) é extremamente complexo:

1. **Exactly-once requer coordenação**: servidor teria que confirmar recebimento do cliente, esperar ack, etc. Cria latência e complexidade.
2. **Overhead de estado**: precisaria manter estado de "qual cliente já recebeu este evento" indefinidamente.
3. **Impossível em rede com partição**: teorema FLP dita que em rede assíncrona com falhas, não dá garantir atomicidade global.

A alternativa (at-least-once) é mais viável e simples:

- **Servidor**: dispara webhook. Se não receber resposta (timeout, erro), retenta (vide ADR-003).
- **Cliente**: assume que pode receber o mesmo evento N vezes. Implementa deduplicação.

Exemplo real: pedido muda para SHIPPED, evento gerado com `event_id = uuid-123`. Worker envia. Cliente recebe com sucesso. Mas ACK não chega no worker (rede flakey). Worker acha que falhou e retenta. Cliente recebe novamente com `event_id = uuid-123`. Cliente detecta pelo ID que é duplicado e ignora.

## Decisão

Implementar semântica **at-least-once** com suporte a deduplicação via `X-Event-Id`:

1. **Cada evento gerado** na `webhook_outbox` recebe um UUID único (`event_id`).
   ```sql
   INSERT INTO webhook_outbox (id, webhook_id, event_id, event_payload, status)
   VALUES (uuid(), webhook_id, uuid(), '{"order_id": "..."}', 'PENDING');
   ```
   - `id`: identifica a linha na outbox (interno, sem exposição).
   - `event_id`: identificador do evento de negócio (exposto ao cliente via header).

2. **Header X-Event-Id**: toda requisição HTTP do webhook inclui:
   ```
   X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
   ```
   - Cliente armazena este ID após processar o evento.
   - Se receber requisição com mesmo `event_id`, cliente sabe que é retry e deduplica.

3. **Semantics**:
   - **Entrega com sucesso**: cliente recebe, processa, responde `2xx`. Worker marca como `DELIVERED`.
   - **Entrega com falha**: cliente não responde ou responde `5xx`. Worker marca para retry.
   - **Duplicação**: cliente recebe evento com `event_id` que já processou. Cliente responde `2xx` (ignora duplicado) ou responde com erro (worker retenta). O ideal é cliente responder `2xx` mesmo se duplicado (para evitar retry loop).

## Alternativas Consideradas

### Alternativa 1: Exactly-Once Delivery
**Descrição**: Garantir que cliente receba evento exatamente uma vez, sem duplicação.

**Trade-off negativo**:
- Requer coordenação de 2-phase commit (preparação + confirmação).
- Cliente teria que manter estado de "eventos já processados" globalmente.
- Se cliente falha após processar mas antes de enviar ack, evento fica em limbo.
- Impossível garantir em rede com partição (impossibilidade de FLP).
- Latência sobe: teria que esperar ack do cliente antes de considerar sucesso.

**Por que foi descartado**: Complexidade injustificada. Padrão de mercado é at-least-once.

### Alternativa 2: At-Most-Once (Best Effort)
**Descrição**: Disparar evento uma vez, sem retries. Se falhar, ignora.

**Trade-off negativo**:
- Cliente que se desconecta por 10 segundos perde eventos.
- Nenhuma garantia de durabilidade.
- Atlas quer "tempo real" reliável; isso não oferece confiabilidade.

**Por que foi descartado**: Não atende requisito de reliability (clientes exigem durabilidade).

## Consequências

### Positivas

✅ **Simplicidade**: cliente implementa deduplicação simples (hashset de `event_id`s processados).

✅ **Durabilidade**: evento é retentado até sucesso. Cliente mesmo que offline por horas não perde eventos.

✅ **Padrão de mercado**: Stripe, GitHub, Twilio usam at-least-once. Cliente já tem código/libs pra deduplicação.

✅ **Sem overhead de coordenação**: servidor não precisa esperar ack; retorna logo após envio.

### Negativas

⚠️ **Deduplicação no cliente**: cliente é responsável por implementar deduplicação. Se não implementa, vai processar evento múltiplas vezes.

⚠️ **Duplicação possível**: cliente pode receber o mesmo evento N vezes. Precisa estar preparado (idempotência).

⚠️ **Armazenamento de IDs**: cliente precisa manter registry de `event_id`s já processados (pode crescer indefinidamente).

## Questões em Aberto

1. **TTL de deduplicação no cliente**: quanto tempo cliente deve guardar `event_id` de eventos já processados? (Recomendação: pelo menos 24h; adiado para docs de integração)
2. **Idempotência do cliente**: como garantir que reprocessar evento não causa efeitos colaterais? (Out of scope; é responsabilidade do cliente, não do OMS)

## Detalhes Técnicos

**Header Format:**
```
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
```
- UUID v4 standard format (36 caracteres com hífens).
- Gerado no servidor quando evento entra na outbox.
- Idem para todos os retries do mesmo evento (reusa o mesmo `event_id`).

**Cliente-side Deduplication (Pseudocódigo):**
```python
@webhook_handler
def handle_webhook(event_id, payload):
  if event_id in processed_events:
    return 200  # já processado, ignora duplicado
  
  # processar payload...
  processed_events.add(event_id)
  return 200
```

**Storage em Outbox:**
```sql
ALTER TABLE webhook_outbox ADD COLUMN event_id CHAR(36) NOT NULL UNIQUE;
-- índice para lookup rápido de evento por ID
CREATE INDEX idx_webhook_outbox_event_id ON webhook_outbox(event_id);
```

## Referências

- Transcrição: `[09:24]` Diego explicando at-least-once com `X-Event-Id`.
- Transcrição: `[09:25]` Sofia sobre responsabilidade do cliente em deduplicação.
- Transcrição: `[09:25]` Diego comparando com padrão de Stripe/GitHub.
- Transcrição: `[09:26]` Marcos sobre documentação destacada pra clientes.
