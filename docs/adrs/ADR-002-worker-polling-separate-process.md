# ADR-002: Worker em Polling Separado do Processo API

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Processamento assíncrono de eventos de webhook  
**Participantes:** Diego (Engenheiro Sênior), Larissa (Tech Lead), Bruno (Engenheiro Pleno)

## Resumo

O worker que lê eventos da `webhook_outbox` e dispara HTTP calls para webhooks clientes executa em **processo separado e independente** do processo API. Usa polling a cada 2 segundos para verificar novos eventos pendentes.

## Problema

O worker precisa executar continuamente, processando eventos em background. Dois caminhos possíveis:

1. **Worker embarcado**: rodar no mesmo processo Node.js da API.
2. **Worker separado**: processo Node.js dedicado com seu próprio loop de polling.

A primeira abordagem cria problemas:

- Se a API reinicia (deploy, crash), o worker é interrompido; eventos acumulam na outbox sem serem processados.
- Worker em loop dentro da API consome CPU/eventos da aplicação, competindo com requisições HTTP.
- Difícil de escalar: quer rodar worker em máquina diferente? Precisa duplicar toda a instância da API.

## Decisão

O worker é um **entry-point separado** do projeto, criado como `src/worker.ts` com seu próprio `npm run worker`. É um processo Node.js diferente:

- Conecta ao mesmo banco (mesma `DATABASE_URL`).
- Usa o mesmo Prisma Client (mas instância separada por processo).
- Executa em loop infinito de polling: a cada 2 segundos, busca eventos `status = PENDING`, processa, marca como `DELIVERED` ou move para DLQ.
- Pode rodar em container separado, máquina diferente ou mesmo host que API, conforme orquestração.

**Pseudocódigo do worker:**

```typescript
// src/worker.ts
const prisma = new PrismaClient();
const logger = createLogger();

async function processOutbox() {
  try {
    const pendingEvents = await prisma.webhookOutbox.findMany({
      where: { status: 'PENDING' },
      orderBy: { createdAt: 'asc' },
      take: 10,
    });

    for (const event of pendingEvents) {
      await dispatchWebhook(event);
    }
  } catch (error) {
    logger.error({ error }, 'Outbox processing error');
  }
}

// Loop: processa a cada 2s
setInterval(processOutbox, 2000);
```

## Alternativas Consideradas

### Alternativa 1: Worker Embarcado
**Descrição**: Job/worker roda no mesmo processo da API, em thread separada ou evento de node.

**Trade-off negativo**:
- Acoplamento: se API cai, worker cai; eventos acumulam.
- Duplicação difícil: quer colocar worker em máquina separada? Não dá, está acoplado ao código da API.
- Concorrência: worker compete com requisições HTTP por CPU/pool de conexões.

**Por que foi descartado**: Quebra isolamento de concerns e dificulta operacional.

### Alternativa 2: Trigger/NOTIFY do Banco
**Descrição**: Banco notifica worker de novo evento (via trigger + listener).

**Trade-off negativo**:
- MySQL não tem `NOTIFY/LISTEN` nativo (Postgres sim).
- Alternativa improvisada (arquivo, endpoint) fica complexa.
- Polling 2s já atende latência requerida (≤10s).

**Por que foi descartado**: Complexidade sem ganho de latência significativo.

## Consequências

### Positivas

✅ **Isolamento de falhas**: crash do worker não afeta API e vice-versa.

✅ **Escalabilidade**: worker pode rodar em container/host separado, escalado independentemente.

✅ **Observabilidade**: logs de worker são separados dos logs da API; fácil de debugar.

✅ **Deploy desacoplado**: pode fazer deploy de worker sem redeployar API.

### Negativas

⚠️ **Operacional**: precisa manter 2 processos em HA (orquestração, monitoring).

⚠️ **Latência polling**: no pior caso, evento leva até 2s para ser processado (ainda < 10s requerido).

⚠️ **Single-point-of-failure**: se worker cai e fica sem reativar, eventos acumulam. Requer monitoring/alertas.

## Questões em Aberto

1. **Health checks**: como verificar que worker está vivo? (Sugestão: expor health endpoint ou monitora heartbeat em tabela separada)
2. **Escalabilidade futura**: múltiplos workers em paralelo? (Decidido como "observar"; pode ser adicionado depois com lock pessimista ou particionamento)

## Detalhes Técnicos

**Entry-point:**
- `src/worker.ts` é novo arquivo que cria PrismaClient, logger e inicia loop.
- `package.json` terá script: `"worker": "tsx src/worker.ts"`

**Versioning:**
- Mesmo package.json, mesma versão de node_modules: worker está sempre sincronizado com API.

**Database:**
- Mesmo `DATABASE_URL`, mesma credencial (worker precisa de acesso à DB).
- PrismaClient separado por processo (não compartilhado com API).

## Referências

- Transcrição: `[09:08]` Diego sobre polling vs triggers.
- Transcrição: `[09:11]` Larissa e Diego sobre processo separado.
- Transcrição: `[09:11]` Bruno sugerindo `src/worker.ts` como entry-point.
- Código: `src/server.ts` - padrão de entry-point existente (será replicado para worker).
- Código: `src/config/database.ts` - PrismaClient que worker reutilizará.
