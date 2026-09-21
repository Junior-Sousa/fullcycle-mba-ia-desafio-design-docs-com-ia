# ADR-002: Worker em Processo Separado com Polling de 2 Segundos

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Após definirmos o padrão Outbox no MySQL (ver [ADR-001](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-001-outbox-no-mysql.md)), precisamos decidir o mecanismo de consumo dos eventos gravados na tabela `webhook_outbox` para envio das requisições HTTP aos endpoints dos clientes B2B.

As premissas estabelecidas pelo produto exigem que os clientes recebam a notificação com latência inferior a 10 segundos. Além disso, a execução da entrega não pode impactar o ciclo de vida da API web HTTP do OMS (`src/server.ts`) nem ser interrompida por deploys ou reinicializações da API HTTP.

---

## Decisão

Decidimos implementar um **Worker dedicado rodando em processo Node.js separado**, utilizando **Polling contínuo com intervalo de 2 segundos**.

1. **Processo Separado:** Criaremos uma nova entrada de execução no projeto em `src/worker.ts` (executada via script `npm run worker`). O worker compartilhará as mesmas configurações de banco de dados (`DATABASE_URL`) e dependências do projeto, mas possuirá sua própria instância de `PrismaClient` e seu próprio ciclo de vida de processo.
2. **Polling de 2 Segundos:** O worker executará um loop de polling a cada 2 segundos. A cada ciclo, buscará um batch pequeno de eventos com status `PENDING` na tabela `webhook_outbox`, ordenados por `created_at ASC`.
3. **Despacho e Atualização de Status:** Para cada evento do batch, o worker identificará as URLs cadastradas e ativas para aquele cliente, efetuará o disparo HTTP e atualizará o status do registro na `webhook_outbox` (para `DELIVERED` ou agendamento de retry).
4. **Garantia de Ordenação por Pedido:** Na fase inicial, a execução será de **Worker Único (Single-worker)**, garantindo a ordenação sequencial dos disparos por `created_at` para cada pedido (`order_id`).

---

## Alternativas Consideradas

### 1. Triggers Nativas do MySQL para Notificação Externa
* **Descrição:** Utilizar triggers no banco de dados para disparar notificações em tempo real após a gravação na outbox.
* **Motivo do Descarte:** O MySQL não possui suporte nativo ao padrão de escuta/notificação assíncrona (como o `LISTEN/NOTIFY` do PostgreSQL). Implementar chamadas externas via triggers exigiria gambiarras como escrita em disco ou bibliotecas C não padrão, tornando a arquitetura frágil e de difícil manutenção.

### 2. Executar o Loop do Worker no Mesmo Processo da API Express (`src/server.ts`)
* **Descrição:** Iniciar um `setInterval` ou loop assíncrono dentro da própria aplicação web Express.
* **Motivo do Descarte:** Se a API web for reiniciada (por deploy, crash ou healthcheck da infraestrutura), o processamento de webhooks será interrompido. Além disso, o worker disputaria o Event Loop do Node.js com as requisições HTTP recebidas pelos clientes do OMS, violando o princípio de isolamento de responsabilidades.

---

## Consequências

### Positivas
* **Isolamento de Falhas e Deploy Independente:** A API HTTP e o Worker de webhooks podem ser reiniciados, atualizados ou escalados independentemente. Um crash na API não derruba o processamento dos webhooks.
* **Atendimento ao SLA de Latência:** Um polling de 2 segundos garante latência máxima de entrega em torno de 2 a 3 segundos no pior caso, atendendo com folga o requisito de negócio (< 10 segundos).
* **Ordenação Sequencial Garantida:** A utilização de um único processo worker processando em ordem cronológica de inserção previne race conditions na ordem de entrega de status para um mesmo pedido.

### Negativas / Trade-offs
* **Queries Periódicas no Banco (Polling Overhead):** Mesmo quando não houver novos eventos, a cada 2 segundos o worker executará uma consulta `SELECT` na tabela `webhook_outbox`. O impacto é irrisório devido ao índice no campo `status`.
* **Escalabilidade Horizontal Futura:** Se no futuro houver necessidade de rodar múltiplos workers em paralelo para lidar com alto volume, será necessário implementar *pessimistic locking* (`FOR UPDATE SKIP LOCKED`) ou particionamento por `order_id` para evitar processamento duplicado ou fora de ordem.
