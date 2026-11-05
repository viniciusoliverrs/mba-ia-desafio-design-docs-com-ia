# Architectural Decision Records (ADRs)

Este diretório armazena os ADRs (*Architecture Decision Records*) do **Sistema de Webhooks de Notificação de Pedidos** do Order Management System (OMS), elaborados no formato MADR e rastreados em [`docs/TRACKER.md`](../TRACKER.md).

## Índice de Decisões

| ADR | Título | Status | Resumo da Decisão |
| --- | --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Transactional Outbox no MySQL para Eventos de Webhook | Aceito | Inserção atômica de eventos com snapshot JSON na tabela `webhook_outbox` dentro da mesma transação Prisma de `OrderService.changeStatus`. |
| [ADR-002](./ADR-002-worker-em-processo-separado-em-polling.md) | Worker em Processo Separado Consumindo a Outbox via Polling | Aceito | Processo Node.js independente (`src/worker.ts`) com `PrismaClient` próprio fazendo polling a cada 2 segundos em modo *single-worker*. |
| [ADR-003](./ADR-003-politica-de-retry-com-backoff-e-dlq.md) | Política de Retry com Backoff Exponencial e DLQ em Tabela Separada | Aceito | Até 5 tentativas (`1m`, `5m`, `30m`, `2h`, `12h`), timeout HTTP de 10s, tabela `webhook_dead_letter` separada e replay manual restrito a `ADMIN`. |
| [ADR-004](./ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | Autenticação de Webhooks com HMAC-SHA256, Secret por Endpoint e Rotação | Aceito | Assinatura `HMAC-SHA256` em `X-Signature`, HTTPS obrigatório, secret exclusiva por endpoint e rotação com *grace period* de 24 horas. |
| [ADR-005](./ADR-005-garantia-at-least-once-com-x-event-id.md) | Semântica de Entrega At-Least-Once e Idempotência via `X-Event-Id` | Aceito | Entrega *at-least-once* com UUID imutável por evento (`X-Event-Id` / `event_id`) para deduplicação idempotente no receptor. |
| [ADR-006](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso Integral dos Padrões Arquiteturais, de Erros, Validação e Logging | Aceito | Reuso da estrutura modular (`src/modules/webhooks`), `AppError` com prefixo `WEBHOOK_*`, `errorMiddleware`, Zod `validate`, `requireRole` e Pino. |
