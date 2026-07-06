# PRD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0  
**Data:** 2026-07-06  
**Product Owner:** Marcos (Product Manager)  
**Tech Lead:** Larissa

---

## Resumo Executivo

Implementar um **sistema de webhooks de notificação em tempo real** para clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) que desejam ser notificados quando o status de seus pedidos muda na plataforma. Hoje, clientes fazem polling manual no endpoint `GET /orders` a cada minuto, gerando latência, custos operacionais e risco de churn. Webhooks eliminam a necessidade de polling e reduzem latência de integração para abaixo de 10 segundos.

---

## Contexto e Motivação

### Problema

Três clientes B2B solicitaram formalmente notificações em tempo real quando status de pedidos muda:

- **Atlas Comercial**: grande cliente, integração crítica. Ameaçou migração se feature não sair até fim de novembro.
- **MaxDistribuição**: cliente B2B em crescimento.
- **Nova Cargo**: cliente de logística.

**Impactos atuais:**
- Polling manual → latência de integração (minutos até horas).
- Custos operacionais aumentados (banda, CPU de ambos os lados).
- Risco de perda de clientes (Atlas especificamente ameaçou churn).

### Alinhamento de Negócio

- **Retenção**: manter clientes B2B satisfeitos evita churn.
- **Diferencial**: notificação em tempo real é feature premium em OMS market.
- **Escalabilidade**: clientes eliminam polling, reduz pressão na API.

---

## Público-Alvo e Cenários de Uso

### Público-Alvo

**Primário**: Clientes B2B do OMS que integram automaticamente pedidos (varejistas, distribuidoras, operadores logísticos).

**Secundário**: Integradores de terceiros que constroem soluções no topo do OMS.

### Cenários de Uso

1. **Atualização em tempo real de status de pedido**
   - Pedido muda de PENDING → PAID → PROCESSING → SHIPPED → DELIVERED
   - Cliente recebe notificação para cada mudança
   - Cliente atualiza seu próprio banco de dados sem polling

2. **Filtro seletivo de status**
   - Cliente só quer ser notificado de SHIPPED e DELIVERED (não quer ruído)
   - Configura webhook para receber só esses eventos
   - Reduz tráfego e processamento do lado do cliente

3. **Múltiplos webhooks por cliente**
   - Cliente tem 3 sistemas diferentes (warehouse, financeiro, CRM)
   - Cadastra 3 webhooks diferentes, cada um recebendo subset de eventos
   - Cada sistema recebe só o que precisa

4. **Reprocessamento manual após indisponibilidade**
   - Cliente esteve offline por 4 horas em manutenção planejada
   - Eventos acumulam em fila (DLQ)
   - Após recuperação, cliente autoriza replay manual via API admin
   - Eventos são retransmitidos

---

## Objetivos e Métricas de Sucesso

### Objetivo Primário

Entregar **notificação de mudança de status de pedidos em tempo real** (latência ≤ 10 segundos) de forma **confiável** (at-least-once) e **segura** (autenticação + integridade).

### Métricas de Sucesso

| Métrica | Target | Observação |
|---------|--------|-----------|
| **Latência P99 de entrega** | ≤ 10 segundos | Medido do momento da mudança de status até chegada da notificação |
| **Taxa de sucesso de entrega** | ≥ 98% (em 24h) | Eventos que chegam com sucesso em até 24h (cobrindo DLQ + retry) |
| **Retenção de clientes B2B** | 100% (Atlas, Max, Nova) | Clientes continuam (ameaça de churn eliminada) |
| **Taxa de reprocessamento de DLQ** | ≥ 95% | Eventos em DLQ são successfully replayed após resolução |
| **Satisfação de cliente** | NPS ≥ 8/10 | Survey com clientes B2B após launch |

### Definição de Sucesso

Feature é considerada bem-sucedida quando:
1. Atlas, MaxDistribuição e Nova Cargo estão recebendo notificações de pedidos em tempo real.
2. Taxa de entrega é ≥ 98% em 24h (clientes conseguem recuperar de outages via DLQ).
3. Nenhuma reclamação de segurança (autenticação, integridade de payload).
4. Latência medida em produção ≤ 10s (P99).

---

## Requisitos Funcionais

### RF-01: Cadastro de Webhook
- Cliente autenticado pode criar múltiplos webhooks
- Fornece: URL (deve ser HTTPS), lista de eventos que quer receber
- Recebe: webhook ID, secret (retornado uma única vez)
- Validação: URL deve ser HTTPS válido

### RF-02: Listagem de Webhooks
- Cliente vê todos seus webhooks cadastrados
- Inclui: ID, URL, events, status (ativo/inativo), data de criação
- Paginação: 10 webhooks por página

### RF-03: Atualização de Webhook
- Cliente pode editar: URL, lista de events, status (ativar/desativar)
- Alterações entram em efeito imediatamente (próximo evento)

### RF-04: Deleção de Webhook
- Cliente pode remover webhook
- Webhook é marcado como inativo (soft delete)
- Nenhum evento futuro será enviado para esse webhook

### RF-05: Filtragem de Eventos
- Cada webhook tem lista de eventos que quer receber (ex: [SHIPPED, DELIVERED])
- Quando status de pedido muda, evento só é enfileirado se novo status está na lista
- Reduz tráfego desnecessário

### RF-06: Notificação de Mudança de Status
- Quando OrderService.changeStatus é chamado, um evento é gerado
- Evento contém: order ID, order number, customer ID, status anterior, status novo, timestamp, payload com dados do pedido
- Evento é persistido na outbox dentro da mesma transação de mudança de status

### RF-07: Entrega de Webhooks
- Worker lê eventos da outbox a cada 2 segundos
- Para cada evento: faz HTTP POST para webhook.url
- Headers: X-Signature (HMAC), X-Event-Id (para dedup), X-Timestamp, X-Webhook-Id
- Timeout: 10 segundos por requisição

### RF-08: Retry com Backoff
- Se entrega falha: retry com backoff exponencial
- Tentativas: 5 (1m, 5m, 30m, 2h, 12h)
- Após 5 falhas: evento vai para Dead Letter Queue

### RF-09: Dead Letter Queue (DLQ)
- Eventos que falharam 5 vezes são movidos para DLQ
- DLQ armazena: payload original, motivo da falha, timestamp
- Persiste para análise e reprocessamento manual

### RF-10: Replay Manual de DLQ
- Endpoint admin: POST /admin/webhooks/dead-letter/:id/replay
- Requer role ADMIN
- Move evento de volta para outbox como PENDING
- Evento será retentado com backoff normal

### RF-11: Histórico de Entregas
- Cada entrega é registrada em webhook_deliveries
- Inclui: success/failure, response status, payload enviado, response recebida, duração, tentativa número
- Cliente pode consultar histórico: GET /webhooks/:id/deliveries

### RF-12: Rotação de Secret
- Endpoint: POST /webhooks/:id/rotate-secret
- Gera novo secret, marca antigo como "rotated"
- Durante 24h: ambos os secrets são válidos (grace period)
- Após 24h: secret antigo expira, só novo é válido
- Permite ao cliente rotar em seus sistemas sem perder eventos

### RF-13: Autenticação de Webhook
- Cada requisição HTTP é assinada com HMAC-SHA256
- Secret por endpoint (não global)
- Cliente valida assinatura no seu lado
- Garante autenticidade e integridade do payload

---

## Requisitos Não Funcionais

### RNF-01: Latência
- Latência ponta-a-ponta ≤ 10 segundos (P99)
- Inclui: detecção do evento (polling 2s) + processamento + HTTP call

### RNF-02: Confiabilidade
- Semântica at-least-once (eventos são garantidamente entregues pelo menos uma vez)
- Cliente é responsável por deduplicação (X-Event-Id)
- Eventos não são perdidos (persistidos em outbox antes de worker processar)

### RNF-03: Segurança
- HMAC-SHA256 obrigatório
- Secret por endpoint
- TLS obrigatório (https://)
- Secrets não são logados em plain text
- Auditoria de rotação de secrets

### RNF-04: Escalabilidade
- Worker roda em processo separado (pode ser escalado independentemente)
- Suporta ~100 webhooks configurados
- Suporta ~1000 eventos/minuto em DLQ (fase 1; fase 2 melhorar)

### RNF-05: Observabilidade
- Logs estruturados com Pino
- Métricas de sucesso/falha de entrega
- Tracing de eventos (fase 2)

### RNF-06: Durabilidade
- Eventos persistidos em MySQL (durabilidade)
- Sem dependência de Redis ou outra cache (fase 1)

### RNF-07: Resiliência a Falhas
- Falha de cliente não afeta mudança de status (assíncrono)
- Falha de worker não afeta API (processo separado)
- Downtime de cliente é tolerado via retry (até 24h)

---

## Decisões e Trade-offs Principais

### Decisão 1: Outbox Pattern vs Redis Streams
**Escolha**: Outbox no MySQL  
**Trade-off**: Menos performance que Redis, mas sem nova infraestrutura. MySQL existente suficiente.

### Decisão 2: Worker Separado vs Embarcado
**Escolha**: Worker em processo separado (src/worker.ts)  
**Trade-off**: Mais complexidade operacional, mas resiliência e escalabilidade.

### Decisão 3: At-Least-Once vs Exactly-Once
**Escolha**: At-least-once com X-Event-Id  
**Trade-off**: Simplicidade. Cliente implementa deduplicação (padrão de mercado).

### Decisão 4: HMAC vs OAuth
**Escolha**: HMAC-SHA256  
**Trade-off**: Mais simples, padrão de mercado (Stripe, GitHub).

### Decisão 5: Retry Automático vs Manual
**Escolha**: Automático com 5 tentativas + DLQ + replay manual  
**Trade-off**: Cobertura de até 24h. Eventos perdidos irreversivelmente são raros.

---

## Escopo

### Incluso ✅

- Notificação de mudança de status de pedidos (todos os 6 status)
- CRUD de webhooks (criar, listar, atualizar, deletar)
- Filtro de eventos por webhook
- Autenticação e integridade (HMAC-SHA256)
- Retry automático com backoff
- Dead Letter Queue para falhas permanentes
- Replay manual de DLQ (admin)
- Histórico de entregas
- Rotação de secret com grace period
- Testes de integração
- Documentação de integração para cliente

### Fora de Escopo (Fase 2+) ❌

- Email ou SMS como fallback de notificação (RF descartado em `[09:37]`)
- Dashboard visual no portal de cliente (decidido fora em `[09:40]`)
- Rate limiting de saída (observar em produção antes de implementar; `[09:38]`)
- Múltiplos workers com escalabilidade horizontal (observar depois)
- Event sourcing full (auditoria de histórico de eventos)
- Webhooks inbound (cliente → servidor; fora de escopo)

---

## Dependências

### Dependências de Produto
- ✅ Dados de pedidos (existentes): Order, OrderStatus, OrderStatusHistory
- ✅ Dados de clientes (existentes): Customer
- ✅ Autenticação JWT (existente)
- ✅ Padrão de módulos (existente)

### Dependências Técnicas
- ✅ Banco MySQL (existente)
- ✅ Prisma ORM (existente)
- ✅ Node.js 18+ (existente)

### Dependências Cross-Time
- 🤝 Review de segurança de Sofia (HMAC + geração de secret)
- 🤝 Documentação de integração de Marcos
- 🤝 Testes do time de QA

---

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|--------|-----------|
| Worker falha e eventos acumulam | Média | Alto | Monitoração 24/7, alertas imediatos, runbook de recovery |
| DLQ cresce indefinidamente | Alta | Médio | Job de arquivamento em 30 dias (fase 2) |
| Cliente não implementa deduplicação | Média | Médio | Documentação destacada, exemplos de código, suporte |
| Secret vaza | Baixa | Alto | Rotação com grace period, audit log, cliente pode rotacionar novamente |
| Taxa de entrega < 95% | Baixa | Alto | Observar em produção, ajustar backoff se necessário |
| Cliente offline >24h perde eventos | Muito Baixa | Médio | Razoável (downtime>24h é problema sério do cliente); suporte manual |

---

## Critérios de Aceitação

- [ ] Webhook é criado com secret gerado com sucesso
- [ ] Pedido que muda de status gera evento na outbox atomicamente
- [ ] Worker processa eventos e envia HTTP POST com headers corretos (X-Signature, X-Event-Id)
- [ ] Evento é retentado com backoff correto se entrega falha
- [ ] Evento é movido para DLQ após 5 falhas
- [ ] Endpoint admin permite replay de DLQ
- [ ] Cliente consegue rotacionar secret com grace period de 24h
- [ ] Latência medida em produção é ≤ 10s (P99)
- [ ] Taxa de sucesso de entrega em 24h é ≥ 98%
- [ ] Documentação de integração está disponível para cliente
- [ ] Sofia (Segurança) aprova implementação de HMAC + secret

---

## Estratégia de Testes e Validação

### Testes Unitários
- Schemas Zod (validação de URL, events)
- Cálculo de HMAC
- Lógica de backoff
- Filtro de eventos

### Testes de Integração
- Criar webhook → mudar status → evento entregue com sucesso
- Simular falha de cliente → retry automático → sucesso na 2ª tentativa
- 5 falhas consecutivas → DLQ
- Replay de DLQ → evento reenviado
- Rotação de secret → ambas as secrets validam por 24h

### Testes de Carga
- 100 webhooks configurados
- 1000 mudanças de status/minuto
- Latência P99 medida

### UAT com Cliente
- Atlas, MaxDistribuição, Nova Cargo testam em staging
- Validam latência, confiabilidade, formato de payload
- Feedback incorporado antes de produção

### Monitoração em Produção
- Dashboard de latência (P50, P95, P99)
- Dashboard de taxa de sucesso (diário)
- Alertas para: worker down, DLQ > 100 eventos, latência > 15s

---

## Referências da Transcrição

- `[09:00]` a `[09:02]`: Contexto do problema (Atlas, MaxDistribuição, Nova Cargo, polling)
- `[09:02]`: Sofia pergunta sobre outbound vs inbound (decidido: outbound)
- `[09:31]` a `[09:44]`: Requisitos funcionais detalhados
- `[09:45]` a `[09:50]`: Prazo (3 sprints, fim de novembro)

---

## Aprovação

| Papel | Nome | Aprovação | Data |
|-------|------|-----------|------|
| Product Owner | Marcos | ⏳ Pendente | - |
| Tech Lead | Larissa | ⏳ Pendente | - |
| Security Review | Sofia | ⏳ Pendente | - |
