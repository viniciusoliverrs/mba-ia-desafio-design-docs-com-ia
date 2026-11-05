# ADR-005: Semântica de Entrega At-Least-Once e Idempotência no Cliente via X-Event-Id

- **ID de Rastreabilidade:** `ADR-005`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Diego (Engenheiro Sênior de Plataforma), Larissa (Tech Lead), Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno de Pedidos), Marcos (PM)

## Status

Aceito

## Contexto

Em sistemas distribuídos que realizam chamadas HTTP através da internet pública com política de timeout (10s) e retentativas automáticas ([ADR-003](./ADR-003-politica-de-retry-com-backoff-e-dlq.md)), existem cenários clássicos de ambiguidade:
- O servidor do cliente recebe o `POST` do webhook, processa a mudança de status internamente, mas demora mais de 10 segundos para devolver o HTTP `200 OK` (ou a conexão TCP cai antes da resposta chegar ao worker do OMS).
- O worker do OMS interpreta o timeout como falha e agenda uma nova tentativa, enviando o mesmo evento uma segunda vez.
- Caso o worker sofra queda abrupta logo após receber `200 OK` e antes de comitar `status = 'DELIVERED'` no MySQL, o evento também será reprocessado.

Precisamos definir qual garantia de entrega o OMS oferecerá contratualmente aos clientes B2B e como evitar efeitos colaterais duplicados nos sistemas receptores (`[09:24] Diego`, `[09:25] Bruno`).

## Decisão

O Sistema de Webhooks garantirá entrega **at-least-once** (pelo menos uma vez), fornecendo um identificador único imutável por evento (**`event_id`** no corpo JSON e **`X-Event-Id`** no header HTTP) para que o **cliente realize a deduplicação idempotente** do seu lado (`[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`).

Diretrizes técnicas e contratuais da decisão:
1. **Geração de UUID Único na Outbox:** Quando o evento é criado na tabela `webhook_outbox` dentro da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), um `UUID` v4 único é atribuído ao evento (`[09:25] Diego`, `[09:51] Larissa`).
2. **Preservação do `X-Event-Id` em Retries e Replays:** Todas as tentativas de entrega (inclusive retentativas por backoff exponencial e replays manuais vindos da DLQ) preservam exatamente o mesmo `event_id` no payload JSON e no header `X-Event-Id` (`[09:25] Diego`, `[09:43] Diego`, `[09:44] Diego`).
3. **Documentação Explícita para os Clientes:** O contrato de idempotência pelo `X-Event-Id` / `event_id` será destacado na documentação do portal de desenvolvedor para os clientes B2B (`[09:26] Marcos`).

## Alternativas Consideradas

### Alternativa 1: Garantia de Entrega Exactly-Once (Exatamente Uma Vez) (`ADR-005-ALT-01`)
- **Descrição:** Implementar um protocolo de coordenação em duas fases (*two-phase commit* / handshake transacional de confirmação) entre o OMS e cada servidor de cliente para assegurar que nenhum evento seja entregue mais de uma vez.
- **Motivo do Descarte:** Descartada por complexidade desproporcional e acoplamento inviável com clientes externos (`[09:25] Diego`). Exigiria coordenação distribuída dos dois lados da integração, fugindo do padrão consolidado da indústria (Stripe, GitHub, etc.), onde *at-least-once* combinado com `event_id` resolve 99% dos casos de uso (`[09:25] Diego`).

### Alternativa 2: Semântica At-Most-Once (Dispara uma vez e descarta em caso de dúvida/timeout) (`ADR-005-ALT-02`)
- **Descrição:** Enviar o webhook apenas uma vez sem retentativas para evitar qualquer possibilidade de duplicidade no cliente.
- **Motivo do Descarte:** Descartada pois viola diretamente a necessidade de confiabilidade das notificações de pedidos B2B e a decisão de retry com backoff exponencial ([ADR-003](./ADR-003-politica-de-retry-com-backoff-e-dlq.md)) (`[09:15] Diego`).

## Consequências

### Positivas
- **Alinhamento com Padrões de Mercado:** Segue exatamente a abordagem adotada por provedores de referência como Stripe e GitHub, facilitando a adoção pelos times de engenharia da Atlas Comercial, MaxDistribuição e Nova Cargo (`[09:25] Diego`).
- **Simplicidade e Alta Confiabilidade no Emissor:** O worker do OMS pode retentar envios de forma segura sempre que houver timeout ou falha de rede, garantindo que nenhum evento seja perdido silenciosamente (`[09:24] Diego`).

### Negativas e Trade-offs
- **Transferência da Responsabilidade de Deduplicação para o Cliente (Trade-off aceito):** O consumidor do webhook precisa armazenar ou verificar os `X-Event-Id` já processados para garantir idempotência do seu lado (`[09:25] Sofia`, `[09:25] Diego`). Esse trade-off é mitigado pela documentação clara no portal do desenvolvedor (`[09:26] Marcos`) e pelo envio consistente do UUID tanto no header `X-Event-Id` quanto no campo `event_id` do payload JSON (`[09:43] Diego`, `[09:44] Diego`).

