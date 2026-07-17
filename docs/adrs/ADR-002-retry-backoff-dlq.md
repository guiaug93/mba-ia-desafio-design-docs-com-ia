# ADR-002: Política de Retry com Backoff Exponencial e DLQ

## Status

Accepted

## Contexto

Os webhooks são entregues via chamadas HTTP para endpoints controlados pelos clientes B2B. Esses endpoints podem estar indisponíveis por indisponibilidade temporária de rede, manutenção planejada ou falha na aplicação do cliente. A aplicação precisa lidar com essas falhas sem perder eventos e sem sobrecarregar o endpoint do cliente com tentativas excessivas.

O sistema expõe internamente uma tabela `webhook_outbox` com status `PENDING`, `PROCESSING`, `SENT` e `FAILED`. Eventos que falham repetidamente precisam de destinação final clara para evitar acumulação infinita na tabela principal e permitir diagnóstico.

## Decisão

Adotaremos uma política de **5 tentativas com backoff exponencial progressivo**, seguida de movimentação para uma **Dead Letter Queue (DLQ) persistida em tabela separada**.

**Progressão de backoff**:
| Tentativa | Intervalo após falha |
|-----------|---------------------|
| 1ª → 2ª   | 1 minuto            |
| 2ª → 3ª   | 5 minutos           |
| 3ª → 4ª   | 30 minutos          |
| 4ª → 5ª   | 2 horas             |
| 5ª → DLQ  | 12 horas            |

Janela total de cobertura: ~15 horas entre primeira falha e última tentativa.

**DLQ**:
- Tabela separada `webhook_dead_letter` com: `id`, `outbox_id`, `webhook_id`, `payload`, `failure_reason`, `last_status_code`, `last_error_body`, `failed_at`, `created_at`
- A outbox original é marcada como `FAILED` e o registro é movido para a DLQ
- Reprocessamento manual via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento na outbox como `PENDING` e zera o contador de tentativas
- Endpoint de replay exige role `ADMIN` (via `requireRole` existente em `src/middlewares/auth.middleware.ts`) e loga o usuário que executou a ação para auditoria

## Alternativas Consideradas

### Alternativa 1: 3 tentativas apenas
- **Trade-off**: mais agressivo, mas cobre apenas ~36 minutos de indisponibilidade. Clientes com manutenções planejadas de 1-2 horas perderiam eventos. Descartado por experiência prévia com indisponibilidades reais de clientes.

### Alternativa 2: Retry indefinido com backoff
- **Trade-off**: elimina a necessidade de DLQ, mas eventos de clientes que descontinuaram o serviço ficariam eternamente na outbox, poluindo a tabela principal e consumindo recursos de polling. Descartado por falta de fechamento e acúmulo.

## Consequências

### Positivas
- Cobertura de 15 horas cobre manutenções planejadas e falhas de infraestrutura de terceiros
- DLQ em tabela separada mantém a outbox principal enxuta para polling eficiente
- Reprocessamento manual administrativo permite recuperação controlada sem mexer no banco diretamente

### Negativas
- Latência total de entrega pode chegar a 15 horas em cenário de falha persistente
- DLQ exige intervenção manual para reprocessamento — não há disparo automático
- Complexidade adicional de uma segunda tabela e lógica de movimentação entre outbox e DLQ
