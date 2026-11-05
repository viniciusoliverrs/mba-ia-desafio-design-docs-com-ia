# Tracker de Rastreabilidade

Este documento mapeia cada item (decisões arquiteturais, alternativas descartadas, questões em aberto, objetivos, escopo/exclusões, requisitos funcionais, requisitos não funcionais, contratos HTTP, códigos de erro, riscos e pontos de integração) registrado nos documentos do pacote (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-001` a `ADR-006`) à sua respectiva origem literal em `TRANSCRICAO.md` (`TRANSCRICAO`) ou no código-fonte da aplicação (`CODIGO`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Adoção do padrão Transactional Outbox no MySQL existente na mesma transação SQL de mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de chamada HTTP síncrona dentro de OrderService.changeStatus por travar transação e acoplar disponibilidade externa | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de Redis Streams / Redis Cluster por overengineering e necessidade de subir infraestrutura adicional para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Descarte de armazenar apenas order_id na outbox e renderizar o payload somente na hora do envio | TRANSCRICAO | [09:52] Larissa |
| ADR-001-TRD-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Aceitação de escrita adicional no MySQL mitigada por índices em status e created_at na tabela webhook_outbox | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md | Decisão | Execução do worker em processo Node.js separado (src/worker.ts) realizando polling na outbox a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md | Alternativa Descartada | Descarte de rodar o worker na mesma instância/processo da API para não perder o worker em reinicializações da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md | Alternativa Descartada | Descarte de triggers do banco MySQL para acionar o worker pela ausência de LISTEN/NOTIFY nativo no MySQL | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md | Alternativa Descartada | Adiamento de múltiplos workers paralelos com particionamento por order_id ou lock pessimista para fase futura | TRANSCRICAO | [09:13] Diego |
| ADR-002-TRD-01 | docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md | Trade-off | Garantia de ordenação apenas por order_id e enquanto operar em single-worker lendo por created_at | TRANSCRICAO | [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Decisão | Política de 5 tentativas com backoff exponencial (1m/5m/30m/2h/12h), tabela separada webhook_dead_letter e replay manual ADMIN | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Alternativa Descartada | Descarte de retry indefinido para evitar eventos pendurados para sempre caso o cliente desative o servidor | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Alternativa Descartada | Descarte de limite curto de 3 tentativas (30m) por não cobrir janelas de manutenção de clientes de 2 horas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md | Alternativa Descartada | Descarte de apenas marcar como failed na própria outbox em favor de tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura HMAC-SHA256 em X-Signature, secret única por endpoint, TLS obrigatório e rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Descarte de secret global única da plataforma para evitar comprometimento total caso uma chave vaze | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Descarte de secret estática sem rotação ou sem grace period de 24h para evitar downtime em troca de chave | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-03 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Descarte de permitir URLs http:// sem TLS, recusando com erro de validação no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia de entrega at-least-once enviando UUID único no header X-Event-Id para deduplicação pelo cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Alternativa Descartada | Descarte de garantia exactly-once por exigir coordenação complexa de ambos os lados da integração | TRANSCRICAO | [09:25] Diego |
| ADR-005-TRD-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Transferência da responsabilidade de deduplicação para o cliente documentada no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso integral de AppError (prefixo WEBHOOK_), Pino, errorMiddleware, schemas Zod e estrutura modular em src/modules/webhooks | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa Descartada | Descarte de extrair customerId implicitamente do JWT atual pois o token representa usuário operador/admin e não Customer | TRANSCRICAO | [09:32] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão de Código | Estrutura do payload JWT atual contendo sub, email e role (ADMIN ou OPERATOR) sem vínculo direto com Customer | CODIGO | src/middlewares/auth.middleware.ts |
| RFC-CTX-01 | docs/RFC.md | Contexto | Demanda formal de Atlas Comercial, MaxDistribuição e Nova Cargo para substituir polling em GET /orders com SLA < 10s | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Escopo estritamente outbound (envio de eventos do OMS para o cliente) e transação atômica com mudança de status | TRANSCRICAO | [09:03] Sofia |
| RFC-PROP-01 | docs/RFC.md | Proposta Arquitetural | Inserção transacional na webhook_outbox filtrada por status assinado com snapshot do payload e limite de 64KB | TRANSCRICAO | [09:34] Bruno |
| RFC-PROP-02 | docs/RFC.md | Proposta Arquitetural | Worker independente em src/worker.ts com PrismaClient próprio em polling de 2s e ordenação implícita single-worker | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | docs/RFC.md | Proposta Arquitetural | Timeout HTTP de 10s, 5 retries com backoff (1m/5m/30m/2h/12h), tabela webhook_dead_letter e replay ADMIN | TRANSCRICAO | [09:17] Diego |
| RFC-PROP-04 | docs/RFC.md | Proposta Arquitetural | Assinatura HMAC-SHA256 por endpoint, rotação com grace period de 24h, X-Event-Id e CRUD autenticado via JWT | TRANSCRICAO | [09:20] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono dentro do OrderService descartado para não travar a transação SQL nem dar rollback por queda do cliente | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / Redis Cluster descartado por overengineering frente ao uso do MySQL já existente | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Triggers no MySQL descartadas por não notificarem processos externos nativamente como o LISTEN/NOTIFY do Postgres | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido ou limite curto de 3 tentativas descartados em favor de 5 tentativas cobrindo quase 15 horas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Marcação de failed na própria outbox descartada em favor da tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Semântica exactly-once descartada pela complexidade de coordenação bilateral em favor de at-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio para clientes em picos de mudança de status (ex: 50 pedidos/min) mantido em observação | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Definição futura de particionamento por order_id ou lock pessimista caso o worker escale horizontalmente | TRANSCRICAO | [09:12] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Endurecimento futuro de permissões (roles) nos endpoints de CRUD de configuração de webhook | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Automação de arquivamento de eventos entregues na webhook_outbox após 30 dias deixada fora do escopo atual | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Alerta por e-mail ao cliente após 3 falhas consecutivas adiado para próxima fase após medição de impacto | TRANSCRICAO | [09:37] Larissa |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Atomicidade transacional entre mudança de status em OrderService.changeStatus e inserção na webhook_outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Processamento assíncrono em processo separado (src/worker.ts) via polling de 2 segundos para cumprir SLA < 10s | TRANSCRICAO | [09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Timeout HTTP de 10s, 5 tentativas de retry (1m/5m/30m/2h/12h), histórico de 100 entregas e DLQ com replay ADMIN | TRANSCRICAO | [09:42] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | Segurança com HTTPS obrigatório, assinatura HMAC-SHA256 por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo Técnico | Entrega at-least-once com X-Event-Id, snapshot renderizado na inserção e limite máximo de 64KB por payload | TRANSCRICAO | [09:24] Diego |
| FDD-OUT-01 | docs/FDD.md | Fora de Escopo | Exclusão de recebimento de webhooks enviados pelos clientes (fluxo exclusivamente outbound) | TRANSCRICAO | [09:02] Marcos |
| FDD-OUT-02 | docs/FDD.md | Fora de Escopo | Exclusão de envio de e-mail automático ao cliente em caso de falhas seguidas no webhook | TRANSCRICAO | [09:37] Larissa |
| FDD-OUT-03 | docs/FDD.md | Fora de Escopo | Exclusão de dashboard ou painel visual de gerenciamento de webhooks (entrega restrita a endpoints de API) | TRANSCRICAO | [09:40] Larissa |
| FDD-OUT-04 | docs/FDD.md | Fora de Escopo | Exclusão de rate limiting de saída no lançamento inicial | TRANSCRICAO | [09:39] Diego |
| FDD-OUT-05 | docs/FDD.md | Fora de Escopo | Exclusão da rotina de arquivamento de registros entregues na outbox após 30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-01 | docs/FDD.md | Fluxo Técnico | Função pura publishWebhookEvent(tx, order, fromStatus, toStatus) filtrando status e gravando snapshot na transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo Técnico | Loop de polling de 2s no worker lendo pendentes em lotes pequenos e enviando POST assinado com HMAC-SHA256 | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-03 | docs/FDD.md | Fluxo Técnico | Progressão de backoff exponencial (1m, 5m, 30m, 2h, 12h) e transferência para webhook_dead_letter após 5 falhas | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo Técnico | Replay manual de item da DLQ exigindo role ADMIN e registrando log de auditoria de quem executou a ação | TRANSCRICAO | [09:36] Sofia |
| FDD-CTR-01 | docs/FDD.md | Contrato HTTP | Endpoint POST /api/v1/webhooks para cadastro de webhook com URL https, lista de status, customerId e retorno da secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CTR-02 | docs/FDD.md | Contrato HTTP | Endpoint GET /api/v1/webhooks para listar os webhooks configurados de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-03 | docs/FDD.md | Contrato HTTP | Endpoint PATCH /api/v1/webhooks/:id para editar configuração e filtro de eventos do webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-04 | docs/FDD.md | Contrato HTTP | Endpoint DELETE /api/v1/webhooks/:id para remover um endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-05 | docs/FDD.md | Contrato HTTP | Endpoint POST /api/v1/webhooks/:id/rotate-secret para rotacionar a secret mantendo a antiga válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-CTR-06 | docs/FDD.md | Contrato HTTP | Endpoint GET /api/v1/webhooks/:id/deliveries retornando as últimas 100 entregas com payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CTR-07 | docs/FDD.md | Contrato HTTP | Endpoint POST /api/v1/admin/webhooks/dead-letter/:id/replay protegido por role ADMIN para reprocessar DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-CTR-08 | docs/FDD.md | Contrato HTTP | Payload outbound enxuto (sem items) e headers X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:43] Diego |
| FDD-CTR-08-HDR | docs/FDD.md | Contrato HTTP | Inclusão do header X-Webhook-Id para identificar qual endpoint cadastrado originou o disparo | TRANSCRICAO | [09:44] Sofia |
| FDD-ERR-01 | docs/FDD.md | Código de Erro | Padronização de códigos de erro do módulo com prefixo WEBHOOK_ (WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Código de Erro | Erro quando o payload do evento ultrapassa o teto de 64KB (WEBHOOK_PAYLOAD_TOO_LARGE) | TRANSCRICAO | [09:24] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Uso do logger Pino já existente no projeto e log de auditoria obrigatório no replay administrativo da DLQ | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração com Código | Extensão da transação em OrderService.changeStatus para invocar publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração com Código | Máquina de estados de pedidos (OrderStatus e canTransition) usada como referência para os eventos de status | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração com Código | Adição dos modelos Webhook, WebhookOutbox, WebhookDelivery e WebhookDeadLetter com chaves UUID (@db.Char(36)) | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração com Código | Reuso da classe base AppError para definição dos erros do domínio de webhooks | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração com Código | Reuso das classes de erro HTTP (BadRequestError, NotFoundError, ConflictError, UnprocessableEntityError) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-06 | docs/FDD.md | Integração com Código | Tratamento centralizado automático de AppError, ZodError e erros do Prisma sem alterar o middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração com Código | Reuso dos middlewares authenticate e requireRole('ADMIN') para proteção das rotas de webhooks e DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração com Código | Reuso do middleware validate com schemas Zod para validar payloads, params, query e URL https obrigatória | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração com Código | Reuso do logger estruturado Pino com redaction de campos sensíveis | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração com Código | Registro do WebhookController em buildControllers e montagem da API em buildApp | CODIGO | src/app.ts |
| FDD-INT-11 | docs/FDD.md | Integração com Código | Montagem das rotas do módulo de webhooks no roteador principal /api/v1 | CODIGO | src/routes/index.ts |
| FDD-INT-12 | docs/FDD.md | Integração com Código | Padrão de bootstrap e graceful shutdown (SIGINT/SIGTERM) usado como referência para o novo src/worker.ts | CODIGO | src/server.ts |
| FDD-INT-13 | docs/FDD.md | Integração com Código | Fábrica createPrismaClient() reutilizada para instanciar PrismaClient isolado no processo do worker | CODIGO | src/config/database.ts |
| FDD-INT-14 | docs/FDD.md | Integração com Código | Helper paginated() reutilizado na listagem paginada de webhooks | CODIGO | src/shared/http/response.ts |
| FDD-INT-15 | docs/FDD.md | Integração com Código | Adição do script npm run worker e reuso das dependências Node >= 20, Prisma, Express, Zod, Pino e UUID | CODIGO | package.json |
| PRD-PRB-01 | docs/PRD.md | Problema de Negócio | Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) sofrem com integração lenta e cara via polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-02 | docs/PRD.md | Risco de Negócio | Atlas Comercial ameaça migrar para concorrente se a notificação de pedidos não for entregue até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo e Métrica | Meta quantitativa de latência < 10 segundos para notificação (atendida com polling de 2 segundos) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo e Métrica | Meta de entrega completa em 3 sprints (incluindo revisão de segurança) até o fim de novembro | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo e Métrica | Meta de resiliência cobrindo quase 15 horas de indisponibilidade com 5 tentativas (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo e Métrica | Disponibilização do histórico das últimas 100 entregas por endpoint de webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-OUT-01 | docs/PRD.md | Fora de Escopo | Envio de e-mail ao cliente após 3 falhas seguidas descartado para esta fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de Escopo | Dashboard / painel visual para o cliente descartado (escopo restrito a endpoints de API) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de Escopo | Webhooks inbound (enviados pelo cliente para o OMS) fora do escopo | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-04 | docs/PRD.md | Fora de Escopo | Rate limiting de envio para o cliente fora do escopo inicial (observar e decidir depois) | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-05 | docs/PRD.md | Fora de Escopo | Arquivamento de eventos entregues na outbox após 30 dias fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de Escopo | Garantia de ordenação global entre pedidos distintos fora do escopo | TRANSCRICAO | [09:14] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (POST) com url, lista de status e customerId, gerando e retornando secret única | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição (PATCH), remoção (DELETE) e listagem (GET) dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status na hora de inserir na outbox (não insere se nenhum webhook assinar o status) | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Inserção atômica na webhook_outbox na transação de changeStatus com snapshot do payload renderizado na inserção | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Disparo HTTP POST com headers X-Event-Id, X-Signature (HMAC-SHA256), X-Timestamp, X-Webhook-Id e Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Endpoint para rotação de secret pelo cliente com validade paralela de 24 horas da secret antiga | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Retentativa automática em até 5 vezes com backoff exponencial de 1m, 5m, 30m, 2h e 12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Persistência de eventos que falharem nas 5 tentativas na tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Consulta das últimas 100 entregas via GET /webhooks/:id/deliveries com status, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via POST /admin/webhooks/dead-letter/:id/replay exigindo role ADMIN e log de auditoria | TRANSCRICAO | [09:36] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Worker em polling a cada 2 segundos atendendo latência de notificação abaixo de 10 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos para a chamada HTTP do worker ao endpoint do cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Obrigatoriedade de TLS (URL https://) validada via schema Zod | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite máximo de 64KB para o payload do evento, retornando erro caso ultrapasse | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once com deduplicação pelo cliente via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Uso de UUID como identificador na outbox e demais tabelas seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | Reserva de pelo menos 2 dias úteis na sprint final para revisão de código de segurança por Sofia | TRANSCRICAO | [09:46] Sofia |
| PRD-RSK-01 | docs/PRD.md | Risco | Risco de migração da Atlas Comercial para concorrente mitigado pelo escopo enxuto entregue em 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Risco de vazamento de secret em log de aplicação do cliente mitigado por secret única e rotação com grace period de 24h | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-03 | docs/PRD.md | Risco | Risco de cliente indisponível em manutenção planejada de horas mitigado pela janela de 15h de backoff e DLQ | TRANSCRICAO | [09:16] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Risco de duplicidade no cliente em caso de retry mitigado pelo envio de X-Event-Id e documentação no portal | TRANSCRICAO | [09:25] Diego |
| PRD-RSK-05 | docs/PRD.md | Risco | Risco de rajada de chamadas em clientes com muitos pedidos mudando simultaneamente monitorado para futuro rate limiting | TRANSCRICAO | [09:38] Diego |
