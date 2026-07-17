# ADR-004: Garantia At-Least-Once com X-Event-Id

## Status

Accepted

## Contexto

O sistema precisa garantir que cada evento de mudança de status seja entregue ao cliente. No entanto, cenários de falha na entrega (timeout de rede, crash do worker entre o envio HTTP e a marcação como `SENT` na outbox) podem fazer com que o worker retente um evento que já foi entregue na tentativa anterior. O cliente, portanto, pode receber o mesmo evento mais de uma vez.

Existem três garantias clássicas de entrega em sistemas distribuídos:
- **At-most-once**: o evento é enviado no máximo uma vez; se falhar, perdeu
- **At-least-once**: o evento é enviado pelo menos uma vez; pode haver duplicatas
- **Exactly-once**: o evento é entregue exatamente uma vez, sem duplicatas

## Decisão

Adotaremos a garantia **at-least-once** com **deduplicação delegada ao cliente via header `X-Event-Id`**.

**Mecanismo**:
- Cada evento inserido na `webhook_outbox` recebe um UUID v4 gerado no momento da inserção
- O header `X-Event-Id: <uuid>` é enviado em toda tentativa de entrega
- Se o cliente receber o mesmo `X-Event-Id` mais de uma vez, ele deve ignorar as duplicatas (idempotência do lado cliente)
- O header é documentado explicitamente no portal de desenvolvedor como parte do contrato de integração

**Por que não exactly-once**:
Exactly-once exigiria coordenação bidirecional — o cliente precisaria confirmar recebimento com acknowledgement antes de o worker marcar como `SENT`, e o worker precisaria de um protocolo de two-phase commit ou idempotência via locking distribuído. Isso introduz latência de confirmação, complexidade de implementação e novos pontos de falha.

## Alternativas Consideradas

### Alternativa 1: At-most-once (sem retry)
- **Trade-off**: extrema simplicidade — envia uma vez e esquece. Mas falhas de rede ou downtime do cliente resultam em perda permanente do evento, o que é inaceitável para notificações de pedidos (ex: cliente nunca saberia que o pedido foi enviado). Descartado por não atender ao requisito de confiabilidade.

### Alternativa 2: Exactly-once com protocolo de acknowledgement
- **Trade-off**: elimina duplicatas do lado cliente, mas exige que o worker aguarde confirmação explícita (ACK) antes de marcar como entregue. Se o ACK não chegar (timeout), o worker não sabe se o evento foi entregue ou não, recriando o problema. Adiciona latência, estado distribuído e complexidade de protocolo. Descartado por custo-benefício desfavorável dado que at-least-once com event_id resolve o caso prático (cliente deduplica).

## Consequências

### Positivas
- Padrão de mercado adotado por Stripe, GitHub, Twilio — clientes B2B conhecem o modelo
- Simplicidade de implementação: worker não precisa gerenciar estado de confirmação externa
- UUID por evento permite rastreabilidade ponta a ponta: mesmo event_id na outbox, nos logs do worker e nos logs do cliente

### Negativas
- Transfere responsabilidade de deduplicação para o cliente — clientes que não implementarem idempotência processarão o mesmo evento duas vezes
- Em cenários de múltiplos workers no futuro, a garantia de ordering por `order_id` é perdida — duplicatas podem chegar fora de ordem
- Não há mecanismo para o cliente sinalizar "já recebi e processei, pode parar de retentar" — o worker sempre esgota as 5 tentativas antes de ir pra DLQ
