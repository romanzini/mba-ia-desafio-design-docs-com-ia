# ADR-001: Padrão Outbox para Entrega de Webhooks

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Sistema de notificação assíncrona de mudanças de pedidos via webhooks  
**Participantes:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)

## Resumo

Adotar o padrão Outbox implementado em tabela MySQL para garantir consistência transacional entre mudanças de status de pedidos e geração de eventos de webhook. Os eventos são inseridos na mesma transação que atualiza o status, garantindo que não há pedidos mudados sem evento correspondente e vice-versa.

## Problema

Quando um pedido muda de status, é necessário notificar webhooks cadastrados de clientes externos. A ingenuidade de fazer HTTP calls síncronos dentro da transação de mudança de status cria problemas:

1. **Aumento de latência transacional**: clientes HTTP lento travam toda a mudança de status.
2. **Acoplamento a disponibilidade externa**: se o webhook do cliente cair, falha o rollback da transação, quebrando a atomicidade do domínio.
3. **Perda de eventos**: se a chamada HTTP falha, o evento é perdido sem retentativa.
4. **Perda de garantia ACID**: não há garantia de que um evento gerado foi realmente entregue.

## Decisão

Implementar o padrão Outbox usando tabela `webhook_outbox` no MySQL existente. Quando um status de pedido muda (na transação `Order.changeStatus`), o evento é inserido na `webhook_outbox` dentro da mesma transação SQL. Um worker separado lê periodicamente a tabela e dispara HTTP calls. Se a transação principal falha, o evento é descartado junto. Se a transação sucede, o evento é garantidamente persistido.

**Tabela `webhook_outbox` (definida em migration Prisma):**

```
id: UUID (PK)
webhookId: UUID (FK → webhooks.id)
orderId: UUID (FK → orders.id)
eventType: String (ex: 'order.status_changed')
eventPayload: JSON (snapshot do evento)
status: PENDING | PROCESSING | DELIVERED | FAILED
failureReason: String (nullable)
retryCount: Int (default 0)
maxRetries: Int (default 5)
createdAt: DateTime
updatedAt: DateTime
processedAt: DateTime (nullable)
indexes: (status, createdAt), (webhookId), (orderId)
```

A `eventPayload` é renderizada e congelada no momento da inserção. Se o pedido é alterado depois, o evento reflete o estado no momento da mudança de status, não o estado atual.

## Alternativas Consideradas

### Alternativa 1: Redis Streams
**Descrição**: Usar fila em Redis Streams em vez de tabela MySQL para armazenar eventos.

**Trade-off negativo**: 
- Requer infraestrutura Redis dedicada (Cluster em HA).
- Time é pequeno, não justifica manutenção de outra tecnologia.
- Redis Streams sem persistência RDB não garante durabilidade em falhas.
- Aumenta complexidade operacional em produção.

**Por que foi descartado**: Overengineering para escala atual. MySQL + polling atende requisito de latência (10s).

### Alternativa 2: HTTP Síncrono dentro da Transação
**Descrição**: Fazer HTTP call direto no método `changeStatus`, dentro da transação.

**Trade-off negativo**:
- Acoplamento a disponibilidade de webhook de cliente; falha de cliente pode travar mudança de status de outros pedidos.
- Se webhook está offline, há dilema: rollback a mudança de status (quebra invariante do domínio) ou ignora falha (perde evento).
- Latência transacional cresce com latência de network.
- Impossível implementar retries sem reabrir transações.

**Por que foi descartado**: Viola isolamento transacional e cria cascata de falhas.

### Alternativa 3: Trigger de Banco de Dados
**Descrição**: Usar trigger SQL no MySQL para notificar worker de novo evento.

**Trade-off negativo**:
- MySQL não tem `NOTIFY/LISTEN` nativo (só Postgres tem).
- Alternativa improvisada (escrever arquivo ou bater endpoint) fica esquisita e difícil de manter.
- Polling com 2s resolve o requisito sem complexidade.

**Por que foi descartado**: Complexidade injustificada; polling funciona bem.

## Consequências

### Positivas

✅ **Garantia de consistência**: todo pedido com mudança de status tem evento correspondente persistido atomicamente.

✅ **Desacoplamento**: mudança de status não depende de disponibilidade de webhook externo.

✅ **Resiliência**: eventos persistidos em banco; worker pode reprocessar com retries sem perder dados.

✅ **Observabilidade**: cada evento na outbox fica registrado, permite audit trail e debugging.

✅ **Escalabilidade inicial**: single table no MySQL existente, sem nova infraestrutura.

### Negativas

⚠️ **Latência mínima**: evento entra na outbox instantaneamente, mas entrega via webhook leva tempo do polling (até 2s no pior caso).

⚠️ **Duplicação possível**: at-least-once guarantee requer deduplicação no lado do cliente (vide ADR-005).

⚠️ **Gerenciamento de tabela**: sem arquivamento, `webhook_outbox` cresce indefinidamente. Requer job de limpeza (adiado para fase 2).

⚠️ **Query pattern**: tabela de eventos cresce; índices devem ser bem planejados para não degradar performance.

## Questões em Aberto

1. **Archival policy**: quando remover eventos processados da tabela? (Adiado para próxima fase; Bruno sugeriu 30 dias)
2. **Particionamento futuro**: se escalar a múltiplos workers, como particionar eventos por `order_id`? (Decidido como não crítico agora)

## Referências

- Transcrição: `[09:06]` Diego sobre Outbox
- Transcrição: `[09:07]` Larissa sobre alternativas (Redis vs Outbox)
- Transcrição: `[09:08]` Diego sobre índices e performance
- Código: `src/modules/orders/order.service.ts` - método `changeStatus` que incorporará insert na outbox
- Código: `src/config/database.ts` - PrismaClient existente para compartilhamento com worker
