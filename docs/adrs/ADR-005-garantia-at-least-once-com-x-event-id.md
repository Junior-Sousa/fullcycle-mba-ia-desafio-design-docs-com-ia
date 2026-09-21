# ADR-005: Garantia de Entrega At-Least-Once com Header X-Event-Id para Idempotência

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Em sistemas distribuídos de notificação por webhook, oscilações de rede, respostas HTTP com timeout (onde o cliente processou a notificação mas a resposta HTTP 200 não chegou a tempo ao worker) ou retentativas automáticas podem resultar no reenvio de uma notificação que o cliente já recebeu.

Precisamos definir o modelo de garantia de entrega do sistema de webhooks do OMS e fornecer aos clientes a capacidade de processar os eventos recebidos de forma **idempotente**, prevenindo efeitos colaterais como duplo processamento de atualização de estoque ou registros duplicados no lado do cliente.

---

## Decisão

Decidimos adotar a garantia de entrega **At-Least-Once (Pelo Menos Uma Vez)** combinada com um identificador único por evento transmitido via o cabeçalho HTTP **`X-Event-Id`**.

1. **Garantia At-Least-Once:** O OMS garante que todo evento confirmado na transação do pedido será entregue *pelo menos uma vez* ao endpoint do cliente. Em cenários de falha parcial de rede ou retry, o mesmo evento poderá ser enviado mais de uma vez.
2. **Identificador Único por Evento (`X-Event-Id`):** No momento em que a linha do evento é criada na tabela `webhook_outbox`, o sistema gerará um identificador único no formato UUID v4.
3. **Transmissão via Header:** Cada disparo HTTP enviará o ID do evento no cabeçalho **`X-Event-Id`**.
4. **Idempotência no Cliente:** O cliente B2B será orientado a armazenar os `X-Event-Id` processados recentemente e desduplicar requisições recebidas com o mesmo ID.

---

## Alternativas Consideradas

### 1. Garantia Exactly-Once (Exatamente Uma Vez)
* **Descrição:** Garantir que o cliente receberá cada notificação exatamente uma única vez, sem a possibilidade de duplicatas sob nenhuma circunstância.
* **Motivo do Descarte:** A garantia *Exactly-Once* em comunicações HTTP com sistemas externos é teoricamente e praticamente inviável sem protocolo de consenso bidirecional de duas fases (2PC) ou acoplamento forte entre o OMS e o servidor do cliente. Forçar essa garantia aumentaria a complexidade do sistema exponencialmente sem ganhos reais de confiabilidade.

---

## Consequências

### Positivas
* **Confiabilidade da Entrega:** Garante que nenhuma notificação será perdida em decorrência de quedas de conexão intermediárias.
* **Padrão de Mercado:** Segue o padrão utilizado pelas maiores APIs de webhooks do mercado (Stripe, GitHub, Twilio, SendGrid).
* **Desduplicação Simples:** Fornece um mecanismo claro e direto (`X-Event-Id`) para que os receptores dos clientes filtrem duplicatas em memória ou cache (ex: Redis com TTL).

### Negativas / Trade-offs
* **Responsabilidade Compartilhada:** Exige que a equipe de integração do cliente B2B implemente o controle de idempotência do seu lado. Isso será devidamente documentado no portal de desenvolvedores.
