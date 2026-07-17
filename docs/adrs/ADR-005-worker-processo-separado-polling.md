# ADR-005: Worker em Processo Separado com Polling

## Status

Accepted

## Contexto

O processamento da outbox — leitura de eventos pendentes e disparo de chamadas HTTP — não pode ocorrer no mesmo processo da API REST. Se a API reiniciar (deploy, crash, scale down), o worker não pode ser perdido junto. Além disso, o processamento de webhooks não deve competir por CPU e event loop com as requisições da API.

A aplicação atual inicia via `src/server.ts` com `app.listen()`. É necessário um ponto de entrada separado para o worker, reutilizando a mesma stack e a mesma conexão de banco, mas em processo isolado.

## Decisão

O worker será implementado como um **processo Node.js separado** com **polling em loop a cada 2 segundos** sobre a tabela `webhook_outbox`.

**Entry point**:
- Arquivo `src/worker.ts` com script npm `"worker": "tsx --env-file=.env src/worker.ts"`
- O worker inicia um PrismaClient próprio (instância separada, mesma `DATABASE_URL`) e um loop infinito com `setInterval` ou `while (true) + sleep`

**Algoritmo de polling**:
1. Busca eventos com `status = 'PENDING' AND next_retry_at <= NOW()` ordenados por `created_at ASC`, limitados a um batch de 10
2. Para cada evento, marca como `PROCESSING`, envia HTTP POST ao endpoint do webhook
3. Em caso de sucesso (2xx), marca como `SENT`
4. Em caso de falha, incrementa `attempts`, calcula `next_retry_at` com base na progressão de backoff (ADR-002) e volta para `PENDING`
5. Se `attempts >= 5`, move para `webhook_dead_letter` e marca como `FAILED`

**Intervalo de polling**: 2 segundos fixos, atendendo ao requisito de latência máxima de ~10 segundos (2s polling + ~10s timeout HTTP ≈ 12s no pior caso, dentro da tolerância declarada pelos clientes).

**Timeout HTTP**: 10 segundos por chamada (ADR-002), usando `AbortController` ou timeout nativo do `fetch`.

**Graceful shutdown**: o worker escuta SIGINT/SIGTERM, finaliza o batch atual e desconecta o Prisma antes de sair.

## Alternativas Consideradas

### Alternativa 1: Worker no mesmo processo da API (thread/fork interno)
- **Trade-off**: simplifica deploy (um único processo), mas acopla ciclo de vida — deploy da API mata o worker. Se o worker travar por loop infinito ou memory leak, a API cai junto. Descartado por exigência de independência de ciclo de vida.

### Alternativa 2: Trigger de banco (MySQL trigger + UDF)
- **Trade-off**: eliminaria o polling, reagindo imediatamente a INSERTs na outbox. Mas MySQL não tem mecanismo nativo de notificação para processos externos (como o `NOTIFY/LISTEN` do PostgreSQL). Implementar via trigger que escreve em arquivo ou chama um endpoint interno seria frágil e de difícil debug. Descartado por falta de suporte nativo e complexidade de operação.

## Consequências

### Positivas
- Independência de ciclo de vida: deploy da API não afeta o worker, e vice-versa
- Isolamento de recursos: CPU e memória do worker não competem com a API
- Simplicidade: polling é trivial de implementar, testar e debugar
- Mesma stack: worker reutiliza Prisma, logger Pino, classes de erro e padrões do projeto

### Negativas
- Latência mínima de 2 segundos (pior caso: evento inserido logo após o ciclo de polling)
- Batch limitado a 10 eventos por ciclo — se houver pico de eventos, o processamento serializa
- Custo operacional: um processo adicional para monitorar, logar e manter no deploy
- Single-worker por design: escalar horizontalmente exige lock distribuído ou particionamento
