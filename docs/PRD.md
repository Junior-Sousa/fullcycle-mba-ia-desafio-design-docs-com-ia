# PRD: Product Requirement Document — Sistema de Webhooks de Notificação de Pedidos

* **Autor:** Marcos (Product Manager)
* **Status:** Aprovado
* **Data:** 2026-09-21
* **Versão:** 1.0.0

---

## 1. Resumo e Contexto da Feature

O **Sistema de Webhooks de Notificação de Pedidos** é uma nova funcionalidade da plataforma de Order Management System (OMS) projetada para notificar clientes B2B ativamente e em tempo real sobre atualizações no ciclo de vida dos seus pedidos. 

Atualmente, o OMS suporta apenas consultas ativas via API REST (`GET /orders`). O sistema de webhooks preencherá a lacuna de comunicação orientada a eventos (*outbound webhooks*), permitindo que sistemas externos de ERP e logística dos nossos clientes recebam notificações imediatas sempre que um pedido mudar de status (por exemplo, de `PENDING` para `PAID`, `SHIPPED` ou `DELIVERED`).

---

## 2. Problema e Motivação

### Problema Atual
Clientes B2B de grande porte (como Atlas Comercial, MaxDistribuição e Nova Cargo) precisam manter seus sistemas internos sincronizados com o OMS. Na ausência de notificações passivas, eles realizam requisições HTTP repetitivas (`polling`) no endpoint `GET /orders` a cada poucos segundos.

### Impactos Negativos
* **Sobrecarga na Infraestrutura:** Milhares de requisições de polling desnecessárias consumindo CPU, I/O e conexões com o MySQL.
* **Experiência Degradada:** Latência na atualização dos dados no lado do cliente, gerando atrasos em processos logísticos e operacionais.
* **Risco Comercial Imediato:** O cliente **Atlas Comercial** comunicou formalmente que a falta de notificações em tempo real pode resultar na rescisão de contrato e migração para um concorrente ao final do trimestre atual.

---

## 3. Público-Alvo e Cenários de Uso

### Público-Alvo
* **Integradores e Desenvolvedores B2B:** Equipes de TI de empresas clientes (Atlas Comercial, MaxDistribuição, Nova Cargo) que consomem a API do OMS.
* **Sistemas de ERP e WMS:** Plataformas parceiras de gestão de estoque e logística que necessitam de gatilhos automáticos para despacho e faturamento.

### Cenários de Uso
1. **Confirmação de Pagamento:** O pedido muda para `PAID`. O ERP do cliente recebe o webhook e gera a nota fiscal automaticamente.
2. **Atualização Logística:** O pedido muda para `SHIPPED`. O sistema de rastreamento do cliente atualiza o status de entrega para o consumidor final.
3. **Cancelamento por Falta de Estoque:** O pedido muda para `CANCELLED`. O sistema do cliente é notificado imediatamente e libera a reserva financeira.

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo de Negócio / Técnico | Métrica de Sucesso | Meta Quantitativa |
| :--- | :--- | :--- |
| **Garantir Notificação em Tempo Real** | Tempo transcorrido entre o commit no OMS e o recebimento no cliente (SLA) | **< 10 segundos** em 99,5% das entregas |
| **Eliminar Polling Excessivo** | Redução no volume de chamadas de leitura `GET /orders` dos clientes parceiros | **Redução de no mínimo 80%** no tráfego de polling |
| **Garantir Integridade Transacional** | Taxa de disparos de eventos perdidos por falhas internas do OMS | **0% de perda** (100% de consistência entre status do pedido e outbox) |
| **Retenção de Clientes B2B** | Manutenção do contrato com os clientes chaves (Atlas Comercial, MaxDistribuição, Nova Cargo) | **100% de retenção** dos 3 clientes B2B |

---

## 5. Escopo

### Dentro do Escopo (Incluso)
* Cadastro, listagem, edição e remoção de configurações de webhooks por cliente B2B.
* Seleção de quais status de pedido (`PAID`, `PROCESSING`, `SHIPPED`, etc.) cada webhook deseja assinar.
* Disparo assíncrono de notificações HTTP via Padrão Outbox no MySQL.
* Autenticação criptográfica de cada mensagem via cabeçalho `X-Signature` (HMAC-SHA256).
* Suporte a rotação de *secret* com período de carência (*grace period*) de 24 horas.
* Política de resiliência com 5 retentativas e backoff exponencial cobrindo ~15 horas.
* Redirecionamento de falhas para Dead Letter Queue (DLQ) com rota de replay administrativo.
* Endpoint para consulta de histórico dos últimos 100 envios de webhook.

### Fora do Escopo (Explicitamente Descartado / Adiado)
1. **Painel Visual / Dashboard no Frontend:** O gerenciamento de webhooks será realizado exclusivamente via API REST. A construção de uma interface visual no painel do cliente foi **descartada nesta fase** e delegada ao time de frontend para ciclo futuro.
2. **Alertas Automáticos por E-mail:** O envio de e-mails avisando o cliente quando seu webhook falhar repetidamente foi **explicitamente adiado para fases futuras**, após medirmos o impacto e estabilidade do sistema em produção.
3. **Garantia de Ordenação Global:** Não haverá garantia de ordenação estrita entre múltiplos pedidos simultâneos, garantindo-se apenas ordenação sequencial por pedido (`order_id`) enquanto for single-worker.
4. **Recebimento de Webhooks (Inbound):** O sistema tratará apenas da saída de notificações (*outbound*).

---

## 6. Requisitos Funcionais

* **RF-01 (Cadastro de Webhook):** O cliente deve conseguir cadastrar um endpoint de webhook informando a URL (exclusivamente `https://`), a lista de eventos de status de pedido que deseja ouvir e a identificação do cliente.
* **RF-02 (Geração de Secret):** O sistema deve gerar automaticamente uma chave secreta (*secret*) única por endpoint com alta entropia no momento da criação.
* **RF-03 (Rotação de Secret com Grace Period):** O cliente deve conseguir solicitar a rotação da chave secreta. A chave antiga deve permanecer válida por 24 horas em paralelo para evitar indisponibilidade.
* **RF-04 (Filtro de Inserção na Outbox):** O sistema só deve criar registros na outbox se houver pelo menos um webhook ativo do cliente inscrito no novo status do pedido, economizando linhas no banco.
* **RF-05 (Gestão e Listagem de Webhooks):** O sistema deve permitir listar todos os webhooks ativos de um cliente e desativar/deletar um webhook cadastrado.
* **RF-06 (Consulta a Histórico de Entregas):** O cliente deve conseguir consultar o histórico de envios do seu webhook (`GET /webhooks/:id/deliveries`), incluindo o código de resposta HTTP recebido, tempo de execução e status.
* **RF-07 (Gravacão Atômica em Transação):** A gravação do evento na outbox deve ocorrer dentro da mesma transação SQL que altera o status do pedido em `OrderService.changeStatus`.
* **RF-08 (Replay Manual de DLQ por ADMIN):** Usuários com papel `ADMIN` devem conseguir re-enfileirar eventos da DLQ para a outbox através de um endpoint seguro com log de auditoria.
* **RF-09 (Assinatura HMAC-SHA256):** Toda notificação enviada deve conter o cabeçalho `X-Signature` contendo o digest hexadecimal da assinatura HMAC-SHA256 do payload.
* **RF-10 (Header X-Event-Id para Idempotência):** Toda notificação deve enviar um UUID único no cabeçalho `X-Event-Id` para permitir desduplicação pelo cliente.

---

## 7. Requisitos Não Funcionais

* **RNF-01 (Latência de Despacho):** As notificações devem ser disparadas e entregues em menos de 10 segundos após a alteração do pedido no banco de dados.
* **RNF-02 (Isolamento de Processo):** O worker de consumo da outbox (`src/worker.ts`) deve rodar em um processo Node.js dedicado, sem compartilhar o mesmo ciclo de vida da API Express.
* **RNF-03 (Protocolo Seguro TLS):** O cadastro de URLs sem suporte a criptografia SSL/TLS (`http://`) deve ser obrigatoriamente recusado.
* **RNF-04 (Teto de Payload):** Notificações cujo payload JSON ultrapasse 64 KB devem ser abortadas com erro de validação.
* **RNF-05 (Timeout de Conexão):** O worker deve aguardar no máximo 10 segundos pela resposta HTTP do cliente antes de registrar timeout e agendar retentativa.
* **RNF-06 (Resiliência):** O sistema deve efetuar até 5 retentativas com intervalos de 1m, 5m, 30m, 2h e 12h antes de mover para DLQ.

---

## 8. Decisões e Trade-offs Principais

* **Outbox no MySQL vs. Broker Externo:** Optou-se por utilizar a tabela Outbox no MySQL existente para evitar complexidade operacional e garantir transações ACID sem o problema de *dual-write*.
* **Single-Worker Polling vs. Triggers:** Adotou-se worker em processo separado com polling de 2s para garantir ordenação por pedido e evitar dependências de recursos não nativos do MySQL.
* **At-Least-Once vs. Exactly-Once:** Escolheu-se a garantia *At-Least-Once* com `X-Event-Id` por ser o padrão de indústria factível e robusto para integrações HTTP assíncronas.

---

## 9. Dependências

* **Dependências de Software:** Node.js v18+, MySQL 8.0+, Prisma ORM, Express.js, Zod, Pino Logger.
* **Dependência Organizacional:** Revisão de segurança de 2 dias úteis conduzida pela engenheira de segurança (Sofia) antes da implantação em produção.
* **Prazo de Entrega:** Desenvolver e testar em 3 sprints (previsão de entrega para fim de novembro).

---

## 10. Riscos e Mitigação

| Risco Identificado | Probabilidade | Impacto | Estratégia de Mitigação |
| :--- | :--- | :--- | :--- |
| **Instabilidade / Indisponibilidade nos Servidores dos Clientes B2B** | Alta | Médio | Política de backoff exponencial em 5 camadas (até 15h) e direcionamento automático para DLQ com replay manual. |
| **Lentidão em Requisições HTTP do Cliente Bloqueando o Worker** | Média | Alto | Timeout estrito de 10 segundos por chamada HTTP e consumo em lote isolado. |
| **Vazamento da Secret do Webhook pelo Cliente** | Média | Alto | Chave única por endpoint, prefixo identificável `whsec_`, armazenamento seguro e rotação com carência de 24h. |
| **Acúmulo de Linhas na Tabela `webhook_outbox`** | Média | Baixo | Criação de índices em `(status, created_at)` e planejamento de rotina de purge após 30 dias. |

---

## 11. Critérios de Aceitação

1. O cliente consegue criar, listar, atualizar e deletar webhooks via API autenticada.
2. A alteração de status do pedido gera o evento correspondente na `webhook_outbox` em até 2 segundos.
3. O cliente recebe o evento com os cabeçalhos `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature` válidos.
4. Caso o servidor do cliente retorne erro 5xx, o worker realiza até 5 tentativas com os intervalos especificados.
5. Um evento que falha 5 vezes é movido para a `webhook_dead_letter` e pode ser reenviado via `POST /admin/webhooks/dead-letter/:id/replay` por um usuário `ADMIN`.
6. Tentar cadastrar uma URL `http://` falha com mensagem de erro amigável e código `WEBHOOK_INVALID_URL`.

---

## 12. Estratégia de Testes e Validação

* **Testes Unitários:** Testar a função de geração de assinaturas HMAC-SHA256, cálculo de backoff exponencial e validações de schemas Zod.
* **Testes de Integração:** Simular transações de alteração de pedido no `OrderService` e validar a criação simultânea dos registros na outbox.
* **Testes Ponta a Ponta (E2E):** Subir a API e o worker com um servidor HTTP mock (ex: MSW ou Fastify mock) e validar o disparo completo, retry e idas para DLQ.
* **Auditoria de Segurança:** Revisão de código de 2 dias úteis realizada pela equipe de segurança para validar a criptografia HMAC e isolamento de secrets.
