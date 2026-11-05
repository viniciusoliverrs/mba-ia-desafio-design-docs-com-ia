# ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ) em Tabela Separada

- **ID de Rastreabilidade:** `ADR-003`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (PM), Sofia (Engenheira de Segurança)

## Status

Aceito

## Contexto

Como os webhooks são enviados para servidores externos operados pelos clientes B2B, falhas transitórias de rede, manutenções programadas ou indisponibilidades temporárias nos sistemas de destino são inevitáveis (`[09:14] Larissa`, `[09:16] Diego`). Além disso, endpoints lentos que não respondem rapidamente podem degradar a vazão do worker (`[09:42] Sofia`).

Precisamos definir:
1. Qual o tempo máximo de espera por resposta HTTP em cada tentativa.
2. Quantas retentativas automáticas realizar e com qual espaçamento temporal antes de desistir.
3. Onde armazenar os eventos que esgotarem todas as tentativas sem sucesso e como permitir sua recuperação quando o cliente restabelecer seu serviço.

## Decisão

Adotaremos uma política de **até 5 tentativas de entrega com Backoff Exponencial** e timeout de 10 segundos por requisição, seguida de movimentação para uma **Dead Letter Queue (DLQ) persistida em tabela separada (`webhook_dead_letter`)** com suporte a **replay manual via endpoint administrativo** (`[09:17] Larissa`, `[09:18] Diego`, `[09:19] Larissa`, `[09:42] Diego`).

Regras detalhadas da decisão:
1. **Timeout por Chamada HTTP:** Cada requisição HTTP `POST` disparada pelo worker terá timeout estrito de **10 segundos**. Respostas acima de 10s (ou códigos HTTP fora da faixa `2xx` / erros de conexão) são tratadas como falha da tentativa (`[09:42] Diego`, `[09:42] Sofia`).
2. **Limite de 5 Tentativas e Progressão de Backoff:** Após a falha inicial, o evento passará por até 5 retentativas com os seguintes intervalos de espera:
   - 1ª retentativa: **1 minuto**
   - 2ª retentativa: **5 minutos**
   - 3ª retentativa: **30 minutos**
   - 4ª retentativa: **2 horas**
   - 5ª retentativa: **12 horas**
   - Janela total de cobertura: aproximadamente **14h36min (~15 horas)** entre a primeira falha e a última tentativa (`[09:16] Larissa`, `[09:17] Diego`, `[09:17] Larissa`).
3. **DLQ em Tabela Dedicada (`webhook_dead_letter`):** Esgotadas as 5 tentativas sem sucesso, o evento é removido do fluxo ativo da `webhook_outbox` e persistido na tabela separada `webhook_dead_letter`, armazenando o payload original, o motivo da última falha (`error_message` / status HTTP) e o timestamp (`[09:18] Diego`).
4. **Replay Manual Restrito a `ADMIN` com Auditoria:** Um endpoint administrativo `POST /api/v1/admin/webhooks/dead-letter/:id/replay` recolocará o evento na `webhook_outbox` com status `PENDING` para novo ciclo de envio (`[09:18] Diego`, `[09:35] Diego`). Este endpoint exigirá obrigatoriamente a role `ADMIN` (reutilizando `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`) e registrará em log estruturado (via `logger` de `src/shared/logger/index.ts`) o identificador do usuário administrador que acionou o replay para fins de auditoria (`[09:36] Sofia`, `[09:36] Larissa`).

## Alternativas Consideradas

### Alternativa 1: Retry Indefinido com Backoff (`ADR-003-ALT-01`)
- **Descrição:** Continuar retentando o envio indefinidamente até que o endpoint do cliente responda com sucesso.
- **Motivo do Descarte:** Descartada (`[09:15] Diego`). Se um cliente desativar seu servidor ou mudar de endereço sem atualizar o cadastro, eventos ficariam pendurados para sempre poluindo o processamento ativo.

### Alternativa 2: Limite Agressivo de 3 Tentativas (`ADR-003-ALT-02`)
- **Descrição:** Limitar a política a apenas 3 tentativas em uma janela curta (~30 minutos) antes de enviar para a DLQ (`[09:16] Bruno`).
- **Motivo do Descarte:** Descartada (`[09:16] Diego`). O histórico operacional da empresa já registrou clientes B2B com janelas de manutenção planejada de até 2 horas; matar o evento em 30 minutos geraria falsos positivos excessivos na DLQ e acionamento manual desnecessário.

### Alternativa 3: Marcar como `FAILED` na própria tabela `webhook_outbox` sem tabela DLQ separada (`ADR-003-ALT-03`)
- **Descrição:** Manter os eventos permanentemente falhos na própria tabela `webhook_outbox` apenas alterando a coluna `status = 'FAILED'` (`[09:17] Larissa`).
- **Motivo do Descarte:** Descartada (`[09:18] Diego`). Uma tabela `webhook_dead_letter` separada mantém a tabela principal `webhook_outbox` enxuta e rápida para o polling contínuo do worker, além de servir como evidência isolada para diagnóstico, auditoria e reprocessamento seguro.

## Consequências

### Positivas
- **Resiliência a Manutenções de Clientes B2B:** A janela de ~15 horas (1m, 5m, 30m, 2h, 12h) absorve quedas temporárias e manutenções programadas de várias horas sem intervenção humana (`[09:16] Diego`, `[09:17] Marcos`).
- **Performance Constante na Outbox Principal:** A extração de falhas definitivas para `webhook_dead_letter` evita o acúmulo de registros mortos no índice varrido a cada 2 segundos pelo worker (`[09:18] Diego`).
- **Governança e Recuperação Controlada:** O replay manual via `POST /api/v1/admin/webhooks/dead-letter/:id/replay` protegido por `requireRole('ADMIN')` e auditado via Pino garante que nenhuma notificação seja perdida definitivamente e que reprocessamentos sejam rastreáveis (`[09:18] Diego`, `[09:36] Sofia`).

### Negativas e Trade-offs
- **Ausência de Alerta Proativo Automático ao Cliente no MVP (Trade-off aceito):** Caso o endpoint do cliente fique indisponível por mais de 15 horas, o evento cairá na DLQ sem envio automático de e-mail ao cliente (notificação por e-mail foi explicitamente deixada fora de escopo para uma próxima fase, `[09:37] Marcos`, `[09:37] Larissa`). Enquanto isso, o cliente pode inspecionar falhas pelo endpoint `GET /api/v1/webhooks/:id/deliveries` (`[09:34] Marcos`).
- **Operação Manual para Replay de DLQ:** Eventos que excederem 5 tentativas exigem intervenção de um usuário `ADMIN` para reprocessamento após a normalização do cliente (`[09:18] Diego`).

