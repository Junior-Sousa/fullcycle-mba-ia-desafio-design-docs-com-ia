# ADR-001: Padrão Outbox no MySQL para Notificação de Webhooks

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

A aplicação atual é um Order Management System (OMS) construído em Node.js com TypeScript, MySQL e Prisma. Atualmente, os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) realizam requisições periódicas via `GET /orders` (polling) para acompanhar mudanças no status dos seus pedidos, gerando sobrecarga desnecessária na infraestrutura e latência na atualização dos dados.

Foi identificada a necessidade de notificar esses clientes em tempo real (latência aceitável < 10 segundos) sempre que houver alteração de status em seus pedidos. No entanto, o processo de alteração de status (`OrderService.changeStatus`) já executa uma transação de banco de dados pesada que atualiza a tabela `orders`, registra o histórico em `order_status_history` e ajusta o estoque de produtos (`stock_quantity`). 

Precisamos de uma solução para capturar as mudanças de status dos pedidos e disparar as notificações HTTP externas garantindo consistência de dados, isolamento transacional e resiliência, sem comprometer a performance da API principal de pedidos.

---

## Decisão

Decidimos adotar o **Padrão Outbox (Transactional Outbox Pattern)** utilizando o banco de dados MySQL existente.

1. **Gravação Atômica:** Sempre que o status de um pedido for alterado no método `OrderService.changeStatus`, o registro do evento de webhook será inserido na nova tabela `webhook_outbox` dentro da **mesma transação SQL** Prisma que atualiza o pedido e o histórico.
2. **Consistência Garantida:** Se a transação principal der `commit`, a notificação estará garantidamente registrada na outbox; se a transação der `rollback` (por exemplo, por falta de estoque), a notificação também sofrerá rollback, eliminando qualquer risco de inconsistência de dados.
3. **Payload Persistido (Snapshot):** O evento gravado na `webhook_outbox` conterá o snapshot completo do payload formatado no momento da mudança de status, prevenindo alterações futuras no pedido de afetarem a notificação.
4. **Indexação:** A tabela `webhook_outbox` contará com índices compostos em `(status, created_at)` para otimizar as consultas batch do worker consumidor.

---

## Alternativas Consideradas

### 1. Disparo de Webhook Síncrono no `OrderService.changeStatus`
* **Descrição:** Realizar a chamada HTTP para a URL do cliente diretamente dentro do serviço de pedidos durante a execução do `changeStatus`.
* **Motivo do Descarte:** A chamada HTTP síncrona bloquearia a transação do MySQL. Se o servidor do cliente estivesse lento ou indisponível, a alteração de status do pedido no OMS travaria ou falharia. Além disso, uma falha HTTP não deve causar `rollback` em um pedido que já mudou de status legitimamente no domínio da aplicação.

### 2. Uso de Broker de Mensageria Externo (Redis Streams / RabbitMQ)
* **Descrição:** Publicar um evento de mudança de status em uma fila Redis Streams ou RabbitMQ para consumo por um serviço de webhooks.
* **Motivo do Descarte:** Adicionaria complexidade operacional e custos de infraestrutura adicionais (gerenciamento e monitoramento de um cluster Redis/RabbitMQ) para uma equipe de engenharia enxuta. Além disso, publicar diretamente em uma fila externa sem passar por outbox no MySQL reintroduziria o problema de *dual-write* (o banco confirma a alteração mas a publicação na fila falha ou vice-versa).

---

## Consequências

### Positivas
* **Consistência Transacional Estrita:** Inexistência de falso-positivo ou evento perdido, pois a alteração do pedido e a notificação estão amarradas na mesma transação ACID do MySQL.
* **Simplicidade de Infraestrutura:** Não exige provisionamento de novas ferramentas ou clusters de mensageria (Redis, RabbitMQ, Kafka), reaproveitando a infraestrutura MySQL + Prisma já existente.
* **Isolamento de Performance:** A API de pedidos não aguarda a resposta HTTP dos clientes B2B, liberando a conexão de banco de dados imediatamente após o `commit`.

### Negativas / Trade-offs
* **Crescimento da Tabela MySQL:** A tabela `webhook_outbox` acumulará registros ao longo do tempo. Será necessário instituir uma política futura de purge/arquivamento de eventos entregues com mais de 30 dias (fora do escopo da fase atual).
* **I/OAdicional no Banco:** Inserções e leituras adicionais no MySQL para cada alteração de status. O impacto é mitigado pelo uso de batching pequeno e índices adequados.
