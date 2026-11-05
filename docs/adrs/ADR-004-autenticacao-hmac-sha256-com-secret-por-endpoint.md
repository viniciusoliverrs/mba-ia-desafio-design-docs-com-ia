# ADR-004: Autenticação de Webhooks com HMAC-SHA256, Secret por Endpoint e Rotação com Grace Period

- **ID de Rastreabilidade:** `ADR-004`
- **Data:** 2026-11-05
- **Participantes / Decisores:** Sofia (Engenheira de Segurança), Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (PM)

## Status

Aceito

## Contexto

O Sistema de Webhooks enviará payloads contendo dados comerciais de pedidos (`order_id`, `order_number`, `customer_id`, `from_status`, `to_status`, `total_cents`) pela internet pública para endpoints HTTP mantidos pelos clientes B2B (`[09:19] Sofia`, `[09:43] Diego`).

Sem mecanismos criptográficos de autenticidade e integridade:
1. Um atacante poderia forjar requisições falsas para a URL do cliente simulando mudanças de status de pedidos (*spoofing*).
2. O conteúdo em trânsito poderia ser interceptado ou modificado caso o transporte não fosse seguro (*tampering*).
3. Requisições legítimas antigas poderiam ser retransmitidas maliciosamente (*replay attack*).
4. Caso um cliente vaze acidentalmente sua chave secreta em logs de aplicação — situação que já ocorreu no histórico da empresa (`[09:22] Diego`) —, ele precisa conseguir substituir a chave sem sofrer indisponibilidade imediata na validação dos webhooks em produção (`[09:21] Sofia`).

## Decisão

Adotaremos **assinatura criptográfica HMAC-SHA256** sobre o corpo da requisição utilizando uma **secret exclusiva por endpoint de webhook**, aliada a **obrigatoriedade de HTTPS (TLS)** e **rotação de secret via API com *grace period* de 24 horas** (`[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Sofia`, `[09:23] Sofia`).

Os pilares técnicos da decisão são:
1. **Algoritmo e Headers de Segurança:**
   - O worker calcula o `HMAC-SHA256` sobre o payload JSON serializado usando a `secret` do endpoint e envia o resultado no header `X-Signature` (`[09:20] Sofia`, `[09:22] Sofia`).
   - Toda requisição inclui também `X-Timestamp` (timestamp do envio para permitir detecção de *replay attack* pelo cliente), `X-Event-Id` (UUID do evento) e `X-Webhook-Id` (UUID do cadastro do endpoint receptor) (`[09:44] Diego`, `[09:44] Sofia`).
2. **Secret Única por Endpoint Gerada pelo Sistema:**
   - Cada registro de webhook possui sua própria `secret` criptograficamente segura, gerada pelo OMS no momento do `POST /api/v1/webhooks` e retornada na resposta de criação (`[09:21] Sofia`, `[09:31] Marcos`).
   - Não existirá secret compartilhada globalmente na plataforma (`[09:21] Sofia`).
3. **Rotação de Secret com Grace Period de 24 Horas:**
   - Será disponibilizado um endpoint para o cliente solicitar uma nova secret (`POST /api/v1/webhooks/:id/rotate-secret`) (`[09:21] Sofia`).
   - Ao rotacionar, a nova secret passa a ser a principal, mas a secret anterior permanece válida em paralelo por **24 horas** (*grace period*), permitindo que o cliente atualize seus sistemas consumidores sem perda ou rejeição de eventos; após 24 horas, a secret antiga é invalidada definitivamente (`[09:21] Sofia`, `[09:22] Sofia`).
4. **TLS Obrigatório:**
   - A URL cadastrada ou atualizada para qualquer webhook deve utilizar obrigatoriamente o esquema `https://`, validado via schema Zod (`src/middlewares/validate.middleware.ts`), rejeitando URLs `http://` com código `WEBHOOK_INVALID_URL` (`[09:23] Sofia`, `[09:28] Bruno`).

## Alternativas Consideradas

### Alternativa 1: Secret Global Única da Plataforma (`ADR-004-ALT-01`)
- **Descrição:** Utilizar uma única chave secreta global configurada no `.env` do OMS para assinar os webhooks de todos os clientes.
- **Motivo do Descarte:** Descartada por risco crítico de segurança (`[09:21] Sofia`). Se um único cliente deixasse vazar a secret (ou agisse de má-fé), a segurança de todos os demais clientes da plataforma estaria comprometida simultaneamente.

### Alternativa 2: Secret por Endpoint Estática (Sem Suporte a Rotação ou Grace Period) (`ADR-004-ALT-02`)
- **Descrição:** Gerar uma secret única na criação do endpoint, porém exigir que o cliente delete e recrie o webhook (ou troque instantaneamente a chave) em caso de vazamento.
- **Motivo do Descarte:** Descartada (`[09:21] Sofia`, `[09:22] Diego`). A troca seca (*hard swap*) causa janela de falha nas assinaturas enquanto o cliente implanta a nova secret nos seus servidores, e já houve caso real de cliente que vazou secret em log de aplicação e precisou rotacionar em produção.

### Alternativa 3: Aceitar Endpoints HTTP sem TLS para Ambientes de Clientes (`ADR-004-ALT-03`)
- **Descrição:** Permitir o cadastro de URLs `http://` além de `https://`.
- **Motivo do Descarte:** Descartada (`[09:23] Sofia`). Mesmo com assinatura HMAC garantindo integridade, o tráfego em texto claro exporia informações comerciais de pedidos na rede; portanto, `https://` é obrigatório.

## Consequências

### Positivas
- **Autenticidade, Integridade e Confidencialidade:** Clientes podem verificar matematicamente via `HMAC-SHA256` (`X-Signature`) e `X-Timestamp` que a notificação partiu do OMS, não foi alterada em trânsito e não é um *replay* tardio (`[09:19] Sofia`, `[09:44] Diego`).
- **Isolamento de Blast Radius:** O comprometimento da secret de um endpoint afeta exclusivamente aquele endpoint específico (`[09:21] Sofia`).
- **Zero-Downtime em Resposta a Incidentes:** O *grace period* de 24 horas permite rotação segura de credenciais sem interromper a integração do cliente (`[09:21] Sofia`).

### Negativas e Trade-offs
- **Complexidade Adicional na Modelagem e Assinatura durante o Grace Period (Trade-off aceito):** A tabela de configuração de webhook precisará armazenar `secret`, `previous_secret` e `previous_secret_expires_at`, e o header `X-Signature` durante a janela de 24h deverá carregar as assinaturas compatíveis (ou permitir validação com ambas as chaves) para que o cliente migre sem falhas (`[09:21] Sofia`).
- **Revisão Obrigatória de Segurança antes do Deploy:** Exige alocação de pelo menos 2 dias úteis da engenheira de segurança (Sofia) na terceira sprint para auditar a geração de secrets e o cálculo de HMAC antes de ir para produção (`[09:46] Sofia`).

