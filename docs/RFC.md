# RFC: Sistema de Webhooks de Notificação de Pedidos

**Status:** Proposto para Revisão  
**Data:** 2026-07-06  
**Autor:** Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)  
**Revisores:** Larissa (Tech Lead), Sofia (Engenheira de Segurança), Marcos (Product Manager)  
**Prazo de Review:** 1 semana

---

## TL;DR (Resumo Executivo)

Implementar **notificação assíncrona em tempo real** para clientes B2B quando o status de pedidos muda na plataforma. Usar padrão **Outbox no MySQL** (vs Redis) para garantir consistência transacional, **worker separado em polling** (vs embarcado) para desacoplamento operacional, e **HMAC-SHA256** com secret por endpoint para autenticação segura. Semântica **at-least-once** com deduplicação por `X-Event-Id`.

Todas as decisões arquiteturais estão documentadas em ADRs específicos. Máxima reutilização de padrões existentes do projeto.

---

## Contexto e Motivação

### Problema de Negócio

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitam notificação em tempo real quando status de pedidos muda na plataforma. Hoje, clientes fazem polling manual no endpoint `GET /orders`, causando:

- **Latência de integração**: clientes não sabem quando status mudou; batem a API de tempos em tempos.
- **Custos operacionais**: polling constante gasta banda e CPU de ambos os lados.
- **Risco de churn**: Atlas ameaçou migração para concorrente se feature não sair até fim do trimestre.

**Requisito de negócio**: entregar webhooks de notificação abaixo de 10 segundos de latência.

### Alinhamento de Escopo

📌 **Incluso nesta proposta**:
- Notificação assíncrona para mudança de status de pedidos.
- Suporte a múltiplos clientes com múltiplos webhooks cada.
- Reprocessamento manual via API admin para eventos em DLQ.
- Validação de URL (https obrigatório).
- Autenticação via HMAC-SHA256.

📌 **Explicitamente Fora de Escopo** (para fase 2):
- Email ou SMS como fallback de notificação.
- Dashboard visual no portal de cliente.
- Rate limiting de saída (observar em produção primeiro).

---

## Proposta Técnica

### Visão Geral da Arquitetura

```
┌─────────────────────────────────────┐
│   API (src/server.ts)               │
│  ┌─────────────────────────────────┐│
│  │ OrderService.changeStatus()     ││
│  │ (mudança de status + persistência)
│  │                                  ││
│  │ Dentro da transação:            ││
│  │ 1. atualiza order.status        ││
│  │ 2. insere orderStatusHistory    ││
│  │ 3. debita/repõe estoque         ││
│  │ 4. ✨ insere na webhook_outbox  ││
│  └─────────────────────────────────┘│
└──────────────────┬──────────────────┘
                   │ (mesma transação)
                   ▼
        ┌──────────────────┐
        │  MySQL Database  │
        │  webhook_outbox  │
        │  status=PENDING  │
        └────────┬─────────┘
                 │
    (polling a cada 2s)
                 ▼
┌─────────────────────────────────────┐
│   Worker (src/worker.ts)            │
│  ┌─────────────────────────────────┐│
│  │ SELECT * FROM webhook_outbox    ││
│  │   WHERE status = PENDING        ││
│  │   ORDER BY createdAt ASC        ││
│  │   LIMIT 10                      ││
│  │                                  ││
│  │ Para cada evento:               ││
│  │ POST webhook.url com:           ││
│  │   - X-Signature: HMAC-SHA256   ││
│  │   - X-Event-Id: UUID (dedup)   ││
│  │   - body: {event_type, order..}││
│  │                                  ││
│  │ Se sucesso: status=DELIVERED    ││
│  │ Se falha: retry com backoff     ││
│  │ Se 5 falhas: mover para DLQ     ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
                   │
                   ├─────────────────────────────┐
                   ▼                             ▼
            ┌───────────────┐         ┌──────────────────┐
            │  Cliente HTTP │         │  webhook_dlq     │
            │  2xx ✅       │         │  (reprocessamento)
            └───────────────┘         └──────────────────┘
                                              │
                                      (POST /admin/...
                                       /dead-letter/:id
                                       /replay)
```

### Componentes Principais

#### 1. **Tabela webhook_outbox** (ADR-001)
- Armazena eventos pendentes de entrega.
- Inserida atomicamente na transação de `changeStatus`.
- Worker processa e marca como `DELIVERED` ou move para DLQ.
- Índices: `(status, createdAt)`, `(webhookId)`.

#### 2. **Worker Separado** (ADR-002)
- Executa em processo Node.js diferente (`src/worker.ts`).
- Loop infinito: a cada 2s, busca eventos pendentes em batch.
- Dispara HTTP POST para cada webhook cadastrado.
- Pode rodar em host/container separado (desacoplado da API).

#### 3. **Retry com Backoff e DLQ** (ADR-003)
- 5 tentativas: 1m, 5m, 30m, 2h, 12h.
- Cobertura até 14h de indisponibilidade.
- Após 5 falhas, evento vai para `webhook_dead_letter`.
- Endpoint admin `POST /admin/webhooks/dead-letter/:id/replay` recoloca na outbox.

#### 4. **Autenticação HMAC-SHA256** (ADR-004)
- Secret única por webhook.
- Worker assina payload com secret: `X-Signature: sha256=<hmac>`.
- Cliente valida no seu lado.
- Rotação de secret com grace period 24h.
- TLS obrigatório (https:// apenas).

#### 5. **At-Least-Once + X-Event-Id** (ADR-005)
- Cada evento recebe UUID único na geração.
- Enviado no header `X-Event-Id`.
- Cliente deduplica no seu lado.
- Padrão de mercado (Stripe, GitHub, Twilio).

#### 6. **Reutilização de Padrões Existentes** (ADR-006)
- Module `src/modules/webhooks/` com structure repository-service-controller.
- Erros herdam de `AppError` com prefixo `WEBHOOK_`.
- Logger Pino, error middleware centralizado, schemas Zod.
- Nenhuma nova dependência.

---

## Alternativas Consideradas

### Alternativa A: Redis Streams (vs Outbox MySQL)

**Proposta**: Usar Redis Streams em vez de tabela MySQL para fila de eventos.

**Trade-offs**:
- ✅ **Latência**: Redis é mais rápido que MySQL.
- ❌ **Infraestrutura**: requer Redis Cluster em HA; time é pequeno.
- ❌ **Durabilidade**: sem RDB, eventos podem ser perdidos em crash.
- ❌ **Complexidade operacional**: outra tecnologia para manter.

**Por que foi descartada**: Overengineering. MySQL resolve com requisito de latência (≤10s). Vide **ADR-001**.

### Alternativa B: Worker Embarcado na API (vs Processo Separado)

**Proposta**: Rodar worker como loop dentro do mesmo processo da API.

**Trade-offs**:
- ✅ **Simplicidade**: sem orquestração de processos.
- ❌ **Resiliência**: API reinicia → worker para → eventos acumulam.
- ❌ **Escalabilidade**: worker não pode rodar em máquina separada.
- ❌ **Observabilidade**: logs e crashes da API e worker misturados.

**Por que foi descartada**: Quebra isolamento de falhas. Vide **ADR-002**.

### Alternativa C: HTTP Síncrono dentro de changeStatus (vs Outbox)

**Proposta**: Chamar webhook.url diretamente na transação de mudança de status.

**Trade-offs**:
- ✅ **Latência**: webhook é disparado imediatamente.
- ❌ **Acoplamento**: falha de cliente externo quebra mudança de status.
- ❌ **Rollback dilema**: cliente offline → rollback status (quebra invariante) ou ignora falha (perde evento)?
- ❌ **Sem retries**: timeout do cliente trava toda a transação.

**Por que foi descartada**: Viola isolamento transacional e cria cascata de falhas. Vide **ADR-001**.

---

## Questões em Aberto

### Q1: Ordenação Global entre Clientes
**Problema**: Com single worker, eventos de um cliente são processados em ordem. Mas eventos de diferentes clientes podem ficar fora de ordem.

**Impacto**: Baixo. Cada cliente vê seus próprios eventos em ordem por `order_id`.

**Decisão futura**: Se houver reclamação, implementar particionamento por `order_id` ou lock pessimista (fase 2).

**Status**: Registrado como limitação conhecida na documentação de cliente.

### Q2: Escalabilidade a Múltiplos Workers
**Problema**: Com vários workers processando em paralelo, como garantir que um evento não é processado 2 vezes?

**Impacto**: Médio. Duplicação causaria webhook disparado 2x ao cliente.

**Opção A**: Lock pessimista na outbox (`SELECT FOR UPDATE`).
**Opção B**: Particionar eventos por `order_id` (determinístico).

**Status**: Decidido como "observar em produção"; não crítico agora (single worker atende).

### Q3: Rate Limiting de Saída
**Problema**: Se cliente tem 50 pedidos mudando de status em 1 min, worker dispara 50 webhooks ao cliente simultaneamente. Cliente pode sofrer DDoS involuntário.

**Impacto**: Baixo em curto prazo. Observar em produção com Atlas.

**Opção A**: Throttle de envios por webhook (max N requisições/min).
**Opção B**: Batch events (agrupa múltiplos eventos em um request).

**Status**: Registrado como "ponto em aberto"; implementar em fase 2 se necessário.

### Q4: Rotação de Secret Sem Grace Period Longo
**Problema**: Se secret vaza, há 24h antes de expirar. Atacante poderia forjar webhooks.

**Mitigação**: 
- Cliente que detecta vazamento pode rotacionar novamente (novo grace period).
- TLS garante que payload em trânsito não é interceptado.
- Padrão de mercado é grace period de 24-48h.

**Status**: Decidido como acceptable; monitorar em produção.

---

## Impacto e Riscos

### Impactos Positivos

✅ **Retenção de clientes**: Atlas, Max, Nova Cargo satisfeitos. Reduz risco de churn.

✅ **Diferencial competitivo**: notificação em tempo real é feature premium em OMS market.

✅ **Escalabilidade do cliente**: clientes eliminam polling, reduz custos de integração.

### Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|--------|-----------|
| Worker falha e eventos acumulam | Média | Alto | Health check + monitoring + alertas. Reativar worker rapidamente. |
| DLQ cresce indefinidamente | Alta | Médio | Job de limpeza de DLQ em 30 dias (fase 2). |
| Duplicação chega ao cliente | Média | Médio | Documentar `X-Event-Id` claramente. Cliente implementa deduplicação. |
| Secret vaza | Baixa | Alto | Rotação de secret com grace period. Cliente pode rotacionar novamente se necessário. |
| Cliente com webhook offline por >14h perde eventos | Baixa | Médio | Razoável; qualquer downtime >14h é problema sério do cliente. Suportar replay manual. |

---

## Dependências

### Dependências Internas

- ✅ **Database**: MySQL existente (nenhuma nova infra).
- ✅ **Prisma**: já integrado; adiciona migrations para outbox + dlq.
- ✅ **Node.js runtime**: worker é processo Node.js (já temos).
- ✅ **Padrões de código**: reutilizam AppError, Pino, schemas Zod.

### Dependências Externas

- ⚠️ **Cliente HTTP**: worker faz requisições HTTP outbound. Requer permissão de egress para IPs de cliente.
- ⚠️ **TLS certificates**: cliente precisa ter HTTPS válido. Validamos em schema (https://).

### Dependências Cross-Time

- 🤝 **Security review**: Sofia (Engenheira de Segurança) revisa código de HMAC + geração de secret antes de deploy.
- 🤝 **Product docs**: Marcos (PM) escreve docs de integração para portal de cliente.

---

## Decisões Relacionadas (ADRs)

Este RFC é suportado pelas seguintes Architecture Decision Records:

| ADR | Título | Status |
|-----|--------|--------|
| [ADR-001](./adrs/ADR-001-outbox-pattern-mysql.md) | Padrão Outbox para Entrega de Webhooks | Decidido |
| [ADR-002](./adrs/ADR-002-worker-polling-separate-process.md) | Worker em Polling Separado do Processo API | Decidido |
| [ADR-003](./adrs/ADR-003-retry-backoff-dlq.md) | Retry com Backoff Exponencial e Dead Letter Queue | Decidido |
| [ADR-004](./adrs/ADR-004-hmac-sha256-authentication.md) | Autenticação de Webhook via HMAC-SHA256 | Decidido |
| [ADR-005](./adrs/ADR-005-at-least-once-event-id.md) | Garantia At-Least-Once com Deduplicação por Event ID | Decidido |
| [ADR-006](./adrs/ADR-006-reuse-existing-patterns.md) | Reutilização de Padrões e Infra Existentes do Projeto | Decidido |

---

## Timeline e Estimativa

**Estimativa de Larissa (Tech Lead):**
- 3 sprints totais (incluindo revisão de segurança de Sofia).
- Sprint 1: Modelagem de outbox + DLQ, worker + retry.
- Sprint 2: CRUD de configuração, histórico de deliveries, integração com `changeStatus`.
- Sprint 3: HMAC, schemas, validações, testes ponta-a-ponta, revisão de Sofia.

**Deadline**: fim de novembro (confirmado com Atlas).

---

## Próximos Passos

1. ✅ **Review deste RFC** pelos revisores (1 semana).
2. ⏳ **Consolidar feedback** e atualizar ADRs conforme necessário.
3. ⏳ **Iniciar FDD** (Feature Design Document) com detalhes de implementação.
4. ⏳ **Começar Sprint 1** com modelagem de outbox.

---

## Referências

**Documentos relacionados:**
- `TRANSCRICAO.md` - Reunião técnica com discussões de todas as decisões.
- `docs/adrs/` - ADRs detalhadas.
- `src/modules/orders/order.service.ts` - Padrão de integração (changeStatus).

**Padrões de mercado citados:**
- Stripe Webhooks: https://stripe.com/docs/webhooks
- GitHub Webhooks: https://docs.github.com/en/developers/webhooks-and-events/webhooks
- Twilio Webhooks: https://www.twilio.com/docs/usage/webhooks
