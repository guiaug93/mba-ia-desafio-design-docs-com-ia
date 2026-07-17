# Sistema de Webhooks de Notificação de Pedidos — Design Docs com IA

## Sobre o desafio

Este desafio consiste em transformar a transcrição de uma reunião técnica de ~55 minutos em um pacote completo de design docs para uma feature de **Sistema de Webhooks de Notificação de Pedidos**, usando IA como ferramenta principal de produção. O cenário: uma empresa opera um Order Management System (OMS) em Node.js + TypeScript com MySQL e Prisma, e precisa adicionar notificações outbound para clientes B2B — uma lacuna que não existia no sistema original.

A entrega é puramente documental: o código fonte (`src/`, `prisma/`, `tests/`) serve exclusivamente como contexto e referência. Os artefatos produzidos — PRD, RFC, FDD, ADRs e Tracker de rastreabilidade — formam um pacote coeso em que cada documento opera numa altura diferente (produto, arquitetura, implementação, governança), sem duplicação de conteúdo e com rastreabilidade total à transcrição ou ao código.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|------------|-------|
| **Claude (OpenCode)** | Ferramenta principal. Usado em modo agente para leitura e análise do código fonte completo, interpretação da transcrição, estruturação dos documentos e geração do conteúdo final. O agente teve acesso ao repositório inteiro e produziu todos os documentos em sequência. |
| **Skills especializadas** | Skills como `pytest-advanced`, `python`, `ui-design` foram carregadas para fornecer contexto sobre padrões de documentação e desenvolvimento, embora o projeto seja TypeScript. A skill `context7-mcp` auxiliou na consulta de referências de bibliotecas (Prisma, Express, Zod) quando necessário. |
| **Wiki de referência (LLM Wiki)** | Repositório `/home/acer/Documentos/projects/mba-full-cycle/llm-wiki` com artigos sobre PRD, FDD, ADR, SDD e Design Docs com IA — usado como referência de formato e boas práticas para cada tipo de documento. |

---

## Workflow adotado

O trabalho seguiu a ordem de execução sugerida no enunciado, com ênfase em construir as decisões (ADRs) antes de derivar os documentos que dependem delas:

1. **Contextualização com IA (30 min)**: Leitura completa de todos os arquivos do repositório — `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md`, `docs/context.md`, wiki de referência. O agente mapeou a estrutura modular, a máquina de estados de pedidos (`order.status.ts`), o padrão de transações (`order.service.ts`), as classes de erro (`AppError` e derivadas), os middlewares (`auth`, `error`, `validate`, `request-logger`), o logger Pino e o schema Prisma.

2. **ADRs primeiro (1h)**: As 6 decisões principais da transcrição foram isoladas e cada uma virou um ADR independente no formato MADR. As decisões secundárias (snapshot de payload, formato de headers) foram incorporadas nos ADRs principais ou no FDD.

3. **RFC (45 min)**: A proposta técnica consolidou as decisões dos ADRs num documento de arquitetura, focado em "o que propomos e por quê", sem descer ao detalhe de implementação. As alternativas descartadas e questões em aberto da reunião encontraram lugar natural aqui.

4. **FDD (1h30)**: O documento mais técnico e extenso. Detalhou fluxos, contratos de API (7 endpoints com exemplos de request/response), matriz de erros `WEBHOOK_*`, estratégias de resiliência, observabilidade e a seção obrigatória de integração com o código existente (7 arquivos referenciados com descrição de como integrar).

5. **PRD (45 min)**: Produzido por último entre os grandes documentos. Com RFC, FDD e ADRs já prontos, o PRD funcionou como consolidação de alto nível — foco em problema, público, métricas e critérios de aceitação.

6. **Tracker (40 min)**: Mapeamento de 82 itens à origem na transcrição (com timestamp e falante) ou no código (com caminho de arquivo). Construído em paralelo com a revisão dos documentos.

7. **README do processo (20 min)**: Escrito ao final, documentando a jornada completa.

---

## Prompts customizados

### Prompt 1: Análise e extração de decisões da transcrição

Este prompt foi usado para extrair sistematicamente todas as decisões, requisitos funcionais, itens descartados e questões em aberto da transcrição, com timestamps:

```
Você é um engenheiro de software revisando a transcrição de uma reunião
técnica sobre um Sistema de Webhooks de Notificação de Pedidos.

Analise a transcrição abaixo e extraia, em formato de lista, APENAS itens
com origem explícita na fala dos participantes. Para cada item, indique
o timestamp [hh:mm] e o nome do falante.

Categorize os itens em:
1. DECISÕES FECHADAS (decisões arquiteturais ou de implementação confirmadas)
2. REQUISITOS FUNCIONAIS (o que o sistema deve fazer)
3. REQUISITOS NÃO FUNCIONAIS (restrições de performance, segurança, etc.)
4. DESCARTADO/ADIADO (itens explicitamente excluídos ou postergados)
5. QUESTÕES EM ABERTO (pontos levantados e não decididos)

IMPORTANTE: Não invente, não infira, não generalize. Se um item não tem
origem textual clara na transcrição, não o inclua.

Transcrição:
[COLAR TRANSCRICAO AQUI]
```

Este prompt garantiu que a filtragem entre "mencionado" e "decidido" fosse precisa, evitando que discussões exploratórias virassem requisitos.

### Prompt 2: Validação cruzada do FDD contra o código

Este prompt foi usado para verificar que todos os caminhos de arquivo e padrões mencionados no FDD realmente existem no código:

```
Você é um revisor técnico. Compare o documento FDD abaixo com o código
fonte do repositório. Verifique:

1. Todo caminho de arquivo mencionado no FDD (ex: src/modules/orders/order.service.ts)
   EXISTE no repositório? Liste os que existem e os que não existem.

2. Toda classe, função ou padrão referenciado (ex: AppError, canTransition,
   errorMiddleware, buildControllers) EXISTE no código com o nome exato citado?

3. O padrão de código descrito (ex: "prisma.$transaction", "Prisma.TransactionClient")
   corresponde ao uso real no código?

4. Há alguma contradição entre o que o FDD descreve e o que o código implementa?

Retorne APENAS problemas encontrados. Se tudo estiver correto, responda
"Validação OK — sem divergências".

FDD:
[COLAR FDD AQUI]
```

Este prompt foi fundamental para garantir que a seção "Integração com o sistema existente" não referenciasse arquivos inexistentes ou padrões que não existem no código.

---

## Iterações e ajustes

### Iteração 1: ADR-006 excessivamente genérico

**Problema**: Na primeira versão do ADR-006 (Reuso dos Padrões Existentes), o documento dizia apenas "devemos reutilizar AppError, Pino e a estrutura modular" sem especificar COMO cada padrão seria aplicado concretamente ao módulo de webhooks.

**Correção**: Reescrevi o ADR incluindo uma tabela de 10 padrões mapeados com localização exata no código (arquivo + linha) e uma seção "Aplicação concreta dos padrões ao módulo" com 9 pontos numerados descrevendo como cada padrão se materializa no novo módulo (ex: "Novas classes estendendo AppError com prefixo WEBHOOK_*", "Função publishWebhookEvent(tx, ...) seguindo o mesmo padrão de debitStock").

### Iteração 2: FDD com seção de integração superficial

**Problema**: A primeira versão do FDD referenciava apenas 2 arquivos na seção "Integração com o sistema existente" e descrevia a integração de forma vaga ("o módulo de webhooks se integrará ao order.service").

**Correção**: Expandi para 7 subseções (12.1 a 12.7), cada uma referenciando um arquivo real com linha específica, descrevendo o que existe hoje naquele arquivo, como a integração será feita e incluindo trechos de código ilustrativos. Por exemplo, a seção 12.1 mostra exatamente em que ponto da transação do `changeStatus` (entre linha 167 e 169) a chamada `publishWebhookEvent` será inserida.

### Iteração 3: Tracker inicial com baixa cobertura de TRANSCRICAO

**Problema**: O primeiro tracker tinha apenas ~40 itens, com muitos requisitos do PRD sem correspondência na transcrição — sinal de que a IA havia inferido requisitos não discutidos na reunião.

**Correção**: Refiz a varredura item por item. Removi 3 requisitos que não tinham origem na transcrição (ex: "webhook com autenticação mTLS" — nunca mencionado). Adicionei timestamps para todos os itens restantes, resultando em 82 linhas com >90% de cobertura de TRANSCRICAO + CODIGO. As 5 linhas CODIGO referenciam arquivos verificados como existentes.

---

## Como navegar a entrega

### Estrutura de arquivos entregues

```
.
├── README.md                                   ← Este arquivo (processo de produção)
├── TRANSCRICAO.md                              ← Transcrição original (não alterada)
├── docs/
│   ├── context.md                              ← Enunciado do desafio (não alterado)
│   ├── PRD.md                                  ← Product Requirements Document
│   ├── RFC.md                                  ← Request for Comments (proposta técnica)
│   ├── FDD.md                                  ← Feature Design Document (implementação)
│   ├── TRACKER.md                              ← Rastreabilidade (82 itens)
│   └── adrs/
│       ├── README.md                           ← Índice dos ADRs
│       ├── ADR-001-outbox-no-mysql.md          ← Decisão: Outbox em MySQL
│       ├── ADR-002-retry-backoff-dlq.md        ← Decisão: Retry + DLQ
│       ├── ADR-003-hmac-sha256-autenticacao.md ← Decisão: HMAC-SHA256
│       ├── ADR-004-at-least-once-event-id.md   ← Decisão: At-least-once
│       ├── ADR-005-worker-processo-separado-polling.md ← Decisão: Worker separado
│       └── ADR-006-reuso-padroes-projeto-existente.md  ← Decisão: Reuso de padrões
```

### Ordem sugerida de leitura

1. **`TRANSCRICAO.md`** — Comece pela transcrição para entender o domínio e o tom da reunião (~10 min)
2. **`docs/PRD.md`** — Entenda o problema de negócio, público-alvo e métricas de sucesso (~8 min)
3. **`docs/RFC.md`** — Veja a proposta técnica, alternativas descartadas e questões em aberto (~10 min)
4. **`docs/adrs/ADR-001-outbox-no-mysql.md` até `ADR-006-...`** — Leia cada ADR para entender as decisões arquiteturais e seus trade-offs (~15 min no total)
5. **`docs/FDD.md`** — Mergulhe nos detalhes de implementação, contratos de API e integração com o código (~20 min)
6. **`docs/TRACKER.md`** — Use como referência cruzada para verificar a origem de cada item (~5 min de consulta)
7. **`README.md`** (este arquivo) — Entenda o processo de produção e as ferramentas utilizadas (~5 min)

### Critérios de aceite — checklist de autoavaliação

#### PRD (`docs/PRD.md`)
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] Identifica 12 requisitos funcionais (mínimo 8)
- [x] Inclui 6 objetivos com métricas e metas quantitativas
- [x] Seção "Fora de escopo" lista 4 itens descartados/adiados
- [x] Seção "Riscos" inclui 5 riscos com probabilidade, impacto e mitigação

#### RFC (`docs/RFC.md`)
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] Seção "Alternativas consideradas" lista 2 alternativas com trade-off
- [x] Seção "Questões em aberto" lista 2 pontos não decididos
- [x] Referencia 6 ADRs com links

#### FDD (`docs/FDD.md`)
- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias
- [x] Seção "Contratos públicos" inclui 7 endpoints HTTP com exemplos
- [x] Matriz de erros usa prefixo `WEBHOOK_` (10 códigos)
- [x] Seção "Integração com o sistema existente" referencia 7 caminhos de arquivo reais
- [x] Seção "Observabilidade" cita métricas, logs e tracing

#### ADRs (`docs/adrs/ADR-NNN-*.md`)
- [x] Pasta contém 6 arquivos no formato correto
- [x] Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- [x] Conjunto cobre todas as 6 decisões principais
- [x] ADR-006 referencia explicitamente 10 padrões do código existente com localização

#### Tracker (`docs/TRACKER.md`)
- [x] Arquivo existe com tabela no formato definido
- [x] 82 linhas cobrindo itens de todos os documentos
- [x] ~85% das linhas têm Fonte = TRANSCRICAO com timestamp válido
- [x] 7 linhas têm Fonte = CODIGO com caminho de arquivo real

#### README (`README.md`)
- [x] Contém todas as seções obrigatórias
- [x] Lista 3 ferramentas de IA utilizadas
- [x] Mostra 2 prompts customizados em blocos de código
- [x] Descreve 3 iterações/ajustes concretos

#### Consistência geral
- [x] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código
- [x] Todos os arquivos de código mencionados nos documentos existem no repositório

---

> **Referência ao enunciado original**: O enunciado completo deste desafio está disponível em [docs/context.md](docs/context.md). O repositório base encontra-se em [github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
