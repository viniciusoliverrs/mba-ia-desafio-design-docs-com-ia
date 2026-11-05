# ADR-006: Reuso Integral dos Padrões Arquiteturais, de Erros, Validação e Logging do Código Existente

- **ID de Rastreabilidade:** `ADR-006`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Bruno (Engenheiro Pleno de Pedidos), Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Sofia (Engenheira de Segurança)

## Status

Aceito

## Contexto

O repositório do Order Management System (OMS) possui uma arquitetura modular bem estabelecida em Node.js + TypeScript + Express + Prisma + Zod + Pino (`package.json`). Ao inspecionar o código-fonte existente, observam-se padrões consistentes em todos os domínios (`src/modules/auth`, `src/modules/users`, `src/modules/customers`, `src/modules/products`, `src/modules/orders`):

1. ** Organização Modular:** Cada domínio possui `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts`, sendo instanciado em `buildControllers` (`src/app.ts`) e montado em `/api/v1` via `buildApiRouter` (`src/routes/index.ts`).
2. **Hierarquia Centralizada de Erros:** Todos os erros de negócio e HTTP herdam de `AppError` (`src/shared/errors/app-error.ts`) ou das subclasses em `src/shared/errors/http-errors.ts` (`BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`), utilizando códigos em `UPPER_SNAKE_CASE` (como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`) e sendo convertidos em JSON padronizado pelo `errorMiddleware` (`src/middlewares/error.middleware.ts`).
3. **Autenticação, Autorização e Validação:** As rotas utilizam os middlewares `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`) e o middleware `validate` baseado em Zod (`src/middlewares/validate.middleware.ts`), além do helper `paginated` (`src/shared/http/response.ts`).
4. **Logging Estruturado e Banco de Dados:** O projeto utiliza o logger Pino configurado com redaction de dados sensíveis em `src/shared/logger/index.ts` e chaves primárias `UUID` (`@db.Char(36)`) no `prisma/schema.prisma`.

Introduzir bibliotecas concorrentes, formatos distintos de resposta de erro ou uma estrutura de diretórios divergente para a feature de Webhooks fragmentaria a manutenção do sistema (`[09:27] Bruno`, `[09:30] Larissa`).

## Decisão

O módulo de Webhooks **reutilizará integralmente os padrões arquiteturais, componentes compartilhados e convenções já existentes no código base**, sem adicionar novas bibliotecas de logging, validação ou tratamento de erros (`[09:27] Bruno`, `[09:29] Bruno`, `[09:30] Larissa`).

Especificamente:
1. **Estrutura de Módulo em `src/modules/webhooks/`:** Segalará exatamente a divisão em `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` e `webhook.worker.ts`, registrado em `src/app.ts` e `src/routes/index.ts` (`[09:27] Bruno`, `[09:28] Bruno`).
2. **Padronização de Erros com Prefixo `WEBHOOK_`:** Todos os erros específicos do domínio estenderão `AppError` / subclasses de `src/shared/errors/http-errors.ts` e utilizarão obrigatoriamente o prefixo `WEBHOOK_` no `errorCode` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`), sendo capturados automaticamente pelo `errorMiddleware` (`src/middlewares/error.middleware.ts`) sem necessidade de alterá-lo (`[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`).
3. **Validação com Zod (`src/middlewares/validate.middleware.ts`):** Todos os payloads, parâmetros de rota e query strings (incluindo a validação de `https://` obrigatório nas URLs de webhook) serão declarados em `webhook.schemas.ts` com Zod e validados pelo middleware `validate` (`[09:23] Sofia`, `[09:30] Larissa`).
4. **Autenticação JWT e RBAC (`src/middlewares/auth.middleware.ts`):** As rotas de CRUD de webhooks e listagem de deliveries usarão o middleware `authenticate` (permitindo qualquer role autenticada, recebendo `customerId` via body/query), enquanto a rota `POST /api/v1/admin/webhooks/dead-letter/:id/replay` combinará `authenticate` e `requireRole('ADMIN')` (`[09:32] Larissa`, `[09:36] Larissa`, `[09:37] Sofia`).
5. **Logger Pino (`src/shared/logger/index.ts`) e UUIDs (`prisma/schema.prisma`):** Tanto a API quanto o worker (`src/worker.ts`) utilizarão o logger Pino existente (estendendo `redactPaths` para ocultar `*.secret` e `*.previousSecret`) e todas as novas tabelas no `prisma/schema.prisma` usarão `id String @id @default(uuid()) @db.Char(36)` (`[09:29] Bruno`, `[09:51] Larissa`).

## Alternativas Consideradas

### Alternativa 1: Criar Microsserviço Separado ou Arquitetura Interna Independente para Webhooks (`ADR-006-ALT-01`)
- **Descrição:** Implementar o gerenciamento e disparo de webhooks em um repositório/serviço independente ou com stack própria de tratamento de erros e logs.
- **Motivo do Descarte:** Descartada (`[09:07] Diego`, `[09:27] Bruno`, `[09:30] Larissa`). Impediria o uso direto da transação local do Prisma em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) e geraria duplicação desnecessária de infraestrutura, autenticação e modelos para uma equipe enxuta.

### Alternativa 2: Extrair `customerId` Implicitamente do Token JWT Atual (`ADR-006-ALT-02`)
- **Descrição:** Assumir que o payload do JWT (`AuthUser` em `src/middlewares/auth.middleware.ts`) já carrega o `customerId` do cliente B2B (`[09:31] Marcos`).
- **Motivo do Descarte:** Descartada após verificação do código existente (`[09:32] Bruno`, `[09:32] Larissa`). O token JWT atual do OMS autentica usuários da tabela `users` (`id`, `email`, `role: 'ADMIN' | 'OPERATOR'`), e não a entidade `Customer` diretamente. Portanto, mantém-se o `authenticate` padrão e passa-se `customerId` explicitamente no body ou query/path dos endpoints de webhook (`[09:32] Larissa`).

## Consequências

### Positivas
- **Consistência de Contratos HTTP e DX:** Clientes da API recebem o mesmo envelope de erro (`{ error: { code, message, details } }`) gerado por `src/middlewares/error.middleware.ts` e o mesmo formato de paginação (`{ data, pagination }`) gerado por `src/shared/http/response.ts` (`[09:28] Bruno`, `[09:29] Bruno`).
- **Menor Esforço e Risco de Implementação:** Reaproveitar `AppError`, `validate`, `authenticate`, `requireRole`, `logger` e `PrismaClient` permite entregar toda a feature dentro da estimativa de 3 sprints (`[09:30] Larissa`, `[09:46] Larissa`).

### Negativas e Trade-offs
- **Permissão Ampla Inicial no CRUD de Webhooks (Trade-off aceito):** Como o JWT atual possui apenas as roles `ADMIN` e `OPERATOR` e não vincula um `User` a um `Customer` específico no schema atual (`prisma/schema.prisma`), qualquer usuário autenticado pode operar o CRUD de webhooks informando o `customerId` (`[09:36] Marcos`, `[09:37] Sofia`). Esse ponto foi aceito para o lançamento inicial e registrado para endurecimento futuro de controle de acesso (`[09:37] Sofia`).

