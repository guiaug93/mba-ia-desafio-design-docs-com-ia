# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status

Accepted

## Contexto

Os webhooks transportam dados de pedidos (status, valores, IDs de cliente) para endpoints externos fora da infraestrutura da aplicação. O cliente receptor precisa ter garantia de que:

1. A requisição realmente veio da plataforma OMS (autenticidade de origem)
2. O payload não foi adulterado em trânsito (integridade)
3. A requisição não é um replay attack (freshness)

Sem essas garantias, um atacante poderia forjar notificações de status falso para sistemas do cliente, causando decisões de negócio incorretas (ex: liberar mercadoria baseado em notificação forjada de pagamento).

O sistema já possui autenticação JWT para chamadas inbound (`src/middlewares/auth.middleware.ts`), mas isso cobre apenas requisições que entram na API. Para chamadas outbound (nossos webhooks para o cliente), precisamos de um mecanismo que o cliente possa verificar sem depender da nossa infraestrutura.

## Decisão

Usaremos **HMAC-SHA256 sobre o corpo do request**, com **secret única por endpoint de webhook cadastrado**.

**Mecanismo**:
- Na criação do webhook (`POST /api/v1/webhooks`), o sistema gera uma secret aleatória de 256 bits (codificada em hex) e a armazena na tabela de configuração
- No envio, o worker calcula `HMAC-SHA256(payload_string, secret)` e envia no header `X-Signature: sha256=<hex_digest>`
- O cliente recalcula o HMAC com a secret que recebeu e compara com o header recebido

**Rotação de secrets**:
- Endpoint `POST /api/v1/webhooks/:id/rotate-secret` gera nova secret
- A secret antiga permanece válida por 24 horas (grace period) em paralelo com a nova
- Após 24 horas, a secret antiga é invalidada
- Durante o grace period, o worker tenta primeiro com a secret nova; se falhar, retenta com a antiga

**Headers de segurança outbound**:
| Header | Conteúdo | Propósito |
|--------|----------|-----------|
| `X-Signature` | `sha256=<hmac_hex>` | Integridade + autenticidade |
| `X-Timestamp` | ISO 8601 do momento do envio | Detecção de replay attack |
| `X-Event-Id` | UUID v4 do evento | Idempotência do lado cliente |

**Validação de URL**:
- Apenas URLs `https://` são aceitas no cadastro — `http://` é rejeitado com erro `WEBHOOK_INVALID_URL` via schema Zod

## Alternativas Consideradas

### Alternativa 1: Secret global única para toda a plataforma
- **Trade-off**: simples de implementar, mas o vazamento de uma única secret comprometeria todos os webhooks de todos os clientes. Descartado por risco de segurança inaceitável para integração B2B.

### Alternativa 2: JWT outbound assinado com chave privada da plataforma
- **Trade-off**: elimina a necessidade de o cliente armazenar secret compartilhada, mas introduz par de chaves assimétricas e dependência de bibliotecas JWT do lado cliente. HMAC é mais simples, universal e igualmente seguro para cenários de API-to-API com segredo pré-compartilhado. Descartado por complexidade desnecessária dado o perfil dos clientes (B2B com capacidade técnica para HMAC).

## Consequências

### Positivas
- Padrão de mercado (Stripe, GitHub, Shopify) — clientes B2B já conhecem o modelo
- Secret por endpoint limita o raio de impacto de um vazamento
- Rotação com grace period elimina janela de downtime durante migração de secrets

### Negativas
- Exige que o cliente armazene e proteja a secret — responsabilidade compartilhada
- Grace period de 24h implica que, em caso de vazamento confirmado, o cliente fica vulnerável por até 24h até a secret antiga expirar
- Complexidade adicional no worker: lógica de fallback entre secret nova e antiga durante o grace period
