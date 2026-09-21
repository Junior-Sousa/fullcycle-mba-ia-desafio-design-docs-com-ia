# ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Em integrações via Webhooks outbound, os servidores dos clientes B2B podem apresentar instabilidades temporárias, lentidão, erros HTTP 5xx, timeouts de rede ou falhas por janelas de manutenção programada (que podem durar até 2 horas).

Precisamos definir uma estratégia de resiliência que realize retentativas automáticas inteligentes em caso de falha de entrega, sem sobrecarregar os servidores dos clientes nem manter eventos pendentes infinitamente. Além disso, quando todas as tentativas se esgotarem, o sistema deve registrar a falha de forma auditável e permitir reprocessamento manual administrativo.

---

## Decisão

Decidimos adotar uma **Política de 5 Tentativas com Backoff Exponencial** e fallback para uma **Dead Letter Queue (DLQ) Persistida no MySQL**.

### 1. Política de Retry e Backoff Exponencial
* **Número de Tentativas:** Máximo de **5 tentativas** de entrega por evento.
* **Progressão dos Intervalos (Backoff):**
  * 1ª retentativa: **1 minuto** após a 1ª falha
  * 2ª retentativa: **5 minutos** após a 2ª falha
  * 3ª retentativa: **30 minutos** após a 3ª falha
  * 4ª retentativa: **2 horas** após a 4ª falha
  * 5ª retentativa: **12 horas** após a 5ª falha
* **Janela Total de Cobertura:** Aproximadamente **15 horas** acumuladas do momento da primeira falha até a tentativa final.

### 2. Dead Letter Queue (DLQ) Persistida
* Caso a 5ª tentativa falhe, o evento deixa de ser retentado automaticamente e é movido para a tabela **`webhook_dead_letter`**.
* A tabela DLQ armazenará a cópia do payload, o ID da configuração do webhook, a contagem de tentativas, o último código de status HTTP/erro recebido, a mensagem de erro e os timestamps.
* A persistência em tabela dedicada mantém a tabela principal `webhook_outbox` limpa e serve como evidência para diagnóstico e auditoria.

### 3. Replay Manual de DLQ
* Criaremos o endpoint administrativo **`POST /admin/webhooks/dead-letter/:id/replay`**.
* O endpoint permitirá que operadores administradores (requer função `ADMIN` no token JWT) re-enfileirem o evento com falha de volta para a `webhook_outbox` com status `PENDING`, zerando a contagem de retentativas.
* Cada ação de replay gerará um log estruturado contendo o ID do usuário administrativo que executou o comando.

---

## Alternativas Consideradas

### 1. Retry Agressivo com Poucas Tentativas (ex: 3 tentativas em 30 minutos)
* **Descrição:** Executar 3 tentativas com intervalos curtos (ex: 1m, 5m, 15m) e mover para DLQ.
* **Motivo do Descarte:** Clientes B2B possuem janelas de manutenção planejada que duram até 2 horas. Três tentativas em 30 minutos esgotariam as retentativas antes da manutenção do cliente terminar, gerando descarte prematuro e volume excessivo de chamadas para o time de suporte.

### 2. Retry Indefinito com Backoff Fixo
* **Descrição:** Continuar tentando enviar a notificação indefinidamente até que o cliente responda HTTP 200.
* **Motivo do Descarte:** Se o cliente desativar um endpoint ou mudar de domínio sem nos avisar, o worker acumularia milhares de chamadas inúteis diariamente para endpoints zumbis, consumindo recursos do banco e do worker indefinidamente.

---

## Consequências

### Positivas
* **Resiliência a Manutenções Prolongadas:** A janela de 15 horas cobre a grande maioria das indisponibilidades operacionais e manutenções noturnas de sistemas parceiros.
* **Proteção de Recursos:** O backoff crescente reduz o tráfego em direção a um servidor cliente que esteja instável, evitando o efeito *thundering herd*.
* **Auditabilidade e Recuperabilidade:** A DLQ em tabela separada fornece histórico claro de falhas e permite recuperação simples via API sem intervenção direta no banco de dados.

### Negativas / Trade-offs
* **Necessidade de Tabela e Endpoint Adicionais:** Requer desenvolvimento da tabela `webhook_dead_letter` e das rotas de gerenciamento administrativo (`POST /replay`).
* **Intervenção Manual após 15h:** Falhas permanentes que excedam 15 horas exigirão ação manual do time operacional após o cliente corrigir seu ambiente.
