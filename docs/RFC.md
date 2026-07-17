# RFC-001: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|-------|-------|
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Proposed |
| **Data** | 2026-07-17 |
| **Revisores** | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança), Marcos (PM) |

## Resumo Executivo (TL;DR)

Propomos implementar um sistema de webhooks outbound que notifica clientes B2B em tempo real sobre mudanças de status de pedidos, substituindo o modelo atual de polling via `GET /orders`. A solução usa **Transactional Outbox no MySQL** — dentro da mesma transação de `changeStatus`, um evento é registrado numa tabela de outbox. Um **worker em processo separado** faz polling a cada 2 segundos e dispara chamadas HTTP autenticadas via **HMAC-SHA256** com secret por endpoint, garantindo entrega **at-least-once** com `X-Event-Id` para deduplicação do lado cliente. Falhas seguem política de **retry com backoff exponencial (5 tentativas)** e eventos exauridos vão para uma **DLQ persistida** com reprocessamento manual administrativo. Todo o módulo reutiliza os padrões existentes do projeto (AppError, Pino, Zod, middleware de erro, autenticação JWT).

## Contexto e Problema

A aplicação atual (`src/modules/orders/order.service.ts`) gerencia o ciclo de vida de pedidos com máquina de estados, controle transacional de estoque e auditoria via `order_status_history`. Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — dependem desses dados para seus fluxos operacionais (logística, faturamento, atendimento), mas hoje precisam fazer polling periódico via `GET /orders` para detectar mudanças de status. Isso é lento, caro em termos de recursos e não escala.

A aplicação **não possui nenhum mecanismo de notificação externa**: não há filas, mensageria, webhooks ou eventos. A mudança de status é puramente interna e síncrona. Preencher essa lacuna é o objetivo da feature.

### Por que isso importa agora
A Atlas Comercial sinalizou que pode migrar para concorrente se a funcionalidade não for entregue até o fim do trimestre. O prazo interno é de 3 sprints.

## Proposta Técnica

### Visão Geral

```
┌──────────────────────────────────────────────────────┐
│ API (server.ts)                                      │
│  POST /orders/:id/status  ──► OrderService           │
│                                  │                   │
│                          $transaction()              │
│                            ├─ UPDATE orders          │
│                            ├─ INSERT history         │
│                            ├─ UPDATE stock           │
│                            └─ INSERT webhook_outbox  │
│                                                      │
├──────────────────────────────────────────────────────┤
│ Worker (worker.ts)          ┌─────────────────────┐  │
│  poll(2s) ──► outbox ──► HTTP ──► cliente B2B    │  │
│               (PENDING)   POST    (https://...)   │  │
│                           HMAC                     │  │
│  retry(5x) ──► backoff ──► falha ──► DLQ         │  │
└──────────────────────────────────────────────────────┘
```

### Componentes

1. **Tabela `webhook_outbox`**: persistência atômica do evento dentro da transação `changeStatus`. Cada linha contém `id` (UUID), `event_type`, `payload` (JSON snapshot), `status`, `attempts`, `next_retry_at`, timestamps.

2. **Módulo `src/modules/webhooks/`**: segue o padrão estrutural do projeto — controller, service, repository, routes, schemas. Responsável pelo CRUD de configuração de webhooks (URL, secret, filtro de status) e endpoint de entregas históricas.

3. **Worker (`src/worker.ts`)**: processo separado com PrismaClient próprio. Loop de polling (`status = PENDING AND next_retry_at <= NOW()`) em batch de 10. Envia HTTP POST com timeout de 10s, headers de segurança e payload JSON.

4. **DLQ (`webhook_dead_letter`)**: tabela separada para eventos que esgotaram as 5 tentativas. Reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN).

5. **Integração no `changeStatus`**: função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação existente, recebendo o `Prisma.TransactionClient` como parâmetro — mesmo padrão de `debitStock` e `replenishStock`.

### Decisões de Segurança

- HMAC-SHA256 sobre o payload para integridade + autenticidade
- Secret única de 256 bits por endpoint, gerada na criação e rotacionável com grace period de 24h
- Apenas URLs `https://` aceitas (validação Zod)
- Headers outbound: `X-Signature`, `X-Timestamp`, `X-Event-Id`, `X-Webhook-Id`
- Limite de payload: 64KB (erro `WEBHOOK_PAYLOAD_TOO_LARGE` se exceder)

### Decisões de Resiliência

- 5 tentativas com backoff: 1min → 5min → 30min → 2h → 12h (~15h de janela total)
- Timeout HTTP: 10 segundos por chamada
- Polling: 2 segundos de intervalo (latência máxima ~12s, dentro da meta de <10s declarada pelos clientes)
- Garantia: at-least-once — cliente deduplica por `X-Event-Id`

## Alternativas Consideradas

### Alternativa 1: Disparo síncrono no `changeStatus`
- **O que é**: Fazer a chamada HTTP ao webhook do cliente dentro da mesma transação do `OrderService.changeStatus`, sem outbox ou worker.
- **Por que foi descartada**: Viola resiliência. Um cliente offline travaria a mudança de status para todos os pedidos. Além disso, se a chamada HTTP falhar, não há como dar rollback apenas do webhook — a transação inteira teria que abortar, o que é incorreto do ponto de vista de negócio (a mudança de status é um fato consumado).
- **Trade-off**: Simplicidade de implementação vs acoplamento fatal entre transação de negócio e disponibilidade de terceiros. O custo do acoplamento supera o ganho de simplicidade.

### Alternativa 2: Redis Streams como broker de mensagens
- **O que é**: Publicar eventos num stream Redis e ter um consumer group processando as entregas.
- **Por que foi descartada**: Exigiria subir e operar Redis Cluster como infraestrutura adicional. Para um time pequeno e apenas 3 clientes B2B no horizonte imediato, a complexidade operacional não se justifica. O MySQL já está provisionado, monitorado e o time o conhece.
- **Trade-off**: Maior throughput e menor latência vs custo operacional de manter infraestrutura adicional. Para o volume atual e projetado, o custo vence.

## Questões em Aberto

### 1. Rate limiting de envio por cliente
Se um cliente tiver 50 pedidos mudando de status em 1 minuto, o worker disparará 50 chamadas HTTP em rápida sucessão. Isso pode sobrecarregar o endpoint do cliente ou disparar proteções anti-DDoS do lado dele. A reunião decidiu **observar e decidir depois** se será necessário implementar rate limiting configurável por webhook. Não há decisão de implementação neste momento.

**Encaminhamento**: monitorar métricas de `webhook_deliveries_per_endpoint_per_minute` nos primeiros 30 dias pós-deploy. Se picos >20/min forem frequentes, abrir RFC específica para rate limiting.

### 2. Notificação proativa ao cliente sobre falhas
O Marcos sugeriu enviar e-mail ao cliente quando um webhook falhar 3 vezes seguidas. A reunião decidiu que **e-mail está fora do escopo desta fase**, mas é um candidato natural para a próxima iteração.

**Encaminhamento**: incluir no backlog da fase 2, após coleta de métricas de taxa de falha por cliente. A decisão de triggering (3 falhas consecutivas? 5? % de falha na janela?) depende dos dados reais.

## Impacto e Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Worker cair e eventos acumularem na outbox | Média | Alto — clientes param de receber notificações | Healthcheck do worker + alerta em métrica `webhook_outbox_lag` (eventos pendentes > N por > T minutos) |
| Cliente cadastrar URL maliciosa ou interna | Baixa | Crítico — SSRF | Validação de URL (apenas https), bloqueio de IPs privados/reservados, revisão de segurança pela Sofia antes do deploy |
| Secret vazar em log da aplicação | Baixa | Alto — atacante pode forjar webhooks | Pino já redacta campos sensíveis (`*.token`, `*.secret`). Revisão adicional dos logs do worker para garantir que headers e secrets não vazem |
| Tabela outbox crescer sem limite | Média | Médio — degradação de performance do polling | Arquivação de linhas `SENT` com >30 dias (job agendado, fora do escopo da feature). Métrica de `webhook_outbox_size` |
| Migração de schema quebrar transação do `changeStatus` | Baixa | Alto — rollback de todas as mudanças de status | A inserção na outbox é parte da transação — se falhar, a transação inteira dá rollback. Testes de integração cobrem o caminho feliz e o de falha |

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md) — decisão de usar outbox em vez de síncrono ou Redis
- [ADR-002: Política de Retry com Backoff Exponencial e DLQ](./adrs/ADR-002-retry-backoff-dlq.md) — 5 tentativas, backoff progressivo, DLQ separada
- [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-003-hmac-sha256-autenticacao.md) — assinatura, secret única, rotação com grace period
- [ADR-004: Garantia At-Least-Once com X-Event-Id](./adrs/ADR-004-at-least-once-event-id.md) — deduplicação delegada ao cliente
- [ADR-005: Worker em Processo Separado com Polling](./adrs/ADR-005-worker-processo-separado-polling.md) — processo isolado, 2s de polling
- [ADR-006: Reuso dos Padrões Existentes do Projeto](./adrs/ADR-006-reuso-padroes-projeto-existente.md) — AppError, Pino, Zod, middleware, estrutura modular
