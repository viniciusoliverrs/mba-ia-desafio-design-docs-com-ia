# ADR-002: Worker em Processo Separado Consumindo a Outbox via Polling

- **ID de Rastreabilidade:** `ADR-002`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (PM)

## Status

Aceito

## Contexto

Com a adoção do padrão Transactional Outbox no MySQL ([ADR-001](./ADR-001-outbox-no-mysql.md)), os eventos de mudança de status de pedidos são persistidos na tabela `webhook_outbox` com status pendente. É necessário definir a arquitetura de execução do consumidor (*worker*) responsável por ler esses registros, realizar as chamadas HTTP `POST` para os endpoints dos clientes e atualizar o estado de entrega.

Os requisitos que restringem essa decisão são:
1. O tempo entre a mudança de status e o recebimento pelo cliente deve ser inferior a 10 segundos para atender à percepção de "tempo real" dos clientes B2B (`[09:02] Marcos`).
2. O processo da API HTTP (`src/server.ts`) não deve ter sua estabilidade, ciclo de deploy/reinicialização ou pool de execução afetados pelo processamento em background dos webhooks (`[09:11] Diego`).
3. A solução deve manter a mesma stack tecnológica do projeto (Node.js, TypeScript, Prisma Client em `src/config/database.ts`, logger Pino em `src/shared/logger/index.ts`) (`[09:11] Bruno`, `[09:11] Diego`).

## Decisão

O processamento da tabela `webhook_outbox` será realizado por um **Worker executando em processo Node.js separado** da API HTTP, operando em **loop de polling a cada 2 segundos** (`[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego`).

Especificações arquiteturais da decisão:
1. **Novo Entry-Point Independente (`src/worker.ts`):** Assim como `src/server.ts` inicializa o servidor Express (`src/app.ts`), será criado o arquivo `src/worker.ts` acionado por um script dedicado `"worker"` no `package.json` (`npm run worker`), mantendo a lógica de domínio encapsulada em `src/modules/webhooks/webhook.worker.ts` (`[09:11] Larissa`, `[09:28] Bruno`).
2. **Instância Própria do `PrismaClient`:** Por rodar em outro processo do sistema operacional, o worker instanciará seu próprio `PrismaClient` (reutilizando a fábrica `createPrismaClient()` de `src/config/database.ts` e a mesma variável `DATABASE_URL` validada em `src/config/env.ts`) (`[09:11] Bruno`, `[09:30] Bruno`).
3. **Ciclo de Polling de 2 Segundos em Lotes Pequenos:** A cada 2 segundos, o worker consulta na tabela `webhook_outbox` os eventos pendentes mais antigos (ou com `next_retry_at <= NOW()`), ordenados por `created_at ASC` em lotes pequenos (*small batch*), marca-os como em processamento, realiza os disparos HTTP com timeout de 10 segundos (`[09:42] Diego`) e atualiza o resultado (`[09:08] Diego`, `[09:09] Diego`).
4. **Single-Worker Inicial e Ordenação Implícita:** Na primeira versão, será executada uma única instância do worker (*single-worker*), garantindo ordenação implícita por `order_id` através da leitura sequencial por `created_at` (`[09:12] Diego`, `[09:13] Larissa`).

## Alternativas Consideradas

### Alternativa 1: Executar o Worker dentro do mesmo processo da API (`src/server.ts`) (`ADR-002-ALT-01`)
- **Descrição:** Iniciar um `setInterval` dentro do próprio bootstrap da API Express em `src/server.ts`.
- **Motivo do Descarte:** Descartada (`[09:11] Diego`). Se a instância da API reiniciar ou escalar independentemente, o ciclo do worker seria interrompido no meio de envios HTTP em andamento ou multiplicado sem controle de concorrência, além de disputar CPU/event-loop e conexões de banco com as requisições síncronas da API REST.

### Alternativa 2: Triggers Nativas no Banco de Dados MySQL para Acionamento Reativo (`ADR-002-ALT-02`)
- **Descrição:** Utilizar triggers do MySQL na tabela `webhook_outbox` para notificar reativamente o worker a cada inserção, evitando polling.
- **Motivo do Descarte:** Descartada por limitação técnica do SGBD (`[09:09] Bruno`, `[09:09] Diego`). Diferente do PostgreSQL (que possui `LISTEN/NOTIFY` nativo), o MySQL não possui mecanismo nativo para notificar processos externos a partir de uma trigger (que apenas executa SQL). Improvisar chamadas externas via UDF/arquivos a partir do MySQL seria frágil e anti-padrão, enquanto o polling de 2 segundos atende com folga o SLA de 10 segundos (`[09:09] Diego`).

### Alternativa 3: Múltiplos Workers Paralelos com Particionamento ou Lock Pessimista no Lançamento (`ADR-002-ALT-03`)
- **Descrição:** Construir desde o dia 1 um cluster de múltiplos workers concorrentes com particionamento por `order_id` ou *row locking* complexo para preservar a ordem dos eventos.
- **Motivo do Descarte:** Adiada para evolução futura (`[09:12] Diego`, `[09:13] Diego`, `[09:13] Larissa`). Para a volumetria atual dos clientes B2B, um *single-worker* atende perfeitamente e garante a entrega ordenada por `created_at` sem complexidade adicional; clientes tampouco exigiram garantia de ordenação global entre pedidos distintos (`[09:14] Marcos`).

## Consequências

### Positivas
- **Atendimento Folgado ao SLA de Negócio:** O intervalo de polling de 2 segundos garante latência mínima de detecção <= 2s, muito abaixo do limite de 10 segundos acordado com os clientes (`[09:02] Marcos`, `[09:10] Larissa`).
- **Isolamento Operacional e Resiliência:** Falhas, lentidão de rede externa ou reinicializações da API (`src/server.ts`) não derrubam o worker (`src/worker.ts`) e vice-versa (`[09:11] Diego`).
- **Ordenação Simples no MVP:** Com *single-worker* lendo por `created_at`, transições rápidas de um mesmo pedido (ex.: `PAID` -> `PROCESSING` -> `SHIPPED`) são processadas na ordem exata em que ocorreram (`[09:12] Diego`).

### Negativas e Trade-offs
- **Consultas Periódicas ao Banco (Trade-off aceito):** O polling a cada 2 segundos gera `SELECT` contínuo no MySQL mesmo em períodos ociosos, trade-off aceito pela simplicidade arquitetural e mitigado pelo índice composto em `(status, created_at)` na `webhook_outbox` (`[09:08] Diego`, `[09:09] Diego`).
- **Limitação Conhecida de Escala Horizontal e Ordenação:** A garantia de ordenação por `order_id` depende de haver apenas um worker ativo; caso seja necessário escalar para múltiplos workers no futuro, será obrigatório introduzir particionamento por `order_id` ou lock pessimista (`[09:12] Diego`, `[09:13] Larissa`).
- **Consumo de Conexões no MySQL:** Como o worker instancia seu próprio `PrismaClient` em processo separado, o dimensionamento do pool de conexões do MySQL deve considerar o processo da API mais o processo do worker (`[09:29] Diego`, `[09:30] Bruno`).

