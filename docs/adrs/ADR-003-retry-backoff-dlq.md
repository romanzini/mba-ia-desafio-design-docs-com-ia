# ADR-003: Retry com Backoff Exponencial e Dead Letter Queue

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Resiliência de entrega de webhooks  
**Participantes:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Bruno (Engenheiro Pleno)

## Resumo

Quando a entrega de um evento webhook falha (timeout, 4xx, 5xx), o worker retenta com backoff exponencial: 1m, 5m, 30m, 2h, 12h. Após 5 falhas consecutivas, o evento é movido para Dead Letter Queue (`webhook_dead_letter`) para análise manual e reprocessamento posterior.

## Problema

Webhooks podem falhar por motivos temporários (cliente em manutenção, network flaky, timeout) ou permanentes (client fora do ar indefinidamente). É necessário uma estratégia que:

1. **Tolere falhas temporárias** sem perder eventos (retries).
2. **Não retenha indefinidamente** eventos que nunca conseguirão ser entregues (teto de tentativas).
3. **Preserve evidência** de falhas para debugging e reprocessamento manual (DLQ).

## Decisão

Implementar retry exponencial com limite de 5 tentativas e progressão de backoff:

| Tentativa | Tempo de Espera | Acumulado |
|-----------|-----------------|-----------|
| 1ª falha  | +1 minuto       | 1 min     |
| 2ª falha  | +5 minutos      | 6 min     |
| 3ª falha  | +30 minutos     | 36 min    |
| 4ª falha  | +2 horas        | 2h36m     |
| 5ª falha  | +12 horas       | 14h36m    |

Após a 5ª falha, o evento é movido para `webhook_dead_letter` com payload, motivo de falha (ex: "HTTP 503 após 5 retries"), e timestamp. Uma API admin permite reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay`.

**Tabela `webhook_dead_letter`:**

```
id: UUID (PK)
webhookOutboxId: UUID (FK → webhook_outbox.id)
webhookId: UUID (FK → webhooks.id)
orderId: UUID (FK → orders.id)
eventPayload: JSON (cópia da payload original)
failureReason: String (ex: "HTTP 503 Client Service Unavailable")
lastAttemptAt: DateTime
createdAt: DateTime
replayedAt: DateTime (nullable, preenchido se replayed)
```

## Alternativas Consideradas

### Alternativa 1: Retry Indefinido (Infinite Retry)
**Descrição**: Não ter teto de tentativas; evento fica retentando com backoff infinitamente.

**Trade-off negativo**:
- Evento perdido indefinidamente se cliente sumiu; ninguém percebe.
- DLQ fica vazio; sem mecanismo de visibility de problemas.
- Cliente em manutenção por 2h? Evento retenta por 2 dias. Desperdício de resources.

**Por que foi descartado**: Falta de sinal de falha permanente; impossível detectar cliente dead.

### Alternativa 2: Retry Curto (3 tentativas)
**Descrição**: Teto de 3 tentativas com backoff mais agressivo.

**Trade-off negativo**:
- Cliente em manutenção planejada por 2h? Evento é descartado após ~30 min.
- Casos reais de downtime são comuns (atualizações, certificados que expiram).
- Sem graça de recuperação para cliente que estava só temporariamente indisponível.

**Por que foi descartado**: Período de tolerância muito curto (real de clientes Atlas, MaxDistribuição).

### Alternativa 3: Variável por Webhook
**Descrição**: Permitir cliente escolher número de retries e backoff.

**Trade-off negativo**:
- Adiciona complexidade ao schema de webhook (campos customizáveis).
- Sem padrão único, difícil de documentar e prever.
- Overhead no worker: cada evento com settings diferentes.

**Por que foi descartado**: Prematura otimização; começa com padrão único.

## Consequências

### Positivas

✅ **Tolerância a falhas transientes**: 5 tentativas com spacing de até 14h cobrem indisponibilidades comuns (manutenção planejada, network hiccups).

✅ **Visibilidade de falhas permanentes**: DLQ agrega eventos irrecuperáveis; operação sabe que existe problema.

✅ **Reprocessamento manual**: endpoint de replay permite ao cliente ou ao time retomar entrega após resolver o problema.

✅ **Respeita SLA de cliente**: Atlas quer "abaixo de 10s"; event é retentado dentro de 14h se cliente tiver downtime.

### Negativas

⚠️ **Overhead de armazenamento**: cada retry reescreve linha na outbox; com 5 tentativas, uma linha sofre ~5-10 updates.

⚠️ **Job de housekeeping**: DLQ cresce indefinidamente; requer job de limpeza (adiado para próxima fase).

⚠️ **Ordering não é garantido**: se cliente tem múltiplos eventos retentando em tempos diferentes, pode receber fora de ordem. (Mitigado por `order_id` + single-worker hoje)

## Questões em Aberto

1. **Jitter**: adicionar aleatoriedade (jitter) ao backoff para evitar thundering herd se múltiplos webhooks falham no mesmo momento? (Decidido: não agora; observar em produção)
2. **Escalabilidade com múltiplos workers**: como coordenar retries se há vários workers processando em paralelo? (Decidido: usar lock pessimista ou tabela de lock no futuro)

## Detalhes Técnicos

**Backoff Calculation:**

```typescript
const backoffMs = [60000, 300000, 1800000, 7200000, 43200000]; // ms
const nextRetryAt = now + backoffMs[retryCount];
```

**Timing:**
- Worker busca eventos `WHERE status = PENDING AND nextRetryAt <= NOW()` a cada 2s.
- Evento com `retryCount = 5` é movido para DLQ e removido da outbox.

## Referências

- Transcrição: `[09:15]` Diego propondo 5 retries vs 3.
- Transcrição: `[09:16]` Bruno questionando 3 vs 5; Diego explicando cobertura de 2h de downtime.
- Transcrição: `[09:17]` Larissa e Marcos aceitando 5 retries com backoff 1m/5m/30m/2h/12h.
- Transcrição: `[09:18]` Diego propondo tabela `webhook_dead_letter` separada.
- Transcrição: `[09:18]` Bruno e Larissa sobre endpoint admin para replay.
