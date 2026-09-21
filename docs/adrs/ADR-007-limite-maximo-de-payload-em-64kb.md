# ADR-007: Limite Máximo de Tamanho de Payload em 64KB

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

As notificações de webhook transmitem dados de eventos de mudanças de status de pedidos em formato JSON. Caso ocorra alguma inconsistência ou tentativa de envio de um evento com dados desproporcionalmente grandes (por exemplo, um histórico inflado ou notas extensas de 500KB), a serialização, transmissão HTTP e armazenamento desses payloads geraria alto consumo de memória Node.js, estouro de largura de banda e potencial degradação da performance do worker.

Precisamos estabelecer um limite máximo rígido e seguro para o tamanho do payload de uma notificação de webhook.

---

## Decisão

Decidimos estabelecer o **Limite Máximo de 64 KB (65.536 bytes)** para o tamanho do payload JSON de qualquer notificação de webhook.

1. **Payload Enxuto por Design:** O payload de notificação transmitirá apenas os atributos essenciais do pedido (`orderId`, `orderNumber`, `customerId`, `fromStatus`, `toStatus`, `totalCents`, `updatedAt`), omitindo a lista detalhada de itens para manter o tamanho reduzido.
2. **Rejeição em Caso de Excesso:** Se o payload formatado ultrapassar 64KB, a notificação não será enviada e o sistema registrará a falha com o código de erro **`WEBHOOK_PAYLOAD_TOO_LARGE`**.
3. **Consulta Complementar:** Caso o cliente B2B necessite de detalhes completos sobre os itens do pedido, ele deverá efetuar uma consulta pontual via `GET /orders/:id`.

---

## Alternativas Consideradas

### 1. Truncamento Automático do Payload Extenso
* **Descrição:** Se o payload ultrapassar um tamanho limite, cortar campos longos ou remover notas para forçar a mensagem a caber no limite.
* **Motivo do Descarte:** O truncamento gera JSON parcial ou altera a semântica do evento, podendo quebrar o parser no lado do cliente ou omitir dados críticos de integridade sem que o cliente perceba.

### 2. Sem Limite de Tamanho (Permitir Payloads de Qualquer Tamanho)
* **Descrição:** Permitir o envio de payloads sem restrição de tamanho.
* **Motivo do Descarte:** Exporia o worker a gargalos de I/O, esgotamento de memória no Node.js e riscos de negação de serviço (DoS) caso um evento anomalamente grande entrasse no banco.

---

## Consequências

### Positivas
* **Previsibilidade e Performance:** Garante que todas as chamadas HTTP do worker trafeguem pacotes pequenos, otimizando o I/O de rede e o consumo de memória.
* **Proteção contra Anomalias:** Evita que falhas upstream façam o worker tentar enviar mensagens gigantescas para endpoints de clientes.

### Negativas / Trade-offs
* **Necessidade de GET Adicional:** Clientes que precisarem do detalhamento dos itens do pedido terão que fazer uma requisição complementar `GET /orders/:id` no OMS.
