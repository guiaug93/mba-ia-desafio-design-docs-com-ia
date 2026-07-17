# ADR-001: Padrão Outbox no MySQL

## Status

Accepted

## Contexto

A aplicação precisa notificar sistemas externos (clientes B2B) sobre mudanças de status de pedidos via webhooks. O método `changeStatus` em `src/modules/orders/order.service.ts` já executa uma transação atômica que atualiza a tabela `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos. Adicionar uma chamada HTTP síncrona dentro da mesma transação traria dois problemas:

1. **Acoplamento de latência**: um cliente lento ou offline bloquearia a mudança de status para outros pedidos, degradando toda a API.
2. **Inconsistência transacional**: se o cliente estiver fora do ar, não há rollback da mudança de status já persistida — a transação SQL teria que abortar por falha externa, o que viola o princípio de independência de camadas.

## Decisão

Usaremos o **padrão Transactional Outbox com MySQL**.

Dentro da mesma transação `prisma.$transaction` do `changeStatus`, uma nova linha será inserida numa tabela `webhook_outbox` contendo o evento serializado. Um processo worker separado fará polling dessa tabela para disparar as chamadas HTTP aos clientes.

**Mecanismo**: se a transação principal comitar, o evento está registrado; se der rollback, o evento desaparece junto. A garantia de atomicidade é herdada do banco relacional existente.

**Detalhes de implementação**:
- Tabela `webhook_outbox` com colunas: `id` (UUID), `event_type`, `payload` (JSON), `status` (PENDING, PROCESSING, SENT, FAILED), `attempts`, `next_retry_at`, `created_at`
- Índices em `(status, created_at)` para consulta eficiente de eventos pendentes
- Snapshot do payload capturado no momento da inserção (não renderizado no envio)
- Arquivação de linhas entregues após 30 dias (fora do escopo da feature)

## Alternativas Consideradas

### Alternativa 1: Disparo síncrono dentro do `changeStatus`
- **Trade-off**: simplicidade de implementação, mas cria acoplamento fatal entre transação de negócio e disponibilidade de sistemas externos. Descartado por violar resiliência mínima exigida por sistemas B2B.

### Alternativa 2: Redis Streams como broker de mensagens
- **Trade-off**: maior reatividade e throughput, mas exigiria subir e manter um Redis Cluster. Para um time pequeno e com apenas 3 clientes B2B no horizonte, é overengineering. Descartado por complexidade operacional desproporcional ao escopo atual.

## Consequências

### Positivas
- Atomicidade entre mudança de status e registro do evento garantida pelo banco existente
- Zero infraestrutura adicional — reutiliza o MySQL já provisionado
- Simplicidade operacional: mesmo banco, mesma stack, mesmo Prisma client

### Negativas
- Latência de entrega limitada pelo intervalo de polling (2 segundos no pior caso)
- Maior carga no banco principal, já que a tabela outbox compete por recursos com as tabelas de negócio
- Single-worker por design — escalar para múltiplos workers no futuro exige particionamento ou lock pessimista
