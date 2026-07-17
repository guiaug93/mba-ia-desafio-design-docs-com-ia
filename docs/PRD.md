# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O Sistema de Webhooks de Notificação de Pedidos é uma feature que permite que clientes B2B da plataforma OMS recebam notificações automáticas via HTTP sempre que o status de um pedido mudar, eliminando a necessidade de polling manual via `GET /orders`. A feature foi concebida a partir de demanda formal de três clientes — Atlas Comercial, MaxDistribuição e Nova Cargo — e é considerada crítica para retenção: a Atlas Comercial sinalizou possível migração para concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

A solução opera como webhooks outbound: o sistema OMS envia chamadas HTTP POST para endpoints configurados pelos clientes, autenticadas via HMAC-SHA256, sempre que o status de um pedido associado ao cliente transiciona na máquina de estados (`PENDING` → `PAID` → `PROCESSING` → `SHIPPED` → `DELIVERED` / `CANCELLED`).

## Problema e Motivação

### Situação atual
Clientes B2B que operam com a plataforma OMS dependem do status dos pedidos para seus fluxos internos — logística programa coleta quando o pedido muda para `SHIPPED`, faturamento emite nota quando muda para `PAID`, SAC acompanha quando entra em `DELIVERED`. Hoje, a única forma de obter essa informação é via polling no endpoint `GET /api/v1/orders`, que os clientes consultam periodicamente (a cada 30-60 segundos) para detectar alterações.

### Problemas identificados
1. **Ineficiência operacional**: cada cliente faz dezenas de chamadas por minuto, sendo que a grande maioria retorna sem alterações — desperdício de recursos de ambos os lados
2. **Alta latência percebida**: entre a mudança real de status e a detecção pelo polling, passam-se até 60 segundos, o que impacta fluxos sensíveis ao tempo (ex: coleta de mercadoria)
3. **Custo de integração**: cada novo cliente B2B precisa implementar lógica de polling, paginação, detecção de mudanças e tolerância a falhas do lado deles
4. **Risco de churn**: a Atlas Comercial formalizou que a ausência de notificações em tempo real é motivo para considerar migração para concorrente

### Motivação
Entregar um mecanismo de notificação push resolve os quatro problemas simultaneamente: reduz chamadas desnecessárias, entrega latência <10 segundos, simplifica a integração do lado cliente (receber POST é mais simples que implementar polling) e elimina o risco de churn.

## Público-Alvo e Cenários de Uso

### Público-alvo primário
Clientes B2B da plataforma OMS que possuem sistemas internos (ERP, WMS, TMS) que consomem dados de status de pedidos para automação de processos.

**Personas técnicas**:
- **Desenvolvedor de integração do cliente**: configura o endpoint de webhook no painel da OMS, implementa o receptor de webhooks do lado do cliente e gerencia a secret de autenticação
- **Operador de logística do cliente**: consome indiretamente — as notificações disparam automações no sistema interno (ex: ordem de coleta ao receber `SHIPPED`)

### Público-alvo secundário
- **Administradores da plataforma OMS**: monitoram entregas de webhooks, reprocessam eventos da DLQ e gerenciam configurações de clientes
- **Time de suporte**: utiliza o histórico de entregas para diagnosticar problemas de integração reportados por clientes

### Cenários de uso

| Cenário | Descrição |
|---------|-----------|
| **Notificação de pagamento** | Pedido transiciona de `PENDING` para `PAID` → sistema de faturamento do cliente emite NF automaticamente |
| **Notificação de envio** | Pedido transiciona para `SHIPPED` → WMS do cliente programa coleta e gera tracking |
| **Notificação de entrega** | Pedido transiciona para `DELIVERED` → CRM do cliente dispara pesquisa de satisfação |
| **Filtro seletivo** | Cliente configura webhook apenas para `SHIPPED` e `DELIVERED`, ignorando estágios intermediários |
| **Rotação de secret** | Cliente detecta vazamento de secret, rotaciona e atualiza sistemas internos dentro do grace period de 24h |
| **Diagnóstico de falha** | Cliente não recebeu notificação → administrador verifica histórico de entregas e identifica falha de rede |
| **Reprocessamento de DLQ** | Após correção do endpoint do cliente, administrador faz replay dos eventos que falharam |

## Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Medição |
|----------|---------|------|---------|
| Reduzir latência de notificação | Latência p95 entre `changeStatus` e chegada no cliente | < 10 segundos | Timestamp da outbox vs timestamp do recebimento (log do cliente, quando disponível) |
| Eliminar polling dos clientes | Redução de chamadas `GET /orders` por cliente | ≥ 80% de redução em 30 dias pós-adoção | Métricas de API discriminadas por customer_id |
| Reter clientes B2B | Churn de clientes B2B por falta de notificação | 0% nos 6 meses seguintes ao lançamento | CRM / feedback de account managers |
| Garantir entregabilidade | Taxa de eventos entregues com sucesso na primeira tentativa | ≥ 95% | `webhook_deliveries_total{status="success"}` / total |
| Não degradar a API principal | Latência p95 do `PATCH /orders/:id/status` | Sem aumento > 5ms | Comparação pré e pós-deploy |
| Cobertura de falhas transientes | % de eventos recuperados via retry | ≥ 90% dos eventos que falharam na 1ª tentativa são entregues até a 5ª | `webhook_deliveries_total` agrupado por `attempt` |

## Escopo

### Incluso (MVP — 3 sprints)
- **RF-01**: CRUD de configuração de webhooks (criar, listar, editar, excluir) por cliente, autenticado via JWT
- **RF-02**: Geração automática de secret de 256 bits na criação do webhook, com rotação via endpoint dedicado e grace period de 24h
- **RF-03**: Integração no fluxo de mudança de status — inserção atômica de evento na outbox dentro da transação do `changeStatus`
- **RF-04**: Worker em processo separado que faz polling da outbox a cada 2 segundos e dispara chamadas HTTP POST
- **RF-05**: Assinatura HMAC-SHA256 sobre o payload, com secret por endpoint
- **RF-06**: Retry automático com backoff exponencial: 5 tentativas (1m, 5m, 30m, 2h, 12h)
- **RF-07**: Dead Letter Queue (DLQ) com reprocessamento manual via endpoint admin (role ADMIN)
- **RF-08**: Headers de idempotência (`X-Event-Id`) para deduplicação do lado cliente
- **RF-09**: Histórico de entregas por webhook (últimos N eventos, status, duração, response)
- **RF-10**: Filtro de eventos por status na configuração do webhook — cliente escolhe quais transições receber
- **RF-11**: Logs estruturados e métricas de entrega para observabilidade
- **RF-12**: Validação de URL (HTTPS obrigatório, bloqueio de IPs privados) com erro `WEBHOOK_INVALID_URL`

### Fora de Escopo

1. **Envio de e-mail ao cliente sobre falhas de webhook** — discutido na reunião ([09:37] Marcos), explicitamente adiado por Larissa para "próxima fase, depois que a gente medir o impacto". A feature atual notifica apenas via webhook; notificação proativa de falha (e-mail, SMS, dashboard) fica para iteração futura.

2. **Dashboard visual de webhooks** — mencionado por Marcos ([09:40]) e descartado por Larissa: "Painel é projeto separado do time de frontend". A entrega atual é apenas de endpoints de API; a interface visual é responsabilidade de outro time.

3. **Rate limiting de envio por cliente** — levantado por Diego ([09:39]) como preocupação, mas explicitamente adiado: "A gente observa e implementa se virar problema". Fica como ponto de observação pós-deploy.

4. **Arquivação automática de eventos antigos** — Diego mencionou arquivar linhas entregues após 30 dias ([09:08]), mas foi classificado como "fora do escopo dessa feature". Será tratado como tarefa de manutenção separada.

## Requisitos Funcionais

### RF-01: Cadastro de Webhook
**Descrição**: O cliente (via usuário autenticado) pode cadastrar um endpoint de webhook informando URL e lista de status de interesse. O sistema gera automaticamente uma secret de 256 bits e a retorna na resposta.
**Prioridade**: P0 (essencial)
**Fluxo principal**: `POST /api/v1/webhooks` → validação → geração de secret → persistência → resposta 201 com secret
**Erros**: `WEBHOOK_INVALID_URL` (URL não HTTPS), `WEBHOOK_DUPLICATE_URL` (mesmo customer + mesma URL)

### RF-02: Listagem de Webhooks
**Descrição**: Cliente pode listar seus webhooks cadastrados, com filtro por customer_id e paginação. A resposta nunca inclui a secret completa — apenas os 8 primeiros caracteres como preview.
**Prioridade**: P1 (importante)
**Fluxo principal**: `GET /api/v1/webhooks?customer_id=<uuid>&page=1&page_size=20`

### RF-03: Edição de Webhook
**Descrição**: Cliente pode alterar URL, lista de eventos ou status ativo/inativo de um webhook existente. A secret não é alterada — para isso, há endpoint específico de rotação.
**Prioridade**: P1
**Fluxo principal**: `PATCH /api/v1/webhooks/:id`

### RF-04: Exclusão de Webhook
**Descrição**: Cliente pode remover uma configuração de webhook. Eventos já pendentes na outbox para este webhook não são cancelados — apenas novas mudanças de status não gerarão eventos.
**Prioridade**: P1
**Fluxo principal**: `DELETE /api/v1/webhooks/:id` → 204

### RF-05: Rotação de Secret
**Descrição**: Cliente pode solicitar nova secret para um webhook. A secret anterior permanece válida por 24 horas (grace period). Durante esse período, o worker tenta primeiro com a secret nova e, se falhar, retenta com a antiga.
**Prioridade**: P1
**Fluxo principal**: `POST /api/v1/webhooks/:id/rotate-secret` → nova secret + `previous_secret_valid_until`

### RF-06: Notificação de Mudança de Status
**Descrição**: Quando um pedido tem seu status alterado via `PATCH /orders/:id/status`, o sistema insere um evento na outbox para cada webhook ativo do customer que esteja configurado para receber aquele status. A inserção é atômica com a transação de mudança de status.
**Prioridade**: P0

### RF-07: Entrega de Webhook
**Descrição**: O worker lê eventos pendentes da outbox, monta requisição HTTP POST com payload JSON e headers de segurança, assina com HMAC-SHA256 e envia ao endpoint do cliente. Em caso de sucesso (2xx), registra a entrega e marca como `SENT`.
**Prioridade**: P0

### RF-08: Retry Automático
**Descrição**: Se a entrega falhar (timeout, erro de rede, HTTP não-2xx), o worker agenda nova tentativa seguindo a progressão de backoff: 1min, 5min, 30min, 2h, 12h. Total de 5 tentativas.
**Prioridade**: P0

### RF-09: Dead Letter Queue
**Descrição**: Após 5 tentativas sem sucesso, o evento é movido para a tabela `webhook_dead_letter` com metadados de falha. O registro original na outbox é marcado como `FAILED`.
**Prioridade**: P1

### RF-10: Reprocessamento de DLQ
**Descrição**: Administrador (role ADMIN) pode reprocessar um evento da DLQ, reinserindo-o na outbox como `PENDING` com tentativas zeradas. A ação é logada com userId do administrador.
**Prioridade**: P1
**Fluxo principal**: `POST /admin/webhooks/dead-letter/:id/replay` → novo `outbox_id`

### RF-11: Histórico de Entregas
**Descrição**: Cliente pode consultar o histórico de entregas de um webhook específico, com informações de sucesso/falha, status code HTTP, duração e timestamp.
**Prioridade**: P2
**Fluxo principal**: `GET /api/v1/webhooks/:id/deliveries?page=1&page_size=20`

### RF-12: Filtro de Eventos por Status
**Descrição**: Na configuração do webhook, o cliente especifica quais status de pedido deseja receber (ex: apenas `SHIPPED` e `DELIVERED`). O filtro é aplicado na inserção da outbox — se nenhum webhook do customer quer aquele status, o evento não é gerado.
**Prioridade**: P1

## Requisitos Não Funcionais

### Performance
- **NF-01**: Inserção na outbox não pode aumentar a latência do `PATCH /orders/:id/status` em mais de 5ms (p95)
- **NF-02**: Worker deve processar batch de 10 eventos em no máximo 12 segundos
- **NF-03**: Latência máxima de entrega (p95): <10 segundos entre `changeStatus` e chegada ao cliente

### Disponibilidade
- **NF-04**: Worker deve sobreviver a reinicializações e recuperar eventos em estado `PROCESSING` travados há mais de 5 minutos
- **NF-05**: Disponibilidade do serviço de webhooks: 99.5% (alinhada com SLAs de sistemas internos)

### Segurança
- **NF-06**: Secrets nunca devem aparecer em logs ou respostas de API de listagem
- **NF-07**: Apenas URLs HTTPS são aceitas para endpoints de webhook
- **NF-08**: SSRF prevenido — bloqueio de IPs privados (RFC 1918), loopback e link-local
- **NF-09**: Repasse de secret expirada após grace period de 24h
- **NF-10**: Auditoria de replay de DLQ (userId + timestamp)

### Observabilidade
- **NF-11**: Métricas de entrega (sucesso, falha, duração, backlog) disponíveis via logs estruturados
- **NF-12**: Todo erro do worker gera log nível ERROR com `eventId` e `webhookId`
- **NF-13**: `X-Event-Id` propagado da outbox ao header HTTP e logs para rastreabilidade ponta a ponta

### Limites
- **NF-14**: Payload máximo de evento: 64KB. Eventos que excederem geram `WEBHOOK_PAYLOAD_TOO_LARGE` e não são inseridos na outbox
- **NF-15**: Timeout HTTP por chamada: 10 segundos
- **NF-16**: Batch máximo por ciclo de polling: 10 eventos

## Decisões e Trade-offs Principais

| Decisão | Trade-off |
|----------|-----------|
| **Outbox no MySQL em vez de Redis Streams** | Simplicidade operacional (zero infra nova) vs menor latência e maior throughput. Para 3 clientes B2B, a simplicidade vence. |
| **At-least-once em vez de exactly-once** | Responsabilidade de deduplicação transferida ao cliente vs complexidade de protocolo de confirmação bidirecional. Padrão de mercado adotado por Stripe e GitHub. |
| **Worker single-thread em vez de multi-worker** | Ordenação implícita por `order_id` preservada, mas throughput limitado e sem alta disponibilidade do worker. Para o volume atual, single-worker atende. |
| **Polling de 2s em vez de triggers de banco** | Latência máxima de 2s no pior caso vs complexidade de notificar processo externo a partir do MySQL (que não tem `LISTEN/NOTIFY`). |
| **5 retries (15h) em vez de 3 (~36min)** | Cobertura de manutenções planejadas longas vs demora para falhar definitivamente. Experiência prévia com indisponibilidades de clientes justifica a janela maior. |

## Dependências

### Técnicas
- MySQL 8.0 (já provisionado via Docker Compose)
- Prisma ORM 5.22 (já em uso — `src/config/database.ts`)
- Express 4.21 + middlewares existentes (`authenticate`, `requireRole`, `validate`, `errorMiddleware`)
- Logger Pino 9.5 (`src/shared/logger/index.ts`)
- Classes de erro `AppError` e derivadas (`src/shared/errors/`)
- `crypto` nativo do Node.js 20 (HMAC-SHA256) — nenhuma dependência nova

### Organizacionais
- Revisão de segurança pela Sofia antes do deploy (2 dias úteis reservados)
- Documentação no portal de desenvolvedor para clientes (responsabilidade do Marcos)
- Coordenação com time de frontend para eventual dashboard (fora do escopo desta feature)

### Externas
- Clientes B2B precisam implementar endpoint receptor de webhooks do lado deles, com verificação HMAC-SHA256 e deduplicação por `X-Event-Id`

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **Cliente não implementar deduplicação por `X-Event-Id`** | Média | Alto — processamento duplicado de eventos pode causar efeitos colaterais nos sistemas do cliente (ex: dupla emissão de NF) | Documentação explícita e destacada no portal de desenvolvedor. Incluir exemplo de código de verificação de idempotência na doc. Marcos responsável por comunicar aos clientes. |
| **Worker crash silencioso (eventos acumulam na outbox sem processamento)** | Média | Alto — clientes param de receber notificações sem alerta visível | Healthcheck externo monitorando métrica `webhook_outbox_pending_count`. Alerta se >100 eventos pendentes por >5 minutos. Docker restart policy `unless-stopped`. |
| **Vazamento de secret em logs do cliente** | Média | Médio — atacante pode forjar notificações para o sistema do cliente | Rotação de secret com grace period de 24h permite resposta rápida. Documentação orienta cliente a proteger secrets como credenciais. |
| **SSRF: cliente cadastra URL apontando para rede interna** | Baixa | Crítico — worker faria chamadas para dentro da infraestrutura | Validação de URL com bloqueio de IPs privados (RFC 1918), loopback e link-local. Revisão de segurança pela Sofia antes do deploy. |
| **Tabela outbox crescer sem limite** | Média | Médio — degradação de performance do polling | Arquivação de registros `SENT` com >30 dias (job agendado fora do escopo da feature). Métrica de tamanho da tabela com alerta. |

## Critérios de Aceitação

### Negócio
- [ ] Cliente consegue cadastrar, listar, editar e excluir webhooks via API
- [ ] Cliente recebe notificação HTTP em até 10 segundos após mudança de status do pedido
- [ ] Cliente consegue verificar autenticidade da notificação via HMAC-SHA256
- [ ] Cliente consegue deduplicar notificações pelo header `X-Event-Id`
- [ ] Cliente consegue rotacionar a secret do webhook sem downtime
- [ ] Cliente consegue filtrar quais status de pedido deseja receber
- [ ] Administrador consegue reprocessar eventos da DLQ
- [ ] Cliente consegue consultar histórico de entregas do seu webhook

### Técnico
- [ ] Nenhuma dependência nova adicionada ao `package.json`
- [ ] Nenhuma alteração de comportamento nos endpoints existentes
- [ ] Cobertura de testes ≥ 90% no módulo de webhooks
- [ ] Revisão de segurança aprovada pela Sofia
- [ ] Métricas e logs disponíveis para monitoramento

## Estratégia de Testes e Validação

### Testes Unitários
- Cálculo de HMAC-SHA256 com vetores de teste conhecidos
- Progressão de backoff (valores corretos para cada `attempts`)
- Filtro de eventos (webhook configurado para `SHIPPED` não recebe `PAID`)
- Validação de URL (HTTPS obrigatório, rejeição de IPs privados)
- Geração de secret (256 bits, entropia suficiente)
- Classes de erro `WEBHOOK_*` com códigos e status HTTP corretos

### Testes de Integração
- Inserção na outbox dentro da transação do `changeStatus` (verificar que evento some no rollback)
- Worker lê da outbox e envia HTTP (com mock de endpoint do cliente)
- Retry automático (mock retorna 500, verificar incremento de `attempts` e `next_retry_at`)
- Movimentação para DLQ após 5 falhas
- Reprocessamento de DLQ via endpoint admin
- Rotação de secret — nova secret funciona, antiga válida por 24h, expira após
- Histórico de entregas paginado

### Testes Ponta a Ponta
- Fluxo completo: criar webhook → mudar status do pedido → worker envia → cliente recebe → verificar HMAC → verificar idempotência
- Cenário de falha: endpoint offline → retry → sucesso na 3ª tentativa → verificar histórico
- Cenário de DLQ: 5 falhas → evento na DLQ → admin faz replay → cliente recebe

### Testes de Segurança
- Penetration test: tentar cadastrar URL com IP privado
- Penetration test: tentar acessar endpoint de replay de DLQ sem role ADMIN
- Verificação de que secrets não aparecem em logs (grep nos logs de dev)
- Verificação de que secrets não aparecem em respostas de API de listagem

### Testes de Performance
- Medir latência do `PATCH /orders/:id/status` com e sem outbox
- Soak test do worker: 1000 eventos em fila, verificar throughput e uso de memória
- Simular cliente lento (delay de 8s) e verificar que não bloqueia outros eventos no batch
