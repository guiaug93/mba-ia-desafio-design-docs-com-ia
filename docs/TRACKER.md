# Tracker de Rastreabilidade

Cada linha deste tracker mapeia um item documentado (requisito, decisão, contrato, etc.) à sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código fonte da aplicação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|--------------------|-------|-------------|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | CRUD de configuração de webhooks (criar, listar, editar, excluir) para clientes autenticados | TRANSCRICAO | [09:31] Marcos; [09:32] Bruno |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Geração automática de secret de 256 bits na criação, com rotação e grace period de 24h | TRANSCRICAO | [09:21] Sofia; [09:21] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Inserção atômica de evento na outbox dentro da transação do changeStatus | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Worker em processo separado com polling de 2s e chamadas HTTP POST | TRANSCRICAO | [09:09] Diego; [09:11] Diego; [09:11] Larissa |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 sobre o payload, secret por endpoint | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Retry automático 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego; [09:16] Larissa; [09:17] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | DLQ com tabela webhook_dead_letter separada e reprocessamento admin | TRANSCRICAO | [09:18] Diego; [09:18] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Headers de idempotência X-Event-Id para deduplicação do lado cliente | TRANSCRICAO | [09:25] Diego; [09:25] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico de entregas por webhook via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status na inserção da outbox | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Logs estruturados Pino com contexto module: 'webhooks' e métricas | TRANSCRICAO | [09:29] Bruno |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Validação de URL HTTPS obrigatório com erro WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| PRD-NF-01 | docs/PRD.md | Requisito Não Funcional | Inserção na outbox não pode aumentar latência do PATCH /orders/:id/status em >5ms (p95) | TRANSCRICAO | [09:04] Bruno |
| PRD-NF-02 | docs/PRD.md | Requisito Não Funcional | Worker deve processar batch de 10 eventos em ≤12s | TRANSCRICAO | [09:08] Diego |
| PRD-NF-03 | docs/PRD.md | Requisito Não Funcional | Latência máxima de entrega <10s entre changeStatus e chegada ao cliente | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-NF-06 | docs/PRD.md | Requisito Não Funcional | Secrets nunca devem aparecer em logs ou respostas de API de listagem | TRANSCRICAO | [09:23] Sofia; [09:22] Diego |
| PRD-NF-07 | docs/PRD.md | Requisito Não Funcional | Apenas URLs HTTPS são aceitas; http rejeitado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NF-08 | docs/PRD.md | Requisito Não Funcional | SSRF prevenido: bloqueio de IPs privados (RFC 1918), loopback e link-local | TRANSCRICAO | [09:23] Sofia |
| PRD-NF-14 | docs/PRD.md | Requisito Não Funcional | Payload máximo de evento: 64KB | TRANSCRICAO | [09:24] Diego; [09:24] Sofia |
| PRD-NF-15 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP por chamada: 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-ESC-01 | docs/PRD.md | Fora de Escopo | Email de notificação de falha para cliente — adiado para fase 2 | TRANSCRICAO | [09:37] Marcos; [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Fora de Escopo | Dashboard visual de webhooks — projeto separado do time de frontend | TRANSCRICAO | [09:40] Marcos; [09:40] Larissa |
| PRD-ESC-03 | docs/PRD.md | Fora de Escopo | Rate limiting de envio por cliente — observar e decidir depois | TRANSCRICAO | [09:39] Diego; [09:39] Larissa |
| PRD-ESC-04 | docs/PRD.md | Fora de Escopo | Arquivação automática de eventos >30 dias — fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| PRD-RSK-01 | docs/PRD.md | Risco | Cliente não implementar deduplicação por X-Event-Id | TRANSCRICAO | [09:25] Diego; [09:26] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Worker crash silencioso — outbox acumula sem processamento | TRANSCRICAO | [09:11] Diego |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret em logs do cliente | TRANSCRICAO | [09:22] Diego; [09:22] Sofia |
| PRD-RSK-04 | docs/PRD.md | Risco | SSRF via URL maliciosa apontando para rede interna | TRANSCRICAO | [09:23] Sofia |
| PRD-RSK-05 | docs/PRD.md | Risco | Tabela outbox crescer sem limite | TRANSCRICAO | [09:08] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência p95 <10s entre changeStatus e chegada no cliente | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Redução ≥80% de chamadas GET /orders por cliente em 30 dias pós-adoção | TRANSCRICAO | [09:00] Marcos |
| PRD-DEC-01 | docs/PRD.md | Decisão/Trade-off | Outbox MySQL vs Redis Streams — simplicidade operacional vence | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão/Trade-off | At-least-once vs exactly-once — padrão de mercado (Stripe, GitHub) | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão/Trade-off | 5 retries (15h) vs 3 (~36min) — cobre manutenções planejadas | TRANSCRICAO | [09:16] Diego; [09:16] Bruno |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores: Bruno, Diego, Sofia, Marcos | TRANSCRICAO | [09:00] Larissa (participantes) |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no changeStatus — descartado por acoplamento fatal | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams como broker — descartado por overengineering | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ABE-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente — observar e decidir depois | TRANSCRICAO | [09:39] Diego; [09:39] Larissa |
| RFC-ABE-02 | docs/RFC.md | Questão em Aberto | Notificação proativa ao cliente sobre falhas (email) — fase 2 | TRANSCRICAO | [09:37] Marcos; [09:37] Larissa |
| RFC-RSK-01 | docs/RFC.md | Risco | Migração de schema quebrar transação do changeStatus | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| RFC-RSK-02 | docs/RFC.md | Risco | Cliente cadastrar URL maliciosa ou interna (SSRF) | TRANSCRICAO | [09:23] Sofia |
| FDD-FLO-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro de $transaction do changeStatus | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-FLO-02 | docs/FDD.md | Fluxo | Função publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| FDD-FLO-03 | docs/FDD.md | Fluxo | Worker: polling 2s → batch 10 → HTTP POST → retry/DLQ | TRANSCRICAO | [09:08] Diego; [09:09] Diego |
| FDD-FLO-04 | docs/FDD.md | Fluxo | DLQ: movimentação após 5 falhas + replay manual admin | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cria configuração com secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks — lista webhooks com preview de secret | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edita url, events, active | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remove configuração | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico paginado | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotação com grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay — admin replay DLQ | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers outbound: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Formato payload: event_id, event_type, timestamp, order_id, order_number, status, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Matriz de Erros | WEBHOOK_NOT_FOUND (404) — configuração não encontrada | TRANSCRICAO | [09:28] Bruno; [09:29] Larissa |
| FDD-ERR-02 | docs/FDD.md | Matriz de Erros | WEBHOOK_INVALID_URL (400) — URL não é HTTPS ou malformada | TRANSCRICAO | [09:23] Sofia; [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Matriz de Erros | WEBHOOK_DUPLICATE_URL (409) — mesmo customer + mesma URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Matriz de Erros | WEBHOOK_PAYLOAD_TOO_LARGE — payload excede 64KB | TRANSCRICAO | [09:24] Diego; [09:24] Sofia |
| FDD-ERR-05 | docs/FDD.md | Matriz de Erros | Prefixo WEBHOOK_ obrigatório em todos os códigos de erro | TRANSCRICAO | [09:29] Larissa |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout HTTP 10s, 5 retries com backoff, DLQ, graceful shutdown | TRANSCRICAO | [09:15] Diego; [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Recuperação de zombie events PROCESSING >5min no startup | TRANSCRICAO | [09:08] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Métricas: webhook_deliveries_total, webhook_outbox_pending_count, etc. | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs Pino com contexto module: 'webhooks', redact de secrets | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | X-Event-Id propagado da outbox ao header HTTP e logs | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | src/modules/orders/order.service.ts — changeStatus: inserir na outbox dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | src/shared/errors/http-errors.ts — novas classes WEBHOOK_* estendendo AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Integração | src/middlewares/auth.middleware.ts — reuso de authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | src/app.ts — registrar WebhookController em buildControllers e buildApiRouter | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | src/server.ts — referência para criar src/worker.ts (mesmo prisma, logger, graceful shutdown) | CODIGO | src/server.ts |
| FDD-INT-06 | docs/FDD.md | Integração | prisma/schema.prisma — novos modelos: WebhookConfig, WebhookOutbox, WebhookDelivery, WebhookDeadLetter | CODIGO | prisma/schema.prisma |
| FDD-INT-07 | docs/FDD.md | Integração | src/routes/index.ts — adicionar webhooks ao tipo Controllers e buildApiRouter | CODIGO | src/routes/index.ts |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL — transação atômica com changeStatus | TRANSCRICAO | [09:06] Diego; [09:07] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono — acoplamento fatal de latência e transação | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams — overengineering para o contexto atual | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Política de retry: 5 tentativas com backoff 1m/5m/30m/2h/12h + DLQ | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | 3 tentativas apenas — não cobre manutenções de 1-2h | TRANSCRICAO | [09:16] Bruno; [09:16] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | Retry indefinido — poluição da outbox sem fechamento | TRANSCRICAO | [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | HMAC-SHA256 com secret por endpoint, rotação com grace period 24h | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa Descartada | Secret global única — vazamento compromete todos os clientes | TRANSCRICAO | [09:21] Sofia |
| ADR-003-ALT-02 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa Descartada | JWT outbound assimétrico — complexidade desnecessária para clientes B2B | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-event-id.md | Decisão | Garantia at-least-once com X-Event-Id para deduplicação do cliente | TRANSCRICAO | [09:25] Diego; [09:25] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-at-least-once-event-id.md | Alternativa Descartada | At-most-once — perda de eventos inaceitável | TRANSCRICAO | [09:25] Diego |
| ADR-004-ALT-02 | docs/adrs/ADR-004-at-least-once-event-id.md | Alternativa Descartada | Exactly-once com ACK — complexidade de two-phase commit | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado com polling de 2s | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Alternativa Descartada | Worker no mesmo processo da API — acoplamento de ciclo de vida | TRANSCRICAO | [09:11] Diego; [09:11] Larissa |
| ADR-005-ALT-02 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Alternativa Descartada | Trigger de banco MySQL — sem NOTIFY/LISTEN, frágil | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-projeto-existente.md | Decisão | Reuso de AppError, Pino, Zod, middleware de erro, estrutura modular | TRANSCRICAO | [09:28] Bruno; [09:29] Bruno; [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-padroes-projeto-existente.md | Alternativa Descartada | Diretório src/webhooks/ fora de src/modules/ — quebra convenção | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-padroes-projeto-existente.md | Alternativa Descartada | Biblioteca externa de filas (Bull/BullMQ) — introduz Redis como dependência | TRANSCRICAO | [09:07] Diego |
| TRK-SEC-01 | docs/PRD.md | Restrição | Secret nunca aparece em respostas de listagem — apenas preview de 8 chars | TRANSCRICAO | [09:21] Sofia; [09:23] Sofia |
| TRK-PAY-01 | docs/FDD.md | Restrição | Snapshot do payload na inserção da outbox, não no envio | TRANSCRICAO | [09:52] Larissa; [09:52] Diego; [09:52] Bruno |
| TRK-UUID-01 | docs/FDD.md | Restrição | IDs usam UUID v4, seguindo padrão do projeto (@default(uuid())) | CODIGO | prisma/schema.prisma |
| TRK-TRX-01 | docs/FDD.md | Restrição | Função publishWebhookEvent recebe Prisma.TransactionClient como parâmetro | CODIGO | src/modules/orders/order.service.ts |
| TRK-ERR-01 | docs/FDD.md | Restrição | Novas classes de erro estendem AppError com padrão de errorCode + statusCode | CODIGO | src/shared/errors/http-errors.ts |
| TRK-LOG-01 | docs/FDD.md | Restrição | Logger Pino configurado com redact de *.secret, *.token, *.password | CODIGO | src/shared/logger/index.ts |
