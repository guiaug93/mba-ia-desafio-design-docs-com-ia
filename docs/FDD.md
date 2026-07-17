# FDD — Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e Motivação Técnica

A aplicação é um monólito Node.js + TypeScript com Express, Prisma ORM sobre MySQL 8.0 e autenticação JWT. Cada domínio segue o padrão `src/modules/{domain}/{domain}.{controller,service,repository,routes,schemas}.ts`. A mudança de status de pedidos ocorre em `src/modules/orders/order.service.ts:126-179` (método `changeStatus`), dentro de uma transação `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`.

O problema técnico: a aplicação não tem mecanismo para notificar sistemas externos sobre essas mudanças. Clientes B2B fazem polling via `GET /orders` para detectar alterações, o que é ineficiente e não escala.

A feature deve introduzir notificação outbound sem comprometer a integridade transacional existente, sem adicionar infraestrutura externa (Redis, RabbitMQ) e sem acoplar a latência da API à disponibilidade de sistemas de terceiros.

## 2. Objetivos Técnicos

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Notificar cliente sobre mudança de status | Latência de entrega (p95) | <10 segundos entre `changeStatus` e chegada no cliente |
| Não degradar a API existente | Latência do endpoint `PATCH /orders/:id/status` (p95) | Sem aumento mensurável (<5ms adicional pela inserção na outbox) |
| Não perder eventos | Taxa de perda de eventos | 0 eventos perdidos após commit da transação |
| Resiliência contra falhas do cliente | Janela de retry coberta | 15 horas (5 tentativas com backoff) |
| Segurança na entrega | Integridade do payload | HMAC-SHA256 verificável pelo cliente |
| Reuso da stack existente | Novas dependências | Zero bibliotecas novas além das já no `package.json` |
| Rastreabilidade | Eventos rastreáveis ponta a ponta | UUID único por evento (`X-Event-Id`) presente em logs, outbox e headers |

## 3. Escopo e Exclusões

### Incluso
- Tabelas `webhook_config`, `webhook_outbox`, `webhook_delivery` e `webhook_dead_letter`
- CRUD de configuração de webhooks (POST, GET, PATCH, DELETE)
- Integração no `changeStatus` com inserção atômica na outbox
- Worker em processo separado com polling de 2s
- Envio HTTP POST autenticado com HMAC-SHA256
- Política de retry com backoff (5 tentativas) + DLQ
- Rotação de secret com grace period de 24h
- Endpoint admin de replay de DLQ
- Endpoint de histórico de entregas
- Schemas Zod, classes de erro `WEBHOOK_*`, logs estruturados

### Exclusões
- Envio de e-mail ao cliente quando webhook falha (adiado para fase 2 — [09:37] Larissa)
- Dashboard visual de webhooks (projeto separado do time de frontend — [09:40] Larissa)
- Rate limiting de envio por cliente (observar e decidir depois — [09:39] Diego)
- Arquivação automática de eventos antigos (fora do escopo da feature — [09:08] Diego)
- Painless migration ou suporte a múltiplos workers em paralelo (single-worker por design)

## 4. Fluxos Detalhados

### 4.1 Criação do Evento na Outbox

**Trigger**: `PATCH /api/v1/orders/:id/status` → `OrderController.changeStatus` → `OrderService.changeStatus`

**Fluxo** (dentro da transação existente em `prisma.$transaction`):

```
1. Buscar order com items (tx.order.findUnique)
2. Validar transição (canTransition)
3. Debitar/repor estoque (debitStock/replenishStock)
4. UPDATE orders SET status = to
5. INSERT order_status_history
6. [NOVO] publishWebhookEvent(tx, order, fromStatus, toStatus)
   ├─ Buscar webhook_configs ativos do customer com filtro de status match
   ├─ Para cada config: renderizar payload JSON snapshot
   ├─ Gerar UUID (event_id)
   ├─ INSERT webhook_outbox (id, event_type, payload, status='PENDING', attempts=0, next_retry_at=NOW())
   └─ Se nenhum webhook configurado: NOOP (não insere linha)
7. Retornar order refreshed
```

**Função `publishWebhookEvent`**:

```typescript
// Localização: src/modules/webhooks/webhook.service.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order & { items: OrderItem[]; customer: { id: string; name: string; email: string } },
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const configs = await tx.webhookConfig.findMany({
    where: {
      customerId: order.customerId,
      active: true,
      events: { has: toStatus },
    },
  });

  if (configs.length === 0) return;

  const eventId = uuidv4();
  const payload: WebhookPayload = {
    event_id: eventId,
    event_type: 'order.status_changed',
    timestamp: new Date().toISOString(),
    order_id: order.id,
    order_number: order.orderNumber,
    from_status: fromStatus,
    to_status: toStatus,
    customer_id: order.customerId,
    total_cents: order.totalCents,
    notes: order.notes,
  };

  await tx.webhookOutbox.createMany({
    data: configs.map((config) => ({
      webhookConfigId: config.id,
      eventType: 'order.status_changed',
      payload: JSON.stringify(payload),
      status: 'PENDING',
      attempts: 0,
      nextRetryAt: new Date(),
    })),
  });
}
```

### 4.2 Processamento pelo Worker

**Entry point**: `src/worker.ts` — processo separado, iniciado via `npm run worker`

**Loop principal**:

```
while (true) {
  1. Buscar eventos PENDING com next_retry_at <= NOW()
     SELECT * FROM webhook_outbox
     WHERE status = 'PENDING' AND next_retry_at <= NOW()
     ORDER BY created_at ASC
     LIMIT 10

  2. Para cada evento:
     a. Marcar como PROCESSING
        UPDATE webhook_outbox SET status = 'PROCESSING' WHERE id = ?

     b. Buscar webhook_config associado
        SELECT url, secret, id FROM webhook_config WHERE id = ?

     c. Montar request HTTP:
        - URL: config.url
        - Method: POST
        - Headers: Content-Type, X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id
        - Body: payload (JSON string)
        - Timeout: 10s

     d. Calcular HMAC-SHA256:
        const signature = crypto.createHmac('sha256', config.secret)
                                .update(payload)
                                .digest('hex');

     e. Enviar HTTP POST (fetch com AbortController)

     f. Se sucesso (2xx):
        - INSERT webhook_delivery (outbox_id, webhook_id, status_code, response_body, duration_ms, delivered_at)
        - UPDATE webhook_outbox SET status = 'SENT'

     g. Se falha (não-2xx, timeout, network error):
        attempts++
        IF attempts >= 5:
          - INSERT webhook_dead_letter (outbox_id, webhook_id, payload, failure_reason, last_status_code, last_error_body)
          - UPDATE webhook_outbox SET status = 'FAILED'
        ELSE:
          - Calcular next_retry_at com backoff (1m, 5m, 30m, 2h, 12h)
          - UPDATE webhook_outbox SET status = 'PENDING', attempts = ?, next_retry_at = ?

  3. Dormir 2 segundos
}
```

**Tratamento de concorrência**: o batch é processado sequencialmente. Eventos em `PROCESSING` não são buscados no próximo ciclo. Se o worker morrer com eventos em `PROCESSING`, um mecanismo de recuperação no startup zera eventos `PROCESSING` com `updated_at > 5 minutos` de volta para `PENDING`.

### 4.3 Retry e Backoff

**Progressão fixa** baseada em `attempts`:

```typescript
const BACKOFF_SCHEDULE: Record<number, number> = {
  1: 60_000,        // 1 minuto
  2: 300_000,       // 5 minutos
  3: 1_800_000,     // 30 minutos
  4: 7_200_000,     // 2 horas
  5: 43_200_000,    // 12 horas (última tentativa)
};

function calcNextRetryAt(attempts: number): Date {
  const delayMs = BACKOFF_SCHEDULE[attempts] ?? 43_200_000;
  return new Date(Date.now() + delayMs);
}
```

### 4.4 Dead Letter Queue (DLQ)

**Tabela**: `webhook_dead_letter`

**Movimentação para DLQ**: ocorre quando `attempts >= 5` após falha. O registro na outbox é marcado como `FAILED` e o evento é copiado para a DLQ com metadados de falha.

**Reprocessamento**: `POST /admin/webhooks/dead-letter/:id/replay`
- Exige role `ADMIN` (`requireRole('ADMIN')`)
- Loga `userId`, `deadLetterId` e timestamp para auditoria
- Insere novo registro na `webhook_outbox` com `status = PENDING`, `attempts = 0`, `next_retry_at = NOW()`, reutilizando o payload da DLQ
- Não remove o registro da DLQ — apenas marca como `REPLAYED`

## 5. Contratos Públicos

### 5.1 POST /api/v1/webhooks

Cria configuração de webhook para um customer.

**Autenticação**: Bearer JWT (qualquer role autenticada)

**Request Body**:
```json
{
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://api.cliente.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Response 201**:
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://api.cliente.com.br/webhooks/oms",
  "secret": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "created_at": "2026-07-17T14:30:00.000Z",
  "updated_at": "2026-07-17T14:30:00.000Z"
}
```

**Status codes**: `201 Created`, `400 Validation Error`, `401 Unauthorized`, `404 Customer Not Found`

### 5.2 GET /api/v1/webhooks

Lista webhooks com filtro opcional por customer.

**Autenticação**: Bearer JWT

**Query params**: `?customer_id=<uuid>&page=1&page_size=20`

**Response 200**:
```json
{
  "data": [
    {
      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "url": "https://api.cliente.com.br/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secret_preview": "a1b2c3d4...",
      "created_at": "2026-07-17T14:30:00.000Z"
    }
  ],
  "pagination": { "page": 1, "page_size": 20, "total": 1, "total_pages": 1 }
}
```

**Nota**: a resposta da listagem nunca retorna a secret completa — apenas os 8 primeiros caracteres como preview.

### 5.3 PATCH /api/v1/webhooks/:id

Atualiza url, events ou active de um webhook.

**Autenticação**: Bearer JWT

**Request Body**:
```json
{
  "url": "https://novo-endpoint.cliente.com.br/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response 200**:
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "url": "https://novo-endpoint.cliente.com.br/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secret_preview": "a1b2c3d4...",
  "updated_at": "2026-07-17T15:00:00.000Z"
}
```

**Status codes**: `200 OK`, `400 Validation Error`, `404 Webhook Not Found`

### 5.4 DELETE /api/v1/webhooks/:id

Remove configuração de webhook. Eventos já na outbox para este webhook não são cancelados — apenas novas mudanças de status não gerarão eventos para este endpoint.

**Autenticação**: Bearer JWT

**Response 204**: No Content

### 5.5 GET /api/v1/webhooks/:id/deliveries

Retorna histórico de entregas de um webhook.

**Autenticação**: Bearer JWT

**Response 200**:
```json
{
  "data": [
    {
      "id": "d4e5f6a7-b8c9-0123-def4-567890abcdef",
      "event_id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
      "event_type": "order.status_changed",
      "status": "success",
      "status_code": 200,
      "duration_ms": 342,
      "delivered_at": "2026-07-17T14:31:05.000Z",
      "order_id": "550e8400-e29b-41d4-a716-446655440000",
      "to_status": "SHIPPED"
    }
  ],
  "pagination": { "page": 1, "page_size": 20, "total": 145, "total_pages": 8 }
}
```

### 5.6 POST /api/v1/webhooks/:id/rotate-secret

Gera nova secret mantendo a antiga válida por 24h.

**Autenticação**: Bearer JWT

**Response 200**:
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "secret": "f1e2d3c4b5a69788796a5b4c3d2e1f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
  "previous_secret_valid_until": "2026-07-18T15:00:00.000Z"
}
```

### 5.7 POST /admin/webhooks/dead-letter/:id/replay

Reprocessa evento da DLQ.

**Autenticação**: Bearer JWT + role `ADMIN`

**Response 200**:
```json
{
  "message": "Event replayed successfully",
  "new_outbox_id": "f6a7b8c9-d0e1-2345-f678-90abcdef0123"
}
```

**Status codes**: `200 OK`, `403 Forbidden` (não-ADMIN), `404 Not Found`

### 5.8 Headers do Webhook Outbound

Headers enviados pelo worker nas chamadas HTTP aos clientes:

| Header | Exemplo | Descrição |
|--------|---------|-----------|
| `Content-Type` | `application/json` | Tipo do corpo |
| `X-Event-Id` | `550e8400-e29b-41d4-a716-446655440000` | UUID único do evento (deduplicação) |
| `X-Signature` | `sha256=9f86d081884c7d659...` | HMAC-SHA256 do payload |
| `X-Timestamp` | `2026-07-17T14:31:05.000Z` | ISO 8601 do momento do envio |
| `X-Webhook-Id` | `b2c3d4e5-f6a7-8901-bcde-f12345678901` | ID da configuração do webhook |
| `User-Agent` | `OMS-Webhook/1.0` | Identificação do remetente |

## 6. Matriz de Erros Previstos

Todos os códigos seguem o prefixo `WEBHOOK_*` conforme decidido na reunião ([09:29] Bruno/Larissa).

| Código | HTTP Status | Condição | Mensagem |
|--------|-------------|----------|----------|
| `WEBHOOK_NOT_FOUND` | 404 | Configuração de webhook não encontrada | `Webhook <id> not found` |
| `WEBHOOK_INVALID_URL` | 400 | URL não é HTTPS ou é malformada | `Webhook URL must use HTTPS` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Tentativa de criar webhook sem secret (não ocorre — secret é gerada) | `Webhook secret is required` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Customer referenciado não existe | `Customer <id> not found` |
| `WEBHOOK_DUPLICATE_URL` | 409 | Mesmo customer já tem webhook com essa URL | `A webhook for this URL already exists` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 500 | Payload do evento excede 64KB (erro interno, não enviado ao cliente) | `Webhook payload exceeds 64KB limit` |
| `WEBHOOK_DELIVERY_FAILED` | — | Erro interno do worker, não exposto via API | `Webhook delivery failed: <reason>` |
| `WEBHOOK_INVALID_EVENT` | 400 | Status de evento não é um OrderStatus válido | `Invalid event: <status>` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Registro DLQ não encontrado | `Dead letter <id> not found` |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | Tentativa de replay de DLQ já reprocessado | `Dead letter <id> was already replayed` |

**Implementação das classes de erro** (seguindo o padrão de `src/shared/errors/http-errors.ts`):

```typescript
// src/modules/webhooks/webhook.errors.ts
export class WebhookNotFoundError extends NotFoundError {
  constructor(id: string) {
    super(`Webhook ${id}`, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends ValidationError {
  constructor(url: string) {
    super('Webhook URL must use HTTPS', [{ path: 'url', message: `Invalid URL: ${url}` }]);
    this.errorCode = 'WEBHOOK_INVALID_URL';
  }
}
```

## 7. Estratégias de Resiliência

| Mecanismo | Configuração | Descrição |
|-----------|-------------|-----------|
| **Timeout HTTP** | 10 segundos | `AbortController` aborta fetch após 10s sem resposta |
| **Retry com backoff** | 5 tentativas, 1m/5m/30m/2h/12h | Intervalo progressivo entre tentativas — ver ADR-002 |
| **DLQ** | Tabela `webhook_dead_letter` | Eventos que esgotaram tentativas são movidos para DLQ |
| **Recuperação de zombie events** | Timeout de 5min em `PROCESSING` | No startup do worker, eventos marcados como `PROCESSING` há mais de 5 minutos voltam para `PENDING` |
| **Graceful shutdown** | SIGINT/SIGTERM | Worker finaliza batch atual, desconecta Prisma e sai |
| **Idempotência do cliente** | `X-Event-Id` | Cliente deduplica eventos recebidos mais de uma vez |
| **Atomicidade outbox** | Transação SQL | Evento só existe se a transação principal commitou |
| **Batch pequeno** | 10 eventos por ciclo | Limita concorrência e evita starvation de eventos individuais |
| **Validação de URL** | Zod regex + bloqueio de IPs privados | Impede SSRF — rejeita URLs com hostnames que resolvem para `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` |

## 8. Observabilidade

### 8.1 Métricas

Métricas expostas via endpoint `GET /metrics` (Prometheus text format) ou logs estruturados para agregação externa.

| Métrica | Tipo | Labels | Descrição |
|---------|------|--------|-----------|
| `webhook_outbox_events_created_total` | Counter | `event_type` | Eventos inseridos na outbox |
| `webhook_deliveries_total` | Counter | `status` (success/failure), `webhook_id` | Entregas realizadas |
| `webhook_delivery_duration_ms` | Histogram | `webhook_id` | Latência de entrega HTTP |
| `webhook_outbox_pending_count` | Gauge | — | Eventos pendentes na outbox |
| `webhook_outbox_processing_count` | Gauge | — | Eventos em processamento |
| `webhook_dead_letter_count` | Gauge | — | Eventos na DLQ |
| `webhook_worker_cycle_duration_ms` | Histogram | — | Duração do ciclo de polling |
| `webhook_worker_errors_total` | Counter | `error_type` | Erros internos do worker |

### 8.2 Logs

Logger Pino (`src/shared/logger/index.ts`) com contexto `module: 'webhooks'`.

**Formato**: JSON estruturado, mesmo padrão do resto da aplicação.

**Campos padrão em todos os logs do módulo**: `module`, `requestId` (quando no contexto HTTP), `webhookId`, `eventId`.

**Níveis**:
- `INFO`: criação de webhook, envio bem-sucedido, replay de DLQ
- `WARN`: falha de entrega (tentativa < 5), secret rotacionada
- `ERROR`: evento movido para DLQ, falha no worker, SSRF detectado

**Proteção PII**: o logger Pino já está configurado para redactar `*.token`, `*.password`, `*.secret` (`src/shared/logger/index.ts:4-11`). Headers de assinatura e secrets não aparecem em logs.

### 8.3 Tracing

- `X-Event-Id` gerado na inserção da outbox é propagado para headers HTTP outbound e logs
- `X-Request-Id` já implementado em `src/middlewares/request-logger.middleware.ts` é mantido nas requisições inbound
- Para tracing distribuído futuro, os headers `traceparent` e `tracestate` (W3C Trace Context) podem ser injetados no worker, mas estão fora do escopo desta feature

## 9. Dependências e Compatibilidade

### Dependências existentes (reutilizadas)
| Biblioteca | Versão | Uso no módulo |
|------------|--------|---------------|
| `express` | 4.21.1 | Rotas e middlewares |
| `@prisma/client` | 5.22.0 | ORM — novas tabelas, transações |
| `zod` | 3.23.8 | Schemas de validação |
| `pino` | 9.5.0 | Logger estruturado |
| `uuid` | 11.0.3 | Geração de UUIDs para event_id e ids de registro |
| `jsonwebtoken` | 9.0.2 | Autenticação (reuso do middleware existente) |

### Novas dependências: **Nenhuma**

A funcionalidade de HMAC-SHA256 usa o módulo nativo `crypto` do Node.js. Chamadas HTTP usam `fetch` nativo (Node 20+). Nenhuma biblioteca nova no `package.json`.

### Compatibilidade
- Node.js ≥ 20 (já é o requisito do projeto — `package.json:7`)
- MySQL 8.0 (já provisionado — `docker-compose.yml:2`)
- Prisma 5.22.0 (já instalado)
- O módulo de webhooks não altera comportamento de nenhum módulo existente — apenas adiciona uma chamada de função dentro da transação do `changeStatus`

## 10. Critérios de Aceite Técnicos

### Funcional
- [ ] `POST /api/v1/webhooks` cria configuração com secret gerada automaticamente
- [ ] `PATCH /orders/:id/status` insere evento na outbox dentro da mesma transação
- [ ] Worker entrega evento ao endpoint do cliente com headers de segurança corretos
- [ ] Worker retenta 5 vezes com backoff progressivo
- [ ] Worker move evento para DLQ após 5 falhas
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` reprocessa evento (apenas ADMIN)
- [ ] Rotação de secret mantém anterior válida por 24h
- [ ] `GET /webhooks/:id/deliveries` retorna histórico paginado
- [ ] URLs HTTP são rejeitadas na criação com erro `WEBHOOK_INVALID_URL`
- [ ] Múltiplos webhooks do mesmo customer recebem eventos independentes

### Performance
- [ ] Latência do `PATCH /orders/:id/status` não aumenta mais que 5ms (p95)
- [ ] Worker processa batch de 10 eventos em < 12 segundos (incluindo timeout HTTP)
- [ ] Polling não consome mais que 5% de CPU em idle

### Resiliência
- [ ] Worker sobrevive a crash e recupera eventos em `PROCESSING` no restart
- [ ] Falha de um webhook não bloqueia processamento de outros no mesmo batch
- [ ] Graceful shutdown completa batch atual antes de sair

### Observabilidade
- [ ] Métricas de entregas expostas via logs estruturados
- [ ] Todo erro do worker gera log nível ERROR com `eventId` e `webhookId`
- [ ] Logs do worker e da API são distinguíveis (`module: 'webhooks'`)

### Segurança
- [ ] HMAC-SHA256 calculado corretamente e verificável pelo cliente
- [ ] Secret nunca aparece em logs ou respostas de listagem
- [ ] SSRF prevenido — URLs com IPs privados são rejeitadas
- [ ] Replay de DLQ loga `userId` para auditoria

## 11. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Worker crash silencioso — outbox acumula sem processamento | Média | Alto | Healthcheck + alerta em `webhook_outbox_pending_count > 100` por >5min. Supervisor (systemd/Docker restart policy) |
| Race condition: dois workers processando o mesmo evento | Baixa (single-worker por design) | Crítico | Documentar que escala horizontal requer lock distribuído. No startup, worker verifica se já existe outro em execução via lock em linha do banco |
| Secret vazar em log | Baixa | Alto | Pino já redacta `*.secret`. Revisão de segurança dos log statements do worker antes do deploy |
| SSRF via URL maliciosa | Baixa | Crítico | Validação de URL com bloqueio de IPs privados + revisão de segurança pela Sofia |
| Transação `changeStatus` ficar mais lenta com muitos webhooks | Baixa | Médio | A inserção na outbox é um `createMany` simples. Índices em `webhook_config(customer_id, active)` garantem lookup rápido. Se o customer tiver >50 webhooks, alertar e considerar arquitetura de fanout |
| Evento duplicado se o worker crashar entre HTTP 200 e UPDATE outbox | Baixa | Baixo | É o cenário para o qual o at-least-once foi projetado. Cliente deduplica por `X-Event-Id` |

## 12. Integração com o Sistema Existente

### 12.1 `src/modules/orders/order.service.ts`

**Como integrar**: Dentro da transação `prisma.$transaction` do método `changeStatus` (linha 131), após a inserção em `order_status_history` (linha 159-167) e antes do `findUnique` de refresh (linha 169-176), adicionar a chamada:

```typescript
await publishWebhookEvent(tx, {
  ...order,
  customer: /* buscar customer */,
}, from, to);
```

A função `publishWebhookEvent` segue o mesmo padrão de `debitStock` (linha 204) e `replenishStock` (linha 233): função pura que recebe o `tx: Prisma.TransactionClient` e opera dentro da transação. Se a inserção na outbox falhar, a transação inteira dá rollback — o status não muda e nenhum evento é gerado.

### 12.2 `src/shared/errors/http-errors.ts`

**Como integrar**: Adicionar novas classes de erro no mesmo arquivo ou em arquivo dedicado `src/modules/webhooks/webhook.errors.ts`, estendendo as classes base existentes:

- `WebhookNotFoundError extends NotFoundError` — reusa o status 404 e o padrão de mensagem `'<Resource> not found'`
- `WebhookInvalidUrlError extends ValidationError` — reusa o status 400 e o padrão de details com `path` e `message`
- `WebhookDuplicateUrlError extends ConflictError` — reusa o status 409 e o construtor flexível com `code` e `details`

O `errorMiddleware` (`src/middlewares/error.middleware.ts:14-24`) já captura qualquer instância de `AppError` e serializa `errorCode`, `message` e `details` — as novas classes funcionam sem nenhuma alteração no middleware.

### 12.3 `src/middlewares/auth.middleware.ts`

**Como integrar**: O módulo de webhooks reutiliza dois middlewares diretamente:

- `authenticate` (linha 27): aplicado em todas as rotas de webhook (`router.use(authenticate)`)
- `requireRole('ADMIN')` (linha 49): aplicado especificamente na rota `POST /admin/webhooks/dead-letter/:id/replay`

Exemplo de uso nas rotas:
```typescript
import { authenticate, requireRole } from '../../middlewares/auth.middleware.js';

router.use(authenticate);
router.post('/', validate({ body: createWebhookSchema }), controller.create);
// ...
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  requireRole('ADMIN'),
  validate({ params: deadLetterIdParamSchema }),
  controller.replayDeadLetter,
);
```

### 12.4 `src/app.ts`

**Como integrar**: Registrar o módulo de webhooks na função `buildControllers` (linha 26-53) e no `buildApiRouter` (linha 55-76):

1. Em `buildControllers`: instanciar `WebhookRepository`, `WebhookService`, `WebhookController` e adicionar ao objeto `Controllers`
2. O tipo `Controllers` em `src/routes/index.ts:13-19` deve ser estendido com `webhooks: WebhookController`
3. Em `buildApiRouter`: adicionar `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`

O `OrderService` atualmente recebe `OrderRepository` e `PrismaClient` (linha 43). Para integrar a outbox, o `OrderService` não deve receber o repositório de webhooks como dependência — a função `publishWebhookEvent` é importada diretamente como função pura, evitando acoplamento de domínios e círculo de dependências.

### 12.5 `src/server.ts`

**Como integrar**: O `src/server.ts` é a referência para a criação do `src/worker.ts`. Ambos compartilham:

- Import do `prisma` de `src/config/database.ts`
- Import do `logger` de `src/shared/logger/index.ts`
- Inicialização via `async function bootstrap()`
- Graceful shutdown com `process.on('SIGINT', ...)` e `process.on('SIGTERM', ...)`

O `src/worker.ts` difere por não criar servidor HTTP — apenas inicia o loop de polling.

### 12.6 `prisma/schema.prisma`

**Como integrar**: Adicionar quatro novos modelos ao schema, mantendo consistência com os padrões existentes:

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  SENT
  FAILED
}

model WebhookConfig {
  id        String   @id @default(uuid()) @db.Char(36)
  customerId String  @db.Char(36)
  url       String   @db.VarChar(2048)
  secret    String   @db.VarChar(512)
  events    Json     // OrderStatus[]
  active    Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])
  outbox   WebhookOutbox[]

  @@index([customerId, active])
  @@map("webhook_configs")
}

model WebhookOutbox {
  id              String              @id @default(uuid()) @db.Char(36)
  webhookConfigId String              @db.Char(36)
  eventType       String              @db.VarChar(100)
  payload         String              @db.Text
  status          WebhookOutboxStatus @default(PENDING)
  attempts        Int                 @default(0)
  nextRetryAt     DateTime            @default(now())
  createdAt       DateTime            @default(now())
  updatedAt       DateTime            @updatedAt

  config   WebhookConfig       @relation(fields: [webhookConfigId], references: [id])
  delivery WebhookDelivery?
  deadLetter WebhookDeadLetter?

  @@index([status, nextRetryAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id          String   @id @default(uuid()) @db.Char(36)
  outboxId    String   @unique @db.Char(36)
  webhookId   String   @db.Char(36)
  statusCode  Int
  responseBody String? @db.Text
  durationMs  Int
  deliveredAt DateTime @default(now())

  outbox WebhookOutbox @relation(fields: [outboxId], references: [id])

  @@index([webhookId, deliveredAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id              String   @id @default(uuid()) @db.Char(36)
  outboxId        String   @unique @db.Char(36)
  webhookId       String   @db.Char(36)
  payload         String   @db.Text
  failureReason   String   @db.VarChar(1000)
  lastStatusCode  Int?
  lastErrorBody   String?  @db.Text
  failedAt        DateTime @default(now())
  replayedAt      DateTime?
  replayedBy      String?  @db.Char(36)

  outbox WebhookOutbox @relation(fields: [outboxId], references: [id])

  @@index([webhookId, failedAt])
  @@map("webhook_dead_letters")
}
```

### 12.7 `src/routes/index.ts`

**Como integrar**: O tipo `Controllers` (linha 13) e a função `buildApiRouter` (linha 21) devem incluir o controller de webhooks:

```typescript
export type Controllers = {
  auth: AuthController;
  users: UserController;
  customers: CustomerController;
  products: ProductController;
  orders: OrderController;
  webhooks: WebhookController;  // novo
};

export function buildApiRouter(controllers: Controllers): Router {
  const router = Router();
  router.use('/auth', buildAuthRouter(controllers.auth));
  router.use('/users', buildUserRouter(controllers.users));
  router.use('/customers', buildCustomerRouter(controllers.customers));
  router.use('/products', buildProductRouter(controllers.products));
  router.use('/orders', buildOrderRouter(controllers.orders));
  router.use('/webhooks', buildWebhookRouter(controllers.webhooks));  // novo
  return router;
}
```
