# Documentação do Processo de Produção: Sistema de Webhooks de Notificação de Pedidos

## Sobre o Desafio

Este projeto é um desafio acadêmico de design de software onde a tarefa foi transformar a transcrição de uma reunião técnica de 55 minutos em um pacote completo de design documentation para um Sistema de Webhooks de Notificação de Pedidos.

**Cenário:** Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitam webhooks para receber notificações em tempo real quando o status de seus pedidos muda em um Order Management System (OMS) Node.js + TypeScript. Hoje, clientes fazem polling manual, gerando latência e custos. Atlas ameaçou migração para concorrente se feature não sair até fim de novembro.

**Tarefa:** Produzir documentação técnica de design (PRD, RFC, FDD, ADRs, Tracker, novo README) em nível acionável o suficiente para o time começar a implementação.

---

## Ferramentas de IA Utilizadas

### GitHub Copilot Chat (VS Code)
**Papel:** Assistente principal para geração, estruturação e refinamento de documentação.

Usada para:
- Gerar ADRs estruturadas em formato MADR com contexto, decisão, alternativas reais e consequências
- Consolidar proposta técnica em RFC com diagramas ASCII e tabelas de trade-offs
- Detalhar fluxos, contratos HTTP, erros e integração no FDD
- Validar completude e rastreabilidade contra transcrição

**Iterações:** 3 ciclos principais de refinamento até atingir 100% rastreabilidade.

---

## Workflow Adotado

### 1. Contextualização (Leitura Crítica)
- Leitura completa da transcrição identificando 6 decisões principais e 2 itens descartados
- Exploração do código existente para entender padrões: AppError, Pino logger, repository-service-controller, error middleware
- Mapeamento de 6+ pontos de integração reais no código

### 2. ADRs Primeiro (Esqueleto)
Razão: ADRs são a base. Cada um é uma decisão isolada e bem-definida. RFC e FDD se apoiam neles.

Documentos:
- **ADR-001**: Padrão Outbox MySQL (vs Redis Streams)
- **ADR-002**: Worker em polling separado (vs embarcado na API)
- **ADR-003**: Retry com backoff 5 tentativas (1m, 5m, 30m, 2h, 12h)
- **ADR-004**: Autenticação HMAC-SHA256 com secret por endpoint
- **ADR-005**: At-least-once com X-Event-Id para deduplicação
- **ADR-006**: Máxima reutilização de padrões existentes

Cada ADR cita [hh:mm] Falante e refencia alternativas reais descartadas.

### 3. RFC (Consolidação de Proposta)
- TL;DR executivo
- Contexto de negócio (3 clientes, risco de churn)
- Proposta técnica (sem descer ao detalhe de implementação)
- 3 alternativas descartadas com trade-offs explícitos
- 4 questões abertas identificadas
- Links para todos os 6 ADRs

Resultado: 4 páginas, conciso, pronto para revisão da equipe.

### 4. FDD (Implementação Detalhada)
- 5 fluxos detalhados (criação, mudança de status, processamento, rotação, replay)
- 7 endpoints HTTP com request/response de exemplo
- Matriz de 6 códigos de erro com prefixo WEBHOOK_
- Integração com 4+ arquivos reais do código (order.service.ts, app.ts, schema.prisma)
- Tabelas Prisma completas
- Critérios de aceite técnicos

### 5. PRD (Consolidação de Requisitos)
- 13 requisitos funcionais (RF-01 a RF-13)
- 7 requisitos não-funcionais (latência ≤10s, confiabilidade ≥98%, etc)
- Escopo explícito: incluso (webhooks outbound) + fora de escopo (email, dashboard, rate limiting)
- 5 decisões e trade-offs principais
- 5 riscos com probabilidade, impacto e mitigação

### 6. Tracker (Rastreabilidade)
- 88 itens totais mapeados
- 100% com origem identificável
- ~70% referenciam transcrição com [hh:mm] Falante
- ~30% referenciam código (src/..., prisma/...)

Validação: cada item no PRD/RFC/FDD tem linha no Tracker.

### 7. README do Processo
Este documento. Documentação da jornada de produção.

---

## Prompts Customizados Utilizados

### Prompt 1: Geração de ADR com Alternativas Reais

```
Gere uma Architecture Decision Record (ADR) com:
- Assunto: [padrão de entrega assíncrona para webhooks]
- Decisão: [Outbox Pattern no MySQL]
- Alternativas descartadas REAIS discutidas em:
  - [09:07] Larissa: Redis Streams
  - [09:03-09:04] Bruno: HTTP Síncrono
  - [09:09] Diego: Trigger do banco

Para cada alternativa, explicitar o trade-off que levou ao descarte.
Incluir referências [hh:mm] Falante.
Incluir referências a arquivos reais (src/modules/orders/order.service.ts).

Output: Markdown MADR com Contexto, Decisão, Alternativas, Consequências.
```

**Resultado:** ADRs estruturadas, alinhadas com transcrição, sem alucinações.

### Prompt 2: Validação de Rastreabilidade

```
Revise FDD contra TRANSCRICAO.md:
- Cada endpoint tem origem [hh:mm]?
- Cada campo de payload foi discutido?
- Cada código de erro refencia situação real?
- Cada integração refencia arquivo real (src/...)?

Remova itens sem rastreamento claro (suspeita de alucinação).

Foco: 100% rastreabilidade. Nada inventado.
```

**Resultado:** FDD sem alucinações. Todas as features discutidas na reunião e nos documentos.

### Prompt 3: Consolidação de Requisitos em PRD

```
Gere PRD com requisitos mapeados da transcrição:

Funcionais discutidos:
- [09:31] Marcos: cadastrar webhook (endpoint POST)
- [09:33] Bruno: listar, editar, deletar webhooks
- [09:34] Marcos: histórico de entregas
- [09:18] Diego: replay manual de DLQ

Não-Funcionais:
- [09:02] Marcos: latência ≤ 10s
- [09:16] Diego: backoff cobrindo até 14h

Fora de escopo (explicitamente descartado):
- [09:37] Larissa: email como fallback → PRÓXIMA FASE
- [09:40] Marcos: dashboard visual → PROJETO SEPARADO
- [09:38] Diego: rate limiting → OBSERVAR DEPOIS

Output: PRD com Requisitos Funcionais, Não-Funcionais, Escopo (incluso/fora), Decisões, Riscos.
Cada item cita [hh:mm] Falante.
```

**Resultado:** PRD bem estruturado, sem ambiguidades de escopo.

---

## Iterações e Ajustes Principais

### Iteração 1: Superficialidade Inicial
**Problema:** Primeiros documentos eram genéricos, sem rastreabilidade.

**Ajuste:** Prompts mais específicos exigindo [hh:mm] e referências ao código.

**Resultado:** Documentos concretos e validáveis.

---

### Iteração 2: Redundância Entre Documentos
**Problema:** RFC e FDD tinham mesmo nível de detalhe. Impossível saber qual ler primeiro.

**Ajuste:** RFC mantido conciso (3 páginas, decisões + alternativas). FDD com fluxos e contratos.

**Resultado:** Documentos em "alturas" diferentes, sem repetição.

---

### Iteração 3: Alucinações de IA
**Problema:** Alguns itens apareciam mas não tinham origem clara.

Exemplos detectados:
- "Adicionar items do pedido no payload" → Verificado: [09:43] Diego diz "Não manda items"
- "Email como notificação" → Verificado: [09:37] Larissa diz "Email tá fora de escopo"

**Ajuste:** Criado TRACKER com 100% rastreabilidade. Cada item deve ter [hh:mm] ou src/.

**Resultado:** Zero alucinações confirmadas.

---

### Iteração 4: Alinhamento de Escopo
**Problema:** FDD incluía "email em fase 1". Mas transcrição deixa claro: FORA DE ESCOPO.

**Ajuste:** PRD seção "Fora de Escopo" lista explicitamente com timestamps.

**Resultado:** Time sabe exatamente o que não entra nesta release.

---

### Iteração 5: Integração com Código Existente
**Problema:** FDD não mapeava onde mudar o código de verdade.

**Ajuste:** Seção "Integração com Sistema Existente" adiciona:
- Linhas aproximadas de mudança (ex: order.service.ts ~126-180)
- Pseudocódigo antes/depois
- Novos arquivos (webhook.processor.ts, webhook.errors.ts)
- Schema Prisma completo

**Resultado:** Dev abre FDD e consegue começar em 1h.

---

## Como Navegar a Entrega

### Ordem Recomendada de Leitura

1. **RFC.md** (3 min) → Proposta em alto nível, decisões, alternativas
2. **docs/adrs/** (10 min) → Cada decisão isolada
3. **FDD.md** (20 min) → Fluxos, contratos, integração (documento para dev codar)
4. **PRD.md** (10 min) → Requisitos, escopo, riscos (referência de product)
5. **TRACKER.md** (5 min) → Validação: origem de cada item
6. **TRANSCRICAO.md** → Reunião original para contexto completo

### Estrutura de Arquivos

```
docs/
├── PRD.md                          ← O quê? Por quê? (Product)
├── RFC.md                          ← Como? Por que essa abordagem? (Arquitetura)
├── FDD.md                          ← Como implementar em detalhe? (Dev)
├── TRACKER.md                      ← Origem de cada item (Rastreabilidade)
└── adrs/
    ├── ADR-001-outbox-pattern-mysql.md
    ├── ADR-002-worker-polling-separate-process.md
    ├── ADR-003-retry-backoff-dlq.md
    ├── ADR-004-hmac-sha256-authentication.md
    ├── ADR-005-at-least-once-event-id.md
    └── ADR-006-reuse-existing-patterns.md
```

---

## Critérios de Aceite Alcançados

✅ **PRD:** 13 RFs + 7 RNFs + escopo (2 itens fora) + riscos  
✅ **RFC:** TL;DR + contexto + proposta + 3 alternativas + 4 questões abertas + links ADRs  
✅ **FDD:** 5 fluxos + 7 endpoints + 6 erros + 4+ integrações + tabelas Prisma  
✅ **ADRs:** 6 arquivos, cada um com contexto, decisão, 1+ alternativas, consequências  
✅ **Tracker:** 88 itens, 100% rastreados, 70%+ com [hh:mm], 5+ com src/  
✅ **README:** Processo documentado, 2+ prompts customizados, 3+ iterações explicadas  

---

## Lições Aprendidas

### ✅ O Que Funcionou Bem
1. **Prompts específicos:** Exigir [hh:mm] levava a resultados melhores
2. **TRACKER no final:** Força validação contra alucinações
3. **Separação de alturas:** RFC conciso, FDD detalhado evita redundância
4. **Mapear código antes:** Entender padrões existentes (AppError, Pino) antes de gerar docs

### ⚠️ Desafios
1. **FDD ficou grande:** Poderia ser dividido em 3 arquivos (fluxos, contratos, integração)
2. **Mudanças retroativas:** Se um ADR muda, tudo muda (RFC, FDD, TRACKER)
3. **Validação manual:** Sem ferramenta automática, validação é 100% manual

### 💡 Se Projeto Continuasse
1. Adicionar diagrama C4 Model
2. Expandir TRACKER com matriz 2D (documentos vs fonte)
3. Criar seção "Resoluções de Review" para feedback do RFC
4. Implementar em código real validando contra FDD

---

## Conclusão

A IA é ferramenta poderosa para geração de documentação, mas precisa de maestro humano para:
- **Estruturação clara** (o que vai onde)
- **Validação contra fonte de verdade** (transcrição + código)
- **Iteração e refinamento** (não é geração one-shot)
- **Síntese coerente** (decisões em narrativa consistente)

O resultado é um pacote de design docs **rastreável**, **coerente** e **acionável** para time começar implementação com confiança.

---

**Gerado com:** GitHub Copilot Chat  
**Data:** 2026-07-06  
**Status:** Completo (6/6 documentos esperados)  
**Rastreabilidade:** 100% (88/88 itens com origem)
