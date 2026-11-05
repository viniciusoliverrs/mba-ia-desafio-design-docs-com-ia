# ADR-001: Padrão Transactional Outbox no MySQL para Eventos de Webhook

- **ID de Rastreabilidade:** `ADR-001`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (PM), Sofia (Engenheira de Segurança)

## Status

Aceito

## Contexto

Clientes B2B do Order Management System (OMS) — especificamente Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram notificações ativas quando o status de seus pedidos sofrer alteração, eliminando a necessidade de polling contínuo no endpoint `GET /api/v1/orders` (`[09:00] Marcos`). O SLA esperado pelos clientes para considerar a notificação como "tempo real" é de até 10 segundos (`[09:02] Marcos`), operando exclusivamente no sentido *outbound* (do OMS para o cliente, `[09:02] Sofia` / `[09:02] Marcos`).

Atualmente, a mudança de status de um pedido ocorre no método `OrderService.changeStatus` em `src/modules/orders/order.service.ts` (linhas 126–179). Esse método executa uma transação interativa do Prisma (`this.prisma.$transaction(async (tx) => { ... })`) que já realiza operações críticas:
1. Busca o pedido atual e seus itens (`tx.order.findUnique`).
2. Valida a transição de estado usando `canTransition` (`src/modules/orders/order.status.ts`).
3. Debita ou repõe estoque de produtos (`debitStock` / `replenishStock` em `tx.product.update`).
4. Atualiza o status do pedido (`tx.order.update`).
5. Registra a trilha de auditoria em `order_status_history` (`tx.orderStatusHistory.create`).

Precisamos garantir consistência atômica entre a mudança de status do pedido no banco de dados MySQL (`prisma/schema.prisma`) e o agendamento da notificação de webhook: um evento de webhook jamais pode ser emitido se a transação do pedido sofrer *rollback*, e nenhuma mudança de status confirmada pode deixar de gerar o evento correspondente (`[09:06] Diego`, `[09:40] Bruno`).

## Decisão

Adotaremos o padrão **Transactional Outbox** utilizando o banco de dados **MySQL** já existente na aplicação (`prisma/schema.prisma`), sem introduzir infraestrutura adicional de mensageria (`[09:06] Diego`, `[09:08] Larissa`).

Os detalhes da decisão compreendem:
1. **Inserção Atômica na Mesma Transação SQL:** Dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), utilizando o mesmo cliente transacional `tx` (`Prisma.TransactionClient`), será invocada uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno`, `[09:41] Diego`) que insere os registros correspondentes na tabela `webhook_outbox`. Se a inserção na outbox falhar, toda a transação de mudança de status sofre *rollback* (`[09:40] Bruno`).
2. **Filtragem na Inserção:** Antes de inserir na `webhook_outbox`, o sistema verifica quais endpoints ativos daquele `customerId` estão inscritos para o novo status (`toStatus`). Se nenhum webhook do cliente estiver inscrito para aquele status, nenhuma linha é inserida na outbox (`[09:34] Bruno`, `[09:34] Diego`).
3. **Snapshot do Payload Renderizado na Inserção:** O registro na `webhook_outbox` armazenará o snapshot JSON do payload já renderizado no momento exato da transição de status (contendo `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, sem a lista de `items`), preservando a fidelidade histórica mesmo que o pedido sofra novas mutações antes do envio HTTP (`[09:43] Diego`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`).
4. **Identificadores e Índices:** A tabela `webhook_outbox` usará `UUID` (`@db.Char(36)`) como chave primária, mantendo o padrão de `prisma/schema.prisma` (`[09:51] Larissa`), e terá índices compostos/dedicados em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e `created_at` para leitura eficiente em lotes pequenos pelo worker (`[09:08] Diego`).

## Alternativas Consideradas

### Alternativa 1: Disparo HTTP Síncrono dentro de `OrderService.changeStatus` (`ADR-001-ALT-01`)
- **Descrição:** Realizar a chamada HTTP `POST` para a URL do cliente diretamente dentro do método `changeStatus` em `src/modules/orders/order.service.ts`.
- **Motivo do Descarte:** Descartada imediatamente (`[09:04] Bruno`, `[09:06] Diego`). A transação de mudança de status já é pesada (atualiza `orders`, insere em `order_status_history` e altera `stock_quantity` em `products`). Incluir uma chamada HTTP externa dentro da transação seguraria locks no MySQL caso o servidor do cliente estivesse lento, degradando a mudança de status de outros pedidos, além de criar um dilema insolúvel de consistência caso o endpoint do cliente estivesse fora do ar (não seria aceitável dar *rollback* na mudança de status do pedido por falha no webhook do cliente, `[09:04] Bruno`).

### Alternativa 2: Fila/Stream em Infraestrutura Externa (Redis Streams / Redis Cluster) (`ADR-001-ALT-02`)
- **Descrição:** Publicar os eventos de mudança de status em um broker dedicado como Redis Streams / Redis Cluster.
- **Motivo do Descarte:** Descartada por *overengineering* e custo operacional (`[09:07] Larissa`, `[09:07] Diego`). O time de engenharia é enxuto e o projeto hoje depende exclusivamente do MySQL (`docker-compose.yml` e `prisma/schema.prisma`). Além disso, publicar no Redis fora da transação SQL reintroduziria o problema de *dual-write* (o commit no MySQL poderia suceder e a publicação no Redis falhar), exigindo uma tabela outbox de qualquer forma.

### Alternativa 3: Armazenar apenas `order_id` na Outbox e renderizar o payload no momento do envio (`ADR-001-ALT-03`)
- **Descrição:** Gravar somente a referência do pedido na tabela `webhook_outbox` e montar o JSON do evento quando o worker fosse disparar a requisição HTTP.
- **Motivo do Descarte:** Descartada (`[09:51] Bruno`, `[09:52] Larissa`). Caso o pedido mudasse novamente de status ou sofresse alterações antes do envio pelo worker, o payload enviado refletiria um estado posterior e inconsistente com a transição que originou o evento.

## Consequências

### Positivas
- **Atomicidade e Consistência Forte:** Garantia transacional absoluta via `this.prisma.$transaction` em `src/modules/orders/order.service.ts` — o evento só existe se a mudança de status comitar, e vice-versa (`[09:06] Diego`).
- **Zero Infraestrutura Nova:** Reaproveita o banco MySQL e o Prisma ORM já configurados em `src/config/database.ts` e `prisma/schema.prisma` (`[09:07] Diego`).
- **Isolamento de Latência na API:** O endpoint `PATCH /api/v1/orders/:id/status` continua respondendo rapidamente com apenas um `INSERT` local adicional, sem depender da rede ou disponibilidade dos clientes B2B (`[09:04] Bruno`).
- **Imutabilidade Histórica do Evento:** O snapshot do payload garante que cada evento enviado representa fielmente o estado do pedido no instante da transição (`[09:52] Larissa`).

### Negativas e Trade-offs
- **Crescimento de Armazenamento no MySQL (Trade-off aceito):** Trocamos a complexidade de operar um broker dedicado (Redis/RabbitMQ) pelo aumento de escrita e armazenamento de JSON na tabela `webhook_outbox` do MySQL. Para mitigar o impacto de performance de leitura, serão criados índices em `status` e `created_at` e a filtragem de eventos não assinados ocorrerá antes da inserção (`[09:08] Diego`, `[09:34] Bruno`). O arquivamento de registros entregues após 30 dias fica mapeado para fase futura (`[09:08] Diego`).
- **Latência Mínima Determinada pelo Worker:** Diferente de um disparo síncrono imediato, a entrega passa a ser assíncrona (acoplada ao ciclo de polling de 2 segundos definido no [ADR-002](./ADR-002-worker-em-processo-separado-em-polling.md)), o que é amplamente compatível com o requisito de negócio de entrega abaixo de 10 segundos (`[09:02] Marcos`, `[09:10] Larissa`).

