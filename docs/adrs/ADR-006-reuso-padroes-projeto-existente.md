# ADR-006: Reuso dos Padrões Existentes do Projeto

## Status

Accepted

## Contexto

A aplicação existente segue convenções bem estabelecidas em toda a codebase. Adicionar o módulo de webhooks com padrões diferentes criaria dissonância cognitiva para o time, aumentaria a curva de aprendizado e geraria inconsistências de manutenção. A reunião técnica foi explícita em determinar que o módulo de webhooks deve "seguir igual" aos módulos existentes.

O código fonte atual estabelece os seguintes padrões:

| Padrão | Localização | Descrição |
|--------|-------------|-----------|
| Estrutura modular | `src/modules/{orders,auth,users,customers,products}/` | Cada domínio com `controller`, `service`, `repository`, `routes`, `schemas` |
| Classes de erro | `src/shared/errors/http-errors.ts` | Hierarquia `AppError` → `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, etc. |
| Códigos de erro | `src/shared/errors/http-errors.ts:49,58` | Identificadores como `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK` |
| Logger | `src/shared/logger/index.ts` | Pino com redação de PII, ISO timestamps, request ID |
| Error middleware | `src/middlewares/error.middleware.ts` | Tratamento centralizado de `AppError`, `ZodError` e `Prisma` errors |
| Autenticação/autorização | `src/middlewares/auth.middleware.ts` | `authenticate` + `requireRole(...roles)` com JWT |
| Validação | `src/middlewares/validate.middleware.ts` | Middleware genérico com schemas Zod para body, query e params |
| DI manual | `src/app.ts` | `buildApp` e `buildControllers` com injeção manual de dependências |
| Transaction client | `src/modules/orders/order.service.ts:24,126-178` | Tipo `Prisma.TransactionClient` passado para métodos privados |
| UUIDs | `prisma/schema.prisma` | Todas as entidades usam `@default(uuid()) @db.Char(36)` |

## Decisão

O módulo de webhooks **reutilizará todos os padrões existentes**, sem introduzir novas abstrações, frameworks ou convenções.

**Aplicação concreta dos padrões ao módulo**:

1. **Estrutura**: `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` e `webhook.processor.ts` (lógica do worker)

2. **Erros**: Novas classes estendendo `AppError` com prefixo `WEBHOOK_*`:
   - `WebhookNotFoundError` → código `WEBHOOK_NOT_FOUND`
   - `WebhookInvalidUrlError` → código `WEBHOOK_INVALID_URL`
   - `WebhookSecretRequiredError` → código `WEBHOOK_SECRET_REQUIRED`
   - `WebhookDeliveryFailedError` → código `WEBHOOK_DELIVERY_FAILED`
   - `WebhookPayloadTooLargeError` → código `WEBHOOK_PAYLOAD_TOO_LARGE`

3. **Logger**: Uso do `logger` do `src/shared/logger/index.ts` (Pino) em todo o módulo, com contexto `module: 'webhooks'`

4. **Error middleware**: Sem alterações — `errorMiddleware` já trata `AppError`, `ZodError` e `Prisma` errors. Novas classes de erro do webhook serão capturadas automaticamente.

5. **Autenticação**: Reuso de `authenticate` e `requireRole('ADMIN')` para endpoints administrativos. Endpoints de CRUD exigem `authenticate` apenas.

6. **Validação**: Schemas Zod no `webhook.schemas.ts` validados via middleware `validate`

7. **DI**: `WebhookRepository`, `WebhookService`, `WebhookController` instanciados em `buildControllers` no `src/app.ts`

8. **Transação**: Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx: Prisma.TransactionClient` da transação atual do `OrderService.changeStatus`, seguindo o mesmo padrão de `debitStock` e `replenishStock`

9. **UUIDs**: Todas as tabelas do webhook usam `@default(uuid()) @db.Char(36)`

## Alternativas Consideradas

### Alternativa 1: Criar um diretório `src/webhooks/` fora de `src/modules/`
- **Trade-off**: destacaria o módulo como infraestrutura transversal, mas quebraria a convenção de que todo domínio fica em `src/modules/`. Descartado para manter consistência estrutural.

### Alternativa 2: Usar biblioteca externa de filas (Bull/BullMQ com Redis)
- **Trade-off**: mais recursos (jobs agendados, filas nomeadas, dashboard), mas exigiria Redis como dependência nova e quebraria o padrão de simplicidade da stack atual. Descartado por violar o princípio de reuso máximo do que já existe.

## Consequências

### Positivas
- Zero curva de aprendizado para o time: qualquer desenvolvedor que conhece o módulo de Orders entende o de Webhooks imediatamente
- Manutenção uniforme: refatorações nos padrões base (ex: mudança no error middleware) propagam automaticamente para o módulo de webhooks
- Revisão de código mais rápida: patterns familiares não exigem discussão de estilo
- Segurança consistente: autenticação, autorização e validação seguem exatamente o mesmo modelo dos outros módulos

### Negativas
- Se o padrão existente tiver limitações (ex: DI manual sem container), o módulo de webhooks herda essas limitações
- O módulo de webhooks tem natureza diferente (processamento assíncrono em worker) que pode tensionar padrões pensados para módulos síncronos (ex: `webhook.processor.ts` não se encaixa no molde controller-service-repository)
- Aderência estrita pode impedir inovações pontuais que fariam sentido apenas para o domínio de webhooks
