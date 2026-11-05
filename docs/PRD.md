# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

- **ID do Documento:** `PRD-001`
- **Produto:** Order Management System (OMS)
- **Feature:** Sistema de Webhooks Outbound de Notificação de Pedidos
- **Product Manager:** Marcos (`[09:00] Marcos`)
- **Tech Lead:** Larissa (`[09:00] Larissa`)
- **Engenharia & Segurança:** Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança)
- **Prazo Alvo:** 3 Sprints / Fim de Novembro (Fim do Trimestre) (`[09:00] Marcos`, `[09:45] Marcos`, `[09:46] Larissa`)

---

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) processa o ciclo completo de pedidos B2B (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` em `prisma/schema.prisma` e `src/modules/orders/order.status.ts`). Atualmente, a plataforma oferece apenas endpoints REST de consulta passiva (`GET /api/v1/orders` e `GET /api/v1/orders/:id`), sem qualquer mecanismo de notificação ativa quando um pedido muda de status.

Esta iniciativa introduz um **Sistema de Webhooks Outbound de Notificação de Pedidos**, permitindo que clientes B2B cadastrem endpoints HTTPS via API para receber notificações automáticas em tempo quase real (< 10 segundos) sempre que seus pedidos sofrerem mudanças de status, com garantia de autenticidade (`HMAC-SHA256`), filtro por eventos de interesse, histórico de entregas e alta resiliência (`[09:00] Marcos`, `[09:02] Marcos`, `[09:48] Larissa`).

---

## 2. Problema e Motivação

### 2.1 O Problema Atual (`PRD-PRB-01`)
Três clientes B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — formalizaram a necessidade urgente de receber notificações ativas de mudança de status de pedidos (`[09:00] Marcos`). Hoje, os sistemas desses clientes precisam realizar polling repetitivo no endpoint `GET /api/v1/orders` para verificar se houve atualização, o que torna a integração lenta, ineficiente e cara para a operação deles (`[09:00] Marcos`).

### 2.2 Urgência de Negócio e Risco de Churn (`PRD-PRB-02`)
A **Atlas Comercial** sinalizou explicitamente que, caso a funcionalidade não seja entregue até o **fim do trimestre (fim de novembro)**, poderá migrar sua operação para um concorrente (`[09:00] Marcos`, `[09:45] Marcos`). Portanto, a entrega dessa feature em **3 sprints** é crítica para retenção de receita e competitividade da plataforma B2B (`[09:46] Larissa`, `[09:47] Marcos`).

---

## 3. Público-Alvo e Cenários de Uso

### 3.1 Público-Alvo (`PRD-PUB-01`)
1. **Clientes B2B Integradores (Atlas Comercial, MaxDistribuição, Nova Cargo e demais contas B2B):** Times de engenharia e usuários autenticados via API que integram seus ERPs/sistemas logísticos ao OMS e precisam reagir rapidamente às mudanças de status de pedidos (`[09:00] Marcos`, `[09:32] Marcos`).
2. **Administradores da Plataforma OMS (Role `ADMIN`):** Equipe interna de operações/plataforma responsável por monitorar falhas permanentes na Dead Letter Queue (DLQ) e acionar o reprocessamento (*replay*) manual após a normalização do ambiente do cliente (`[09:18] Diego`, `[09:36] Sofia`).

### 3.2 Cenários de Uso Principais (`PRD-CEN-01`)
- **Cenário 1 — Configuração Seletiva de Eventos:** O integrador do cliente B2B cadastra seu endpoint HTTPS informando que deseja receber apenas eventos de `SHIPPED` e `DELIVERED`, recebendo na resposta a `secret` exclusiva para validar assinaturas (`[09:31] Marcos`, `[09:33] Marcos`).
- **Cenário 2 — Recebimento de Mudança de Status em < 10s:** Quando um pedido muda para `SHIPPED` no OMS, o cliente recebe em até 2–10 segundos um `POST` assinado com os dados básicos do pedido e o `X-Event-Id`; se precisar da lista detalhada de itens do pedido, consulta `GET /api/v1/orders/:id` (`[09:02] Marcos`, `[09:10] Larissa`, `[09:43] Diego`).
- **Cenário 3 — Recuperação de Indisponibilidade Temporária do Cliente:** O servidor do cliente entra em manutenção planejada por 2 horas; o OMS retenta automaticamente com backoff exponencial até entregar com sucesso sem perder o evento (`[09:16] Diego`, `[09:17] Diego`).
- **Cenário 4 — Auditoria de Entregas e Rotação de Secret:** O cliente consulta os últimos 100 disparos em `GET /api/v1/webhooks/:id/deliveries` para diagnosticar latência/respostas (`[09:34] Marcos`) e, caso suspeite de exposição de sua chave em logs, rotaciona a secret via API mantendo a chave anterior válida por 24 horas para migração sem downtime (`[09:21] Sofia`, `[09:22] Diego`).

---

## 4. Objetivos e Métricas de Sucesso

| ID | Objetivo | Métrica Quantitativa | Meta de Sucesso | Fonte |
| --- | --- | --- | --- | --- |
| `PRD-OBJ-01` | Entregar notificações em "tempo real" para eliminar o polling em `GET /orders` | Latência entre o commit da mudança de status e o disparo HTTP pelo worker | **< 10 segundos** (com ciclo de polling configurado em **2 segundos**) | `[09:00] Marcos`<br>`[09:02] Marcos`<br>`[09:10] Larissa` |
| `PRD-OBJ-02` | Retenção da Atlas Comercial e ativação das contas B2B demandantes | Prazo de entrega em produção e adoção pelas 3 contas B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) | Entrega concluída em **até 3 sprints (fim de novembro)** com **0 churn** da Atlas Comercial | `[09:00] Marcos`<br>`[09:45] Marcos`<br>`[09:46] Larissa` |
| `PRD-OBJ-03` | Garantir resiliência contra indisponibilidades temporárias de clientes | Janela de cobertura de retentativas automáticas antes de enviar para DLQ | Cobertura de **~15 horas (14h36m)** através de **5 tentativas** (`1m`, `5m`, `30m`, `2h`, `12h`) e **0 perda** de eventos comitados | `[09:06] Diego`<br>`[09:16] Diego`<br>`[09:17] Diego` |
| `PRD-OBJ-04` | Visibilidade de autoatendimento para diagnóstico de integração pelo cliente | Histórico consultável de entregas por endpoint de webhook | Disponibilização das **últimas 100 entregas** (`GET /webhooks/:id/deliveries`) com payload, status, response e tempo de resposta | `[09:34] Marcos` |

---

## 5. Escopo (Incluso e Fora de Escopo)

### 5.1 Dentro do Escopo (Incluso nesta Fase) (`PRD-ESC-IN`)
- Envio *outbound* de webhooks HTTP `POST` em formato JSON quando o status de um pedido muda (`[09:02] Marcos`, `[09:03] Sofia`).
- API autenticada via JWT para CRUD de configuração de webhooks por `customerId` (`POST`, `GET`, `PATCH`, `DELETE`) com filtro de status de interesse (`[09:31] Marcos`, `[09:32] Larissa`, `[09:33] Bruno`, `[09:33] Marcos`).
- Geração de `secret` exclusiva por endpoint, assinatura `HMAC-SHA256` (`X-Signature`) e endpoint de rotação de secret com *grace period* de 24 horas (`[09:20] Sofia`, `[09:21] Sofia`).
- Registro transacional via Outbox no MySQL e processamento assíncrono por worker separado em polling de 2s (`[09:06] Diego`, `[09:09] Diego`, `[09:11] Diego`).
- Política de 5 retentativas com backoff exponencial, tabela separada de Dead Letter Queue (`webhook_dead_letter`) e endpoint de replay manual restrito à role `ADMIN` (`[09:17] Larissa`, `[09:18] Diego`, `[09:36] Larissa`).
- Endpoint de consulta do histórico das últimas 100 entregas por webhook (`GET /webhooks/:id/deliveries`) (`[09:34] Marcos`).
- Documentação no portal do desenvolvedor orientando os clientes sobre integração via API e deduplicação por `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

### 5.2 Fora de Escopo (Explicitamente Descartados ou Adiados na Reunião) (`PRD-ESC-OUT`)
1. **`PRD-OUT-01` — Notificação por E-mail ao Cliente após Falhas Consecutivas:** A ideia de enviar um e-mail automático ao cliente quando o webhook falhar 3 vezes seguidas foi explicitamente descartada para esta fase e adiada para uma fase futura após medição de impacto (`[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos`).
2. **`PRD-OUT-02` — Dashboard / Painel Visual de Gerenciamento de Webhooks:** Construção de interface gráfica (frontend) para os clientes visualizarem ou cadastrarem webhooks foi descartada; o escopo contempla exclusivamente endpoints de API (`[09:39] Marcos`, `[09:40] Larissa`).
3. **`PRD-OUT-03` — Webhooks Inbound (Recebimento de Eventos Enviados pelos Clientes):** O fluxo é estritamente *outbound* (do OMS para o cliente); clientes não enviam webhooks para o OMS (`[09:02] Sofia`, `[09:02] Marcos`, `[09:03] Sofia`).
4. **`PRD-OUT-04` — Rate Limiting de Envio para os Clientes:** Limitação de taxa de disparos por minuto para um mesmo cliente foi deixada fora do escopo inicial; o time irá apenas observar em produção e decidir posteriormente se necessário (`[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa`).
5. **`PRD-OUT-05` — Rotina de Arquivamento/Expurgo de Eventos Entregues após 30 Dias:** A automação de arquivamento de linhas antigas já entregues na tabela `webhook_outbox` após 30 dias foi explicitamente declarada fora do escopo desta feature (`[09:08] Diego`).
6. **`PRD-OUT-06` — Garantia de Ordenação Global e Escala Multi-Worker no Lançamento:** Os clientes não solicitaram ordenação global (`[09:14] Marcos`); o escalonamento horizontal com múltiplos workers particionados por `order_id` ou lock pessimista ficou para o futuro (`[09:12] Diego`, `[09:13] Diego`, `[09:13] Larissa`).

---

## 6. Requisitos Funcionais

| ID | Requisito Funcional | Descrição Detalhada | Origem |
| --- | --- | --- | --- |
| `PRD-FR-01` | **Cadastro de Webhook (`POST`)** | Permitir cadastrar um endpoint de webhook informando `customerId`, `url` (`https://`) e a lista de status (`OrderStatus`) que deseja receber; o sistema deve gerar automaticamente uma `secret` exclusiva para o endpoint e devolvê-la na resposta de criação. | `[09:21] Sofia`<br>`[09:31] Marcos`<br>`[09:32] Larissa` |
| `PRD-FR-02` | **Listagem, Edição e Remoção de Webhooks (`GET`, `PATCH`, `DELETE`)** | Permitir listar os webhooks configurados de um `customerId`, editar suas configurações (`PATCH` de URL, eventos inscritos e estado ativo) e remover um webhook (`DELETE`). | `[09:21] Bruno`<br>`[09:33] Bruno` |
| `PRD-FR-03` | **Filtro de Eventos por Status na Inserção da Outbox** | Permitir selecionar por endpoint quais status do pedido disparam notificação (ex.: apenas `SHIPPED` e `DELIVERED`), filtrando no momento da inserção na `webhook_outbox` para não criar registros caso nenhum webhook do cliente assine aquele status. | `[09:33] Marcos`<br>`[09:34] Bruno`<br>`[09:34] Diego` |
| `PRD-FR-04` | **Enfileiramento Atômico com Snapshot na Mudança de Status** | Ao alterar o status de um pedido (`OrderService.changeStatus`), inserir o evento na tabela `webhook_outbox` dentro da mesma transação SQL contendo o snapshot já renderizado do payload JSON (com `event_id`, `event_type: "order.status_changed"`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`, omitindo `items`). | `[09:06] Diego`<br>`[09:40] Bruno`<br>`[09:43] Diego`<br>`[09:52] Larissa` |
| `PRD-FR-05` | **Envio Assíncrono com Headers Padronizados e Assinatura HMAC** | Disparar requisição HTTP `POST` para a URL do cliente contendo `Content-Type: application/json`, `X-Event-Id` (UUID do evento), `X-Signature` (assinatura `HMAC-SHA256`), `X-Timestamp` (timestamp do envio) e `X-Webhook-Id` (ID do endpoint cadastrado). | `[09:20] Sofia`<br>`[09:44] Diego`<br>`[09:44] Sofia` |
| `PRD-FR-06` | **Rotação de Secret com Grace Period de 24 Horas** | Disponibilizar endpoint para o cliente solicitar uma nova `secret` via API, mantendo a `secret` antiga válida em paralelo por 24 horas para permitir migração sem perda de eventos. | `[09:21] Sofia`<br>`[09:22] Sofia` |
| `PRD-FR-07` | **Retentativa Automática com Backoff Exponencial** | Em caso de falha ou timeout (> 10s) no envio ao cliente, retentar automaticamente até 5 vezes seguindo a progressão de backoff de `1 minuto`, `5 minutos`, `30 minutos`, `2 horas` e `12 horas`. | `[09:16] Larissa`<br>`[09:17] Diego`<br>`[09:17] Larissa` |
| `PRD-FR-08` | **Movimentação para Dead Letter Queue (DLQ)** | Após esgotar as 5 tentativas sem sucesso, mover o evento para a tabela separada `webhook_dead_letter` registrando o payload original, o motivo da falha e o timestamp. | `[09:18] Diego` |
| `PRD-FR-09` | **Consulta de Histórico de Entregas (`GET /webhooks/:id/deliveries`)** | Permitir ao cliente consultar as últimas 100 tentativas de entrega de um webhook, exibindo indicador de sucesso/falha, payload enviado, resposta recebida e tempo de resposta. | `[09:34] Marcos` |
| `PRD-FR-10` | **Replay Manual de DLQ por Administrador (`POST /admin/webhooks/dead-letter/:id/replay`)** | Permitir que usuários com role `ADMIN` reprocessam manualmente um evento da `webhook_dead_letter`, recolocando-o na `webhook_outbox` como pendente e registrando em log de auditoria quem executou o replay. | `[09:18] Diego`<br>`[09:35] Diego`<br>`[09:36] Sofia`<br>`[09:36] Larissa` |

---

## 7. Requisitos Não Funcionais

- **`PRD-NFR-01` — Latência de Notificação:** Intervalo de polling do worker de **2 segundos**, garantindo entrega em **menos de 10 segundos** em condições normais de operação (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`).
- **`PRD-NFR-02` — Timeout de Chamada Externa:** Timeout máximo de **10 segundos** por requisição HTTP enviada ao cliente (`[09:42] Sofia`, `[09:42] Diego`).
- **`PRD-NFR-03` — Segurança de Transporte (TLS Obrigatório):** Apenas URLs com esquema `https://` são aceitas no cadastro/edição de webhooks; URLs `http://` devem ser rejeitadas na validação Zod (`[09:23] Sofia`).
- **`PRD-NFR-04` — Limite de Tamanho de Payload:** O payload de cada evento de webhook tem teto estrito de **64KB**, retornando erro caso seja ultrapassado (`[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`).
- **`PRD-NFR-05` — Semântica de Entrega e Ordenação:** Garantia de entrega **at-least-once** com deduplicação no cliente via `X-Event-Id` (`[09:24] Diego`, `[09:25] Diego`) e ordenação implícita por `order_id` (`created_at`) enquanto operar em *single-worker* (`[09:12] Diego`, `[09:13] Larissa`).
- **`PRD-NFR-06` — Isolamento de Processos e Padronização Técnica:** O worker deve rodar como processo separado da API (`src/worker.ts`) (`[09:11] Diego`), reutilizando `PrismaClient`, logger Pino (`src/shared/logger/index.ts`), `AppError` com códigos prefixados por `WEBHOOK_` e identificadores `UUID` (`[09:28] Bruno`, `[09:29] Larissa`, `[09:30] Larissa`, `[09:51] Larissa`).

---

## 8. Decisões e Trade-offs Principais

1. **Outbox no MySQL vs. Broker Externo (Redis Streams) (`PRD-TRD-01`):** Optou-se por usar o MySQL existente em vez de subir Redis Cluster para evitar complexidade operacional em um time enxuto e garantir atomicidade na mesma transação SQL, aceitando o trade-off de maior volume de escrita no banco relacional (`[09:06] Diego`, `[09:07] Diego`, [ADR-001](./adrs/ADR-001-outbox-no-mysql.md)).
2. **Polling de 2s em Processo Separado vs. Triggers / Mesmo Processo (`PRD-TRD-02`):** Adotou-se polling a cada 2s em processo independente (`src/worker.ts`) porque o MySQL não possui `NOTIFY/LISTEN` nativo para processos externos e o isolamento evita que reinicializações da API derrubem o worker, cumprindo o SLA < 10s (`[09:09] Diego`, `[09:11] Diego`, [ADR-002](./adrs/ADR-002-worker-em-processo-separado-em-polling.md)).
3. **5 Tentativas (~15h) + Tabela DLQ Separada vs. 3 Tentativas / Retry Infinito (`PRD-TRD-03`):** Cinco tentativas equilibram resiliência a manutenções de clientes de várias horas sem deixar eventos pendurados eternamente, isolando falhas definitivas em `webhook_dead_letter` (`[09:15] Diego`, `[09:16] Diego`, `[09:18] Diego`, [ADR-003](./adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md)).
4. **At-Least-Once com `X-Event-Id` vs. Exactly-Once (`PRD-TRD-04`):** Transferiu-se a responsabilidade de deduplicação idempotente para o cliente via `X-Event-Id` (padrão Stripe/GitHub) em troca de eliminar a complexidade proibitiva de coordenação distribuída bilateral (`[09:25] Sofia`, `[09:25] Diego`, [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
5. **Payload Enxuto (Snapshot sem `items`) vs. Payload Completo (`PRD-TRD-05`):** O webhook carrega o snapshot dos dados essenciais do pedido na transição de status sem a lista de itens para manter o payload leve (< 64KB); clientes que necessitarem dos itens consultam `GET /api/v1/orders/:id` (`[09:43] Diego`, `[09:52] Larissa`).

---

## 9. Dependências

- **Código e Módulos Existentes do OMS (`PRD-DEP-01`):** Integração direta com `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), modelos `Customer`, `Order` e `User` (`prisma/schema.prisma`), middlewares `authenticate`, `requireRole` (`src/middlewares/auth.middleware.ts`), `validate` (`src/middlewares/validate.middleware.ts`), `errorMiddleware` (`src/middlewares/error.middleware.ts`), `AppError` (`src/shared/errors/app-error.ts`) e logger Pino (`src/shared/logger/index.ts`) (`[09:30] Larissa`, `[09:40] Bruno`).
- **Revisão de Segurança Obrigatória (`PRD-DEP-02`):** Alocação de pelo menos **2 dias úteis** da engenheira de segurança (Sofia) antes do deploy para revisar a geração de secrets e validação HMAC (`[09:46] Sofia`).
- **Documentação Externa no Portal do Desenvolvedor (`PRD-DEP-03`):** Atualização da documentação para clientes pelo PM (Marcos) cobrindo os contratos da API e a obrigatoriedade de deduplicação pelo `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

---

## 10. Riscos e Mitigação

| ID | Risco | Probabilidade | Impacto | Estratégia de Mitigação | Origem |
| --- | --- | --- | --- | --- | --- |
| `PRD-RSK-01` | **Churn da Atlas Comercial por atraso no cronograma de fim de trimestre** | Média | Alto | Escopo enxuto focado em API (sem dashboard visual, sem e-mail, sem infra extra de Redis), reuso total dos padrões do código existente e planejamento fechado em 3 sprints (`1 sprint` Outbox/DLQ + `1 sprint` Worker/Retry + `1 sprint` CRUD/Integração/Revisão de Segurança). | `[09:00] Marcos`<br>`[09:45] Marcos`<br>`[09:46] Larissa` |
| `PRD-RSK-02` | **Vazamento acidental de secret de webhook pelo cliente em logs ou falha na validação criptográfica** | Média | Alto | Secret individual por endpoint (isolando o *blast radius*), endpoint de rotação com *grace period* de 24h, HTTPS obrigatório e reserva de 2 dias úteis de revisão dedicada de segurança (Sofia) antes do deploy. | `[09:21] Sofia`<br>`[09:22] Diego`<br>`[09:46] Sofia` |
| `PRD-RSK-03` | **Indisponibilidade prolongada ou lentidão nos servidores dos clientes B2B** | Alta | Médio | Desacoplamento via Transactional Outbox e worker em processo separado, timeout de 10s por chamada, 5 retentativas com backoff exponencial cobrindo ~15h e tabela `webhook_dead_letter` com replay manual por `ADMIN`. | `[09:04] Bruno`<br>`[09:16] Diego`<br>`[09:18] Diego`<br>`[09:42] Diego` |
| `PRD-RSK-04` | **Processamento duplicado de eventos no ERP do cliente em casos de retry** | Média | Médio | Inclusão do UUID único no header `X-Event-Id` e no campo `event_id` do payload JSON, acompanhado de documentação destacada no portal do desenvolvedor sobre deduplicação idempotente. | `[09:24] Diego`<br>`[09:25] Diego`<br>`[09:26] Marcos` |
| `PRD-RSK-05` | **Sobrecarga de requisições em clientes com picos de atualização (ex.: 50 mudanças/min)** | Baixa | Baixo | Monitorar métricas de envio por cliente no lançamento inicial e avaliar implementação de *rate limiting* de saída em fase posterior caso necessário. | `[09:38] Diego`<br>`[09:39] Diego` |

---

## 11. Critérios de Aceitação

- [ ] **CA-01:** Clientes autenticados via JWT conseguem criar (`POST`), listar (`GET`), editar (`PATCH`) e remover (`DELETE`) configurações de webhook associadas a um `customerId`, filtrando os `OrderStatus` desejados (`[09:31] Marcos`, `[09:33] Bruno`).
- [ ] **CA-02:** O cadastro ou edição de webhook com URL `http://` (sem TLS) é rejeitado com erro de validação `400` (`WEBHOOK_INVALID_URL`) (`[09:23] Sofia`, `[09:28] Bruno`).
- [ ] **CA-03:** Ao alterar o status de um pedido em `PATCH /api/v1/orders/:id/status`, o evento é gravado na `webhook_outbox` na mesma transação SQL apenas para os webhooks ativos inscritos naquele status, contendo o snapshot do payload (< 64KB e sem `items`) (`[09:34] Bruno`, `[09:40] Bruno`, `[09:43] Diego`, `[09:52] Larissa`).
- [ ] **CA-04:** O worker em processo separado (`src/worker.ts`) envia o webhook em até 2 segundos (polling) com timeout de 10s e headers `X-Event-Id`, `X-Signature` (`HMAC-SHA256`), `X-Timestamp` e `X-Webhook-Id` (`[09:09] Diego`, `[09:42] Diego`, `[09:44] Diego`, `[09:44] Sofia`).
- [ ] **CA-05:** O cliente consegue rotacionar a secret do endpoint via API, e a secret anterior permanece válida em paralelo por 24 horas (`[09:21] Sofia`).
- [ ] **CA-06:** Falhas de entrega seguem 5 retentativas com backoff de `1m`, `5m`, `30m`, `2h` e `12h`; após a 5ª falha, o evento é movido para `webhook_dead_letter` (`[09:17] Larissa`, `[09:18] Diego`).
- [ ] **CA-07:** O endpoint `GET /api/v1/webhooks/:id/deliveries` retorna as últimas 100 entregas com sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`).
- [ ] **CA-08:** O endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige obrigatoriamente role `ADMIN`, recoloca o evento na `webhook_outbox` como pendente e registra quem executou o replay nos logs (`[09:18] Diego`, `[09:36] Sofia`, `[09:36] Larissa`).

---

## 12. Estratégia de Testes e Validação

A validação será conduzida ao longo das 3 sprints planejadas (`[09:46] Larissa`) utilizando a suíte `vitest` e `supertest` já presente no projeto (`package.json`):

1. **Testes Unitários (Sprints 1 e 2):**
   - Validação de schemas Zod (`https://` obrigatório, lista de `OrderStatus`, limite de 64KB).
   - Geração de assinatura `HMAC-SHA256` e verificação de dupla validade durante o *grace period* de 24h da rotação de secret.
   - Cálculo determinístico dos intervalos de backoff exponencial (`1m`, `5m`, `30m`, `2h`, `12h`) e transição para DLQ na 5ª falha.
2. **Testes de Integração e Ponta a Ponta — E2E (Sprint 3):**
   - Teste transacional de `OrderService.changeStatus`: confirmar que o commit cria linha na `webhook_outbox` com snapshot correto apenas quando há webhook inscrito, e que uma falha na inserção da outbox faz *rollback* da mudança de status do pedido (`[09:40] Bruno`, `[09:46] Larissa`).
   - Testes de contrato HTTP com `supertest` para CRUD de webhooks, histórico das últimas 100 deliveries e controle de acesso `requireRole('ADMIN')` no endpoint de replay da DLQ.
3. **Revisão Dedicada de Segurança (Sprint 3 — 2 dias úteis):**
   - Auditoria de código conduzida por Sofia cobrindo geração de secrets, cálculo de HMAC, headers de segurança e logs de auditoria antes da liberação para produção (`[09:46] Sofia`).
