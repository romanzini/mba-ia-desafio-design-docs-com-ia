# ADR-004: Autenticação de Webhook via HMAC-SHA256

**Status:** Decidido  
**Data:** 2026-07-06  
**Contexto:** Segurança de webhooks outbound para clientes externos  
**Participantes:** Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior), Larissa (Tech Lead)

## Resumo

Cada webhook é autenticado via HMAC-SHA256. O servidor gera uma `secret` única por endpoint de webhook. O worker assina o body da requisição HTTP com essa secret e envia a assinatura no header `X-Signature`. O cliente verifica a assinatura do seu lado para validar autenticidade da requisição.

Secrets são rotacionáveis: quando gerada uma nova secret, a anterior fica válida por 24h em paralelo, permitindo ao cliente migrar seus sistemas gradualmente.

## Problema

Webhooks são chamadas HTTP saindo do nosso sistema para endpoints de clientes externos (URLs HTTP/HTTPS cadastradas). Sem autenticação:

1. **Falsificação**: cliente não consegue validar se a requisição realmente veio de nós.
2. **Adulteração**: se payload passa por rede não criptografada, pode ser interceptado e modificado.
3. **Replay attacks**: cliente sem forma de detectar se está recebendo o mesmo evento duas vezes (ou de um atacante repetindo requisições antigas).

Sofia (Segurança) reforça: em B2B, cliente exige validação de authenticity de cada webhook.

## Decisão

**Autenticação via HMAC-SHA256:**

1. **Secret por Endpoint**: cada webhook cadastrado tem sua própria `secret` aleatória, gerada no servidor (não é escolhida pelo cliente).
   - Armazenada em `webhooks.secret` (encrypted no banco, se possível).
   - Retornada **uma única vez** ao cliente após criação. Se cliente perde, precisa pedir nova secret.

2. **Assinatura do Payload**:
   ```
   signature = HMAC-SHA256(payload_bytes, secret)
   ```
   - `payload_bytes`: body raw da requisição HTTP (é o JSON serializado).
   - `secret`: a secret do webhook em questão.
   - Resultado: string hexadecimal.

3. **Header X-Signature**:
   - Worker envia requisição HTTP com header: `X-Signature: sha256=<hex_signature>`
   - Cliente valida no seu lado usando a mesma secret.

4. **Rotação de Secret com Grace Period**:
   - Cliente pode solicitar nova secret via `POST /webhooks/:id/rotate-secret`.
   - API gera nova secret, marca a atual como "rotated".
   - Durante **24 horas**, ambas (atual e rotated) são válidas para assinatura.
   - Após 24h, secret anterior expira; cliente DEVE estar usando a nova.
   - Isso permite ao cliente atualizar seus integradores sem perder webhooks.

5. **TLS Obrigatório**:
   - Webhook URL deve ser `https://`, não `http://`.
   - Validação no schema Zod: rejeita cadastro de `http://` com erro de validação.

6. **Headers Adicionais para Segurança**:
   - `X-Event-Id`: UUID único do evento (para deduplicação; vide ADR-005).
   - `X-Timestamp`: ISO 8601 timestamp do envio (cliente pode detectar replay attack histórico).
   - `X-Webhook-Id`: UUID do webhook configurado (cliente que tem múltiplos webhooks cadastrados consegue saber qual foi disparado).

## Alternativas Consideradas

### Alternativa 1: API Key Única do Cliente
**Descrição**: Um único API key por cliente que assina todos os webhooks.

**Trade-off negativo**:
- Se key vaza, vaza para todos os webhooks daquele cliente.
- Ataca superfície de risco; um client compromised afeta tudo do cliente.
- Não permite isolamento por endpoint.

**Por que foi descartado**: Sofia enfatiza: "secret por endpoint, não global". Uma vazada não deve comprometer todos os webhooks.

### Alternativa 2: OAuth 2.0 / JWT
**Descrição**: Usar OAuth para que cliente se autentica antes de receber webhook.

**Trade-off negativo**:
- Overhead: cliente teria que fazer handshake OAuth a cada webhook. Complexo demais.
- HMAC é padrão de mercado para webhooks (Stripe, GitHub, Twilio usam).
- Não encaixa no fluxo; webhook é push, não pull.

**Por que foi descartado**: Complexidade injustificada para use case.

### Alternativa 3: Sem Autenticação (Confiança em HTTPS)
**Descrição**: Confiar que HTTPS garante autenticidade, sem assinar payload.

**Trade-off negativo**:
- HTTPS protege em trânsito, mas não protege se servidor foi comprometido ou se payload foi logged/capturado.
- Cliente não tem forma de auditar se webhook veio realmente do OMS ou foi forjado.
- Padrão de indústria é assinar + HTTPS (defesa em profundidade).

**Por que foi descartado**: Sofia veta; requisito de segurança B2B exige assinatura.

## Consequências

### Positivas

✅ **Autenticidade garantida**: cliente valida que webhook é realmente do OMS.

✅ **Isolamento por endpoint**: comprometimento de um webhook não afeta outros do mesmo cliente.

✅ **Conformidade com padrão de mercado**: Stripe, GitHub, Twilio usam HMAC-SHA256; cliente já tem infra pra isso.

✅ **Rotação sem downtime**: grace period de 24h permite ao cliente rotar chaves sem perder eventos.

✅ **Detectabilidade de adulteração**: cliente consegue validar que payload não foi modificado.

### Negativas

⚠️ **Gerenciamento de secrets**: cliente precisa armazenar secret com segurança. Se vaza, pode ser usada para forjar webhooks.

⚠️ **Sem revogação instantânea**: novo secret é válido imediatamente, mas antigo ainda funciona por 24h. Se secret vaza, não há revogação imediata (decidido como acceptable).

## Questões em Aberto

1. **Encryption de secret no banco**: deve `webhook.secret` ser criptografado com chave master? (Sim, implementar com libsodium ou similar)
2. **Audit de rotações**: logar quem rotacionou secret e quando? (Sim, incluir em tabela de webhooks ou em audit log separada)
3. **Limite de rotações**: pode cliente rotacionar secret 100 vezes por dia (DoS)? (Sim, rate limit no endpoint em fase 2)

## Detalhes Técnicos

**Schema Prisma (webhook):**
```prisma
model Webhook {
  id              String   @id @default(uuid())
  customerId      String   @db.Char(36)
  url             String   // validação: https://
  secretHash      String   // HMAC-SHA256(secret, masterKey) - secret não é stored em plain
  secretHashRotated String? // secret anterior, válido por 24h
  rotatedAt       DateTime?
  active          Boolean  @default(true)
  // ...
}
```

**HMAC em Node.js:**
```typescript
import crypto from 'crypto';
const signature = crypto
  .createHmac('sha256', secret)
  .update(JSON.stringify(payload))
  .digest('hex');
```

**Validação de Secret:**
- Cliente recebe secret em resposta de `POST /webhooks` (uma vez).
- Se cliente perde, não há forma de recuperar (no banco está hashado). Precisa rotar.

## Referências

- Transcrição: `[09:19]` Sofia sobre autenticação de webhook.
- Transcrição: `[09:20]` Sofia propondo HMAC-SHA256 como padrão.
- Transcrição: `[09:21]` Sofia enfatizando secret por endpoint (não global).
- Transcrição: `[09:21]` Sofia sobre rotação de secret com grace period 24h.
- Transcrição: `[09:23]` Sofia sobre TLS obrigatório (https://).
- Transcrição: `[09:44]` Diego e Sofia sobre headers (X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id).
