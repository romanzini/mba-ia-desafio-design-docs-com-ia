# FDD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0  
**Data:** 2026-07-06  
**Autores:** Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)  
**Revisor de Segurança:** Sofia (Engenheira de Segurança)

---

## Objetivo

Especificar em detalhe a implementação do Sistema de Webhooks de Notificação de Pedidos: fluxos, contratos públicos, erros, integração com código existente, observabilidade e critérios de aceite técnicos. Este documento é acionável o suficiente para um desenvolvedor começar a codar.

---

## Fluxos Detalhados

### Fluxo 1: Criação de Webhook

```
POST /webhooks
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://client.example.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}

Passos:
1. Validar JWT (authenticate middleware)
2. Validar schema Zod (URL https, customerId UUID, events não vazio)
3. Verificar se cliente existe → erro: WEBHOOK_CUSTOMER_NOT_FOUND
4. Gerar secret aleatório (32 bytes, hex)
5. Inserir webhook (id, customerId, url, secretHash, events, active=true)
6. Responder 201 Created com: id, url, events, secret (retorna uma vez!), createdAt
```

### Fluxo 2: Mudança de Status Gerando Evento

```
Quando OrderService.changeStatus é chamado:

1. Dentro de prisma.$transaction:
   - Validar transição de status
   - Debitar/repor estoque conforme novo status
   - Atualizar order.status
   - Inserir order_status_history
   
   ✨ NOVO:
   - Chamar publishWebhookEvent(tx, order, fromStatus, toStatus)
     Para cada webhook do customer:
     - Se novo status está em webhook.events:
       - Gerar event_id (UUID)
       - Render payload (snapshot)
       - INSERT webhook_outbox (pendente para processamento)

2. Se qualquer insert falha, toda transação faz rollback
   (consistência garantida: se status mudou, evento existe)
```

### Fluxo 3: Processamento pelo Worker

```
Worker (src/worker.ts) - Loop infinito:

A cada 2 segundos:
1. SELECT * FROM webhook_outbox 
   WHERE status = 'PENDING' 
   ORDER BY createdAt ASC 
   LIMIT 10

2. Para cada evento:
   a. Buscar webhook config (url, secretHash)
   b. Parse eventPayload (JSON)
   c. Gerar HMAC-SHA256(payload, secret)
   d. Montar HTTP POST com headers:
      - X-Signature: sha256=<hex>
      - X-Event-Id: <event_id>
      - X-Timestamp: <timestamp>
      - X-Webhook-Id: <webhook.id>
   e. Executar com timeout 10s
   
3. Analisar resposta:
   IF 2xx:
     UPDATE status='DELIVERED', processedAt=now()
   ELSE IF retryCount < 5:
     Calcular backoff [60s, 5m, 30m, 2h, 12h]
     UPDATE status='PENDING', retryCount++, nextRetryAt=now()+backoff
   ELSE:
     INSERT webhook_dead_letter (payload, failureReason, lastAttemptAt)
     DELETE FROM webhook_outbox

4. Log resultado (Pino logger)

5. Dormir 2 segundos, ir para passo 1
```

### Fluxo 4: Rotação de Secret

```
POST /webhooks/:id/rotate-secret
Authorization: Bearer <jwt_token>

1. Validar JWT e ownership (webhook.customerId == req.user.customerId)
2. Gerar novo secret
3. UPDATE webhook:
   - secretHash = hash(newSecret)
   - secretHashRotated = hash(oldSecret)
   - rotatedAt = now() + 24h
4. Responder 200 OK com novo secret (retorna uma vez!)

Durante 24h grace period:
- Worker valida assinatura com AMBOS os secrets
- Antigo expira após 24h
```

### Fluxo 5: Replay Manual de DLQ

```
POST /admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <jwt_token_admin>
(requer role ADMIN)

1. Buscar DLQ entry → erro: WEBHOOK_DLQ_NOT_FOUND
2. INSERT webhook_outbox (recoloca como PENDING):
   - eventId, webhookId, orderId, eventPayload
   - status='PENDING', retryCount=0
3. UPDATE webhook_dlq:
   - replayedAt=now()
4. Log de auditoria (userId, action, dlqId, timestamp)
5. Responder 200 OK
```

---

---

## Contratos Públicos (Endpoints HTTP)

### Endpoint 1: Criar Webhook (POST /webhooks)

```
POST /webhooks
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://client.example.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}

Response 201 Created:
{
  "webhook": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "customerId": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://client.example.com/webhooks/orders",
    "secret": "abc123def456ghi789jkl012mno345pqr678stu",
    "events": ["SHIPPED", "DELIVERED"],
    "active": true,
    "createdAt": "2026-07-06T14:30:00.000Z"
  }
}

Erros Possíveis:
- 400: VALIDATION_ERROR (Zod validation failed)
- 400: WEBHOOK_INVALID_URL (não https://)
- 404: WEBHOOK_CUSTOMER_NOT_FOUND (customer não existe)
- 401: Unauthorized (JWT inválido)
```

### Endpoint 2: Listar Webhooks (GET /webhooks)

```
GET /webhooks?customerId=550e8400-e29b-41d4-a716-446655440000&page=1&pageSize=10
Authorization: Bearer <jwt_token>

Response 200 OK:
{
  "webhooks": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "customerId": "550e8400-e29b-41d4-a716-446655440000",
      "url": "https://client.example.com/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-06T14:30:00.000Z",
      "updatedAt": "2026-07-06T14:30:00.000Z"
    }
  ],
  "pagination": {
    "total": 1,
    "page": 1,
    "pageSize": 10,
    "totalPages": 1
  }
}
```

### Endpoint 3: Atualizar Webhook (PATCH /webhooks/:id)

```
PATCH /webhooks/550e8400-e29b-41d4-a716-446655440001
Authorization: Bearer <jwt_token>
Content-Type: application/json

Request Body:
{
  "url": "https://new.endpoint.com/webhooks",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true
}

Response 200 OK:
{
  "webhook": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "url": "https://new.endpoint.com/webhooks",
    "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
    "active": true,
    "updatedAt": "2026-07-06T15:00:00.000Z"
  }
}

Erros Possíveis:
- 404: WEBHOOK_NOT_FOUND
- 403: WEBHOOK_FORBIDDEN (não é owner)
- 400: WEBHOOK_INVALID_URL (http://)
```

### Endpoint 4: Deletar Webhook (DELETE /webhooks/:id)

```
DELETE /webhooks/550e8400-e29b-41d4-a716-446655440001
Authorization: Bearer <jwt_token>

Response 204 No Content
(sem body)

Erros Possíveis:
- 404: WEBHOOK_NOT_FOUND
- 403: WEBHOOK_FORBIDDEN
```

### Endpoint 5: Histórico de Entregas (GET /webhooks/:id/deliveries)

```
GET /webhooks/550e8400-e29b-41d4-a716-446655440001/deliveries?page=1&pageSize=10
Authorization: Bearer <jwt_token>

Response 200 OK:
{
  "deliveries": [
    {
      "eventId": "550e8400-e29b-41d4-a716-446655440002",
      "orderId": "550e8400-e29b-41d4-a716-446655440003",
      "status": "DELIVERED",
      "responseStatus": 200,
      "payload": {
        "event_id": "550e8400-e29b-41d4-a716-446655440002",
        "event_type": "order.status_changed",
        "timestamp": "2026-07-06T14:30:00.000Z",
        "order": {
          "id": "550e8400-e29b-41d4-a716-446655440003",
          "orderNumber": "ORD-001234",
          "customerId": "550e8400-e29b-41d4-a716-446655440000",
          "from_status": "PENDING",
          "to_status": "PAID",
          "total_cents": 150000
        }
      },
      "response": "OK",
      "duration_ms": 250,
      "attemptNumber": 1,
      "sentAt": "2026-07-06T14:30:05.000Z"
    }
  ],
  "pagination": {
    "total": 42,
    "page": 1,
    "pageSize": 10,
    "totalPages": 5
  }
}
```

### Endpoint 6: Rotação de Secret (POST /webhooks/:id/rotate-secret)

```
POST /webhooks/550e8400-e29b-41d4-a716-446655440001/rotate-secret
Authorization: Bearer <jwt_token>

Response 200 OK:
{
  "webhook": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "url": "https://client.example.com/webhooks/orders",
    "secret": "xyz789abc123def456ghi789jkl012mno345pqr",
    "rotatedAt": "2026-07-06T16:00:00.000Z",
    "graceUntil": "2026-07-07T16:00:00.000Z"
  }
}

Erros Possíveis:
- 404: WEBHOOK_NOT_FOUND
- 403: WEBHOOK_FORBIDDEN
- 429: WEBHOOK_SECRET_ROTATED_RECENTLY
```

### Endpoint 7: Replay de DLQ (POST /admin/webhooks/dead-letter/:id/replay)

```
POST /admin/webhooks/dead-letter/550e8400-e29b-41d4-a716-446655440099/replay
Authorization: Bearer <jwt_token_admin>
(requer role ADMIN)

Response 200 OK:
{
  "message": "Event moved back to outbox for reprocessing",
  "dlqId": "550e8400-e29b-41d4-a716-446655440099",
  "replayedAt": "2026-07-06T16:30:00.000Z"
}

Erros Possíveis:
- 404: WEBHOOK_DLQ_NOT_FOUND
- 403: FORBIDDEN (não é ADMIN)
```

---

## Matriz de Erros

| Código | Status HTTP | Mensagem | Contexto |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook not found | GET/PATCH/DELETE de webhook inexistente |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Customer not found | Cliente não existe |
| `WEBHOOK_INVALID_URL` | 400 | Webhook URL must use HTTPS protocol | URL não é https:// |
| `WEBHOOK_FORBIDDEN` | 403 | Not allowed to modify this webhook | Usuário não é owner |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Dead letter entry not found | DLQ inexistente |
| `VALIDATION_ERROR` | 400 | Validation failed | Erro Zod (details array) |

---

## Integração com o Sistema Existente

### 1. Em `src/modules/orders/order.service.ts`

**Localização**: Método `changeStatus` (linhas ~126-180)

**Antes** (pseudocódigo):
```typescript
async changeStatus(id: string, input: UpdateOrderStatusInput, userId: string) {
  return this.prisma.$transaction(async (tx) => {
    const order = await tx.order.findUnique({ where: { id } });
    if (!order) throw new NotFoundError('Order');
    
    const from = order.status;
    const to = input.toStatus;
    // validações...
    
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({ data: {...} });
    
    const refreshed = await tx.order.findUnique({ where: { id }, include: {...} });
    return refreshed!;
  });
}
```

**Depois**:
```typescript
import { publishWebhookEvent } from '../webhooks/webhook.processor.js';

async changeStatus(id: string, input: UpdateOrderStatusInput, userId: string) {
  return this.prisma.$transaction(async (tx) => {
    const order = await tx.order.findUnique({ where: { id } });
    if (!order) throw new NotFoundError('Order');
    
    const from = order.status;
    const to = input.toStatus;
    // validações...
    
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({ data: {...} });
    
    // ✨ NOVA LINHA: Publicar evento webhook
    await publishWebhookEvent(tx, order, from, to);
    
    const refreshed = await tx.order.findUnique({ where: { id }, include: {...} });
    return refreshed!;
  });
}
```

### 2. Em `src/app.ts`

**Localização**: Função `buildControllers`

**Mudança**: Adicionar injeção de WebhookController

```typescript
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository);
const webhookController = new WebhookController(webhookService);

// ... resto do código
```

### 3. Em `src/routes/index.ts`

**Mudança**: Registrar rotas de webhooks

```typescript
router.use('/webhooks', controllers.webhooks.router);
router.use('/admin/webhooks', controllers.webhooks.adminRouter);
```

### 4. Novas Tabelas em `prisma/schema.prisma`

```prisma
model Webhook {
  id              String   @id @default(uuid()) @db.Char(36)
  customerId      String   @db.Char(36)
  url             String   @db.VarChar(2048)
  secretHash      String   @db.VarChar(255)
  secretHashRotated String? @db.VarChar(255)
  events          Json     // ["SHIPPED", "DELIVERED"] etc
  active          Boolean  @default(true)
  rotatedAt       DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  customer        Customer               @relation(fields: [customerId], references: [id])
  outbox          WebhookOutbox[]
  deliveries      WebhookDelivery[]
  deadLetters     WebhookDeadLetter[]

  @@index([customerId])
  @@index([active])
  @@map("webhooks")
}

model WebhookOutbox {
  id              String   @id @default(uuid()) @db.Char(36)
  eventId         String   @unique @db.Char(36)
  webhookId       String   @db.Char(36)
  orderId         String   @db.Char(36)
  eventPayload    Json
  status          String   @default("PENDING")
  failureReason   String?  @db.VarChar(1024)
  retryCount      Int      @default(0)
  maxRetries      Int      @default(5)
  nextRetryAt     DateTime?
  processedAt     DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  webhook         Webhook  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  order           Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)

  @@index([status, createdAt])
  @@index([webhookId])
  @@index([orderId])
  @@index([nextRetryAt])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id              String   @id @default(uuid()) @db.Char(36)
  webhookOutboxId String?  @db.Char(36)
  webhookId       String   @db.Char(36)
  orderId         String   @db.Char(36)
  eventPayload    Json
  failureReason   String   @db.VarChar(1024)
  lastAttemptAt   DateTime
  replayedAt      DateTime?
  createdAt       DateTime @default(now())

  webhook         Webhook  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  order           Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)

  @@index([webhookId])
  @@index([orderId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id              String   @id @default(uuid()) @db.Char(36)
  webhookId       String   @db.Char(36)
  orderId         String   @db.Char(36)
  eventId         String   @db.Char(36)
  payload         Json
  responseStatus  Int?
  responseBody    String?  @db.LongText
  durationMs      Int
  attemptNumber   Int
  success         Boolean
  error           String?  @db.VarChar(1024)
  sentAt          DateTime

  webhook         Webhook  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  order           Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)

  @@index([webhookId])
  @@index([orderId])
  @@index([sentAt])
  @@map("webhook_deliveries")
}
```

### 5. Classe AppError em `src/shared/errors/webhook-errors.ts`

```typescript
import { AppError } from './app-error.js';

export class WebhookNotFoundError extends AppError {
  constructor() {
    super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends AppError {
  constructor() {
    super('Webhook URL must use HTTPS protocol', 400, 'WEBHOOK_INVALID_URL');
  }
}

export class WebhookCustomerNotFoundError extends AppError {
  constructor() {
    super('Customer not found', 404, 'WEBHOOK_CUSTOMER_NOT_FOUND');
  }
}

export class WebhookForbiddenError extends AppError {
  constructor() {
    super('Not allowed to modify this webhook', 403, 'WEBHOOK_FORBIDDEN');
  }
}

export class WebhookDLQNotFoundError extends AppError {
  constructor() {
    super('Dead letter entry not found', 404, 'WEBHOOK_DLQ_NOT_FOUND');
  }
}
```

---

## Observabilidade

### Logs (Pino)

**Em webhook.service.ts:**
```typescript
private logger = logger.child({ module: 'webhook-service' });

this.logger.info({
  action: 'webhook_created',
  webhookId: webhook.id,
  customerId: webhook.customerId,
}, 'Webhook created successfully');
```

**Em worker (src/worker.ts):**
```typescript
const workerLogger = logger.child({ module: 'webhook-worker' });

workerLogger.info({
  eventId: event.eventId,
  webhookId: event.webhookId,
  url: webhook.url,
}, 'Dispatching webhook');

if (response.ok) {
  workerLogger.info({
    eventId: event.eventId,
    status: response.status,
    durationMs: elapsed,
  }, 'Webhook delivered successfully');
} else {
  workerLogger.warn({
    eventId: event.eventId,
    status: response.status,
    retryCount,
  }, 'Webhook delivery failed, will retry');
}
```

### Métricas

- `webhook.created` - Contador
- `webhook.delivered` - Contador
- `webhook.failed` - Contador
- `webhook.retry` - Contador
- `webhook.dlq` - Contador
- `webhook.dispatch.duration_ms` - Histograma

### Tracing Distribuído

Implementação de tracing para rastreamento de requisições end-to-end:

**Padrão de Propagação:**
```typescript
// Endpoint POST /webhooks (criar webhook)
export const authenticate: RequestHandler = (req, _res, next) => {
  const traceId = req.headers['x-trace-id'] ?? generateUUID();
  const spanId = generateUUID();
  
  req.id = traceId; // Propagado para todo o request
  
  const childLogger = logger.child({ 
    traceId, 
    spanId,
    module: 'webhook-api' 
  });
  
  childLogger.info({
    action: 'webhook_api_request',
    method: req.method,
    path: req.path,
  }, 'API request started');
  
  next();
};

// Em webhook.service.ts (criar webhook)
private logger = logger.child({ module: 'webhook-service' });

async create(input: CreateWebhookInput, userId: string): Promise<Webhook> {
  const spanId = generateUUID();
  const childLogger = this.logger.child({ spanId });
  
  childLogger.info({
    webhookId: webhook.id,
    customerId: webhook.customerId,
    action: 'webhook_creation_started',
  }, 'Creating webhook');
  
  // ... criação do webhook
  
  childLogger.info({
    webhookId: webhook.id,
    durationMs: elapsed,
    action: 'webhook_creation_completed',
  }, 'Webhook created successfully');
}

// Em order.service.ts (changeStatus → publishWebhookEvent)
async changeStatus(id: string, input: UpdateOrderStatusInput, userId: string): Promise<OrderWithRelations> {
  const traceId = req.id;
  const spanId = generateUUID();
  const childLogger = logger.child({ traceId, spanId, module: 'order-service' });
  
  return this.prisma.$transaction(async (tx) => {
    childLogger.info({
      orderId: id,
      action: 'status_change_started',
      fromStatus: order.status,
      toStatus: input.toStatus,
    }, 'Order status change initiated');
    
    // ... mudança de status
    
    // Chamada para publicar eventos webhook
    await publishWebhookEvent(tx, order, fromStatus, toStatus);
    
    childLogger.info({
      orderId: id,
      action: 'webhook_events_published',
      eventCount: events.length,
    }, 'Webhook events published');
  });
}

// Em worker (src/worker.ts)
const workerLogger = logger.child({ module: 'webhook-worker' });

const event = // fetch from outbox
const traceId = event.traceId ?? generateUUID();
const spanId = generateUUID();
const childLogger = workerLogger.child({ traceId, spanId });

childLogger.info({
  eventId: event.eventId,
  webhookId: event.webhookId,
  action: 'webhook_dispatch_started',
  retryCount: event.retryCount,
}, 'Dispatching webhook');

try {
  const startTime = Date.now();
  const response = await fetch(webhook.url, { /* ... */ });
  const durationMs = Date.now() - startTime;
  
  if (response.ok) {
    childLogger.info({
      eventId: event.eventId,
      status: response.status,
      durationMs,
      action: 'webhook_dispatch_succeeded',
    }, 'Webhook delivered successfully');
  } else {
    childLogger.warn({
      eventId: event.eventId,
      status: response.status,
      durationMs,
      retryCount: event.retryCount,
      action: 'webhook_dispatch_failed_will_retry',
    }, 'Webhook delivery failed, will retry');
  }
} catch (error) {
  childLogger.error({
    eventId: event.eventId,
    error: error.message,
    action: 'webhook_dispatch_error',
  }, 'Webhook dispatch error');
}
```

**Headers de Propagação:**
- `X-Trace-Id`: ID único da transação end-to-end
- `X-Span-Id`: ID do segmento atual da transação
- `X-Parent-Span-Id`: ID do segmento pai

**Campos Obrigatórios em Cada Log:**
- `traceId`: Para correlação global
- `spanId`: Para correlação local (função/operação)
- `module`: Origem do log
- `action`: Nome descritivo da ação
- `timestamp`: ISO 8601 (automático no Pino)

---

## Critérios de Aceite Técnicos

- [ ] Tabelas criadas com migrations Prisma
- [ ] POST /webhooks cria webhook com secret gerado
- [ ] GET /webhooks?customerId=X lista webhooks
- [ ] PATCH /webhooks/:id atualiza
- [ ] DELETE /webhooks/:id inativa
- [ ] publishWebhookEvent chamada dentro de changeStatus
- [ ] Worker processa eventos a cada 2s
- [ ] Retry com backoff (1m, 5m, 30m, 2h, 12h)
- [ ] 5 falhas → DLQ
- [ ] HMAC-SHA256 enviado em X-Signature
- [ ] X-Event-Id gerado e enviado
- [ ] Rotação de secret com 24h grace period
- [ ] Replay de DLQ funciona
- [ ] Logs com Pino (estruturados, contexto)
- [ ] Tracing distribuído com X-Trace-Id e X-Span-Id
- [ ] Propagação de trace em chamadas síncronas e assíncronas
- [ ] Todas as classes AppError implementadas
