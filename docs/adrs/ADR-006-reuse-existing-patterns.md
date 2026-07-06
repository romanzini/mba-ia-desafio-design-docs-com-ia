# ADR-006: Reutilização de Padrões e Infra Existentes do Projeto

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Arquitetura e design do módulo de webhooks  
**Participantes:** Bruno (Engenheiro Pleno), Larissa (Tech Lead), Diego (Engenheiro Sênior)

## Resumo

O módulo de webhooks segue **exatamente os padrões estabelecidos** no projeto existente: estrutura de modules, classes de erro, logger, middleware de autenticação, schemas Zod, padrão de repository-service-controller. Máxima reutilização de infra e código existente. Sem inovações arquiteturais desnecessárias.

## Problema

Ao introduzir novo módulo (webhooks), há duas abordagens:

1. **Inovação arquitetural**: criar novos padrões, novas camadas, novas convenções.
2. **Reutilização**: seguir exatamente o que já funciona no projeto.

A primeira cria:
- Inconsistência na codebase: developers precisam aprender padrões diferentes.
- Overhead de manutenção: múltiplas formas de fazer a mesma coisa.
- Onboarding complexo: novo dev não sabe qual padrão usar em qual contexto.

Bruno e Diego concordam: **não inovar, reutilizar**.

## Decisão

O módulo `webhooks` é criado em `src/modules/webhooks/` com a mesma estrutura que `src/modules/orders/`, `src/modules/products/`, etc:

```
src/modules/webhooks/
  ├── webhook.controller.ts       # endpoints HTTP
  ├── webhook.service.ts          # lógica de negócio
  ├── webhook.repository.ts       # acesso a dados
  ├── webhook.routes.ts           # definição de rotas
  ├── webhook.schemas.ts          # schemas Zod para validação
  └── webhook.processor.ts        # processamento de eventos (novo, específico do worker)
```

### Reuso Específico

#### 1. **Classe AppError e Subclasses**
- **Arquivo existente**: `src/shared/errors/app-error.ts`
- **Reuso**: erros de webhook herdam de `AppError`.
- **Codes**: prefixo `WEBHOOK_` seguindo padrão (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
- **Exemplo**:
  ```typescript
  export class WebhookNotFoundError extends AppError {
    constructor() {
      super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND');
    }
  }
  ```

#### 2. **Logger Pino**
- **Arquivo existente**: `src/shared/logger/index.ts`
- **Reuso**: worker e serviço usam o mesmo `logger` singleton.
- **Config**: já vem com redação de secrets, timestamps ISO, service name.
- **Uso no worker**:
  ```typescript
  import { logger } from '../shared/logger/index.js';
  logger.info({ event_id }, 'Webhook dispatched successfully');
  ```

#### 3. **Error Middleware Centralizado**
- **Arquivo existente**: `src/middlewares/error.middleware.ts`
- **Reuso**: endpoints de webhook usam o mesmo middleware. Sem catch específico.
- **Vantagem**: `AppError` de webhook é automaticamente transformado em JSON com code + message.

#### 4. **Middleware de Autenticação (authenticate + requireRole)**
- **Arquivo existente**: `src/middlewares/auth.middleware.ts`
- **Reuso**:
  - Endpoints de CRUD de webhook (criação, listagem) usam `authenticate` (valida JWT, carrega `req.user`).
  - Endpoint de admin (replay de DLQ) usa `authenticate` + `requireRole('ADMIN')`.
- **Customização**: webhooks herdam `customer_id` do request body (usuário autenticado cria webhook para seu customer).

#### 5. **Schemas Zod**
- **Arquivo existente**: `src/modules/users/user.schemas.ts`, `src/modules/orders/order.schemas.ts`
- **Reuso**: mesmo pattern.
- **Exemplo**:
  ```typescript
  // src/modules/webhooks/webhook.schemas.ts
  export const createWebhookSchema = z.object({
    customerId: z.string().uuid(),
    url: z.string().url().startsWith('https://'),
    events: z.array(z.enum(['PENDING', 'PAID', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED'])),
  });
  ```

#### 6. **Padrão Repository-Service-Controller**
- **Existente em**: todos os modules (OrderRepository, OrderService, OrderController).
- **Reuso**: WebhookRepository (acesso a dados), WebhookService (lógica de negócio), WebhookController (endpoints HTTP).
- **Data layer**: usa Prisma (já existe em projeto).

#### 7. **PrismaClient Existente**
- **Arquivo existente**: `src/config/database.ts`
- **Reuso**:
  - API usa `prisma` client injetado em controllers (vide `src/app.ts`).
  - Worker cria seu próprio `PrismaClient` (mesmo DATABASE_URL, instância separada por processo).
- **Transações**: webhook outbox é inserida dentro da transação de `changeStatus` (vide integration no FDD).

#### 8. **Padrão de Resposta HTTP**
- **Arquivo existente**: `src/shared/http/response.ts` (função `paginated` para listas).
- **Reuso**: endpoints de webhook listagem usam `paginated()`.

## Alternativas Consideradas

### Alternativa 1: Novos Padrões Específicos de Webhook
**Descrição**: Criar event bus, job queue, ou event sourcing específico para webhooks.

**Trade-off negativo**:
- Adiciona complexidade; dificulta manutenção.
- Developers precisam conhecer 2 formas de fazer CRUD (uma pra orders, outra pra webhooks).
- Overhead de dependencies (kafka, bull, etc.).

**Por que foi descartado**: Over-engineering. Padrão existente funciona.

### Alternativa 2: Usar Estrutura "Pluggable"
**Descrição**: Criar framework abstrato que todos os modules herdam.

**Trade-off negativo**:
- Prematura abstração; esperamos até ter 5+ modules similares.
- Overhead de aprendizado desnecessário agora.

**Por que foi descartado**: Não está no escopo atual.

## Consequências

### Positivas

✅ **Consistência**: webhook é um módulo como qualquer outro no projeto.

✅ **Onboarding**: dev novo entende estrutura em 5 min (já conhece o padrão).

✅ **Manutenção**: código é familiar, fácil de debugar.

✅ **Sem dependências novas**: nenhuma biblioteca nova (não precisa Kafka, Bull, etc.). Usa o que já existe.

✅ **Integração simples**: `changeStatus` chama `publishWebhookEvent(tx, order)` função simples.

### Negativas

⚠️ **Sem event sourcing**: não temos auditoria full de eventos históricos. Outbox é só pra delivery. (Decidido como não crítico agora)

⚠️ **Sem abstração futura**: se um dia temos múltiplos tipos de eventos (webhooks, email, SMS), não há camada unificada. (Decidido como scale problem do futuro)

## Detalhes Técnicos

### Injeção de Dependências (Pattern Existente)

No `src/app.ts`, seguir padrão:

```typescript
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository);
const webhookController = new WebhookController(webhookService);
```

### Error Codes Prefix

Todos os erros de webhook usam prefixo `WEBHOOK_`:
- `WEBHOOK_NOT_FOUND`
- `WEBHOOK_INVALID_URL`
- `WEBHOOK_SECRET_REQUIRED`
- `WEBHOOK_ALREADY_INACTIVE`
- `WEBHOOK_SECRET_ROTATED_RECENTLY`
- etc.

### Logger Context

Worker adiciona contexto ao logger:
```typescript
const log = logger.child({ module: 'webhook-worker', event_id });
log.info('Processing event');
```

## Referências

- Transcrição: `[09:27]` Bruno propondo estrutura de module em `src/modules/webhooks/`.
- Transcrição: `[09:28]` Diego concordando com pattern repository-service.
- Transcrição: `[09:29]` Bruno sobre reutilização de AppError, Pino, error middleware, padrão de código.
- Transcrição: `[09:30]` Larissa finalizando: "reuso máximo do que já existe".
- Transcrição: `[09:36]` Sofia e Larissa sobre middleware `requireRole` pra admin.
- Código: `src/app.ts` - padrão de injeção de dependências.
- Código: `src/shared/errors/app-error.ts` - classe base.
- Código: `src/shared/logger/index.ts` - logger Pino.
- Código: `src/middlewares/error.middleware.ts` - tratamento centralizado.
- Código: `src/modules/orders/order.service.ts` - exemplo de service existente.
