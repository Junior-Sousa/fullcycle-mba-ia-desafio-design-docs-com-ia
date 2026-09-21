# Matriz de Rastreabilidade (TRACKER)

Este documento estabelece a rastreabilidade cruzada entre cada item registrado nos Design Docs (`PRD.md`, `RFC.md`, `FDD.md` e ADRs) e sua respectiva origem na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte da aplicação (`src/`, `prisma/`).

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PRD-RF-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook com URL HTTPS, eventos e customerId | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-RF-02** | `docs/PRD.md` | Requisito Funcional | Geração automática de secret única por endpoint de webhook | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-RF-03** | `docs/PRD.md` | Requisito Funcional | Rotação de secret com período de carência (grace period) de 24h | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-RF-04** | `docs/PRD.md` | Requisito Funcional | Filtro de eventos na inserção da outbox por status assinado | TRANSCRICAO | `[09:33] Bruno` |
| **PRD-RF-05** | `docs/PRD.md` | Requisito Funcional | Listagem, atualização PATCH e remoção DELETE de webhooks | TRANSCRICAO | `[09:33] Bruno` |
| **PRD-RF-06** | `docs/PRD.md` | Requisito Funcional | Consulta de histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | `[09:34] Marcos` |
| **PRD-RF-07** | `docs/PRD.md` | Requisito Funcional | Inserção atômica na outbox dentro da transação do changeStatus | TRANSCRICAO | `[09:40] Bruno` |
| **PRD-RF-08** | `docs/PRD.md` | Requisito Funcional | Replay manual de DLQ via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | `[09:18] Diego` |
| **PRD-RF-09** | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 enviada no cabeçalho X-Signature | TRANSCRICAO | `[09:20] Sofia` |
| **PRD-RF-10** | `docs/PRD.md` | Requisito Funcional | Inclusão de UUID único no header X-Event-Id para idempotência | TRANSCRICAO | `[09:25] Diego` |
| **PRD-RNF-01** | `docs/PRD.md` | Requisito Não Funcional | SLA de entrega abaixo de 10 segundos (polling de 2s) | TRANSCRICAO | `[09:02] Marcos` |
| **PRD-RNF-02** | `docs/PRD.md` | Requisito Não Funcional | Worker executando como processo separado (src/worker.ts) | TRANSCRICAO | `[09:11] Diego` |
| **PRD-RNF-03** | `docs/PRD.md` | Requisito Não Funcional | Exigência estrita de protocolo HTTPS (validação Zod) | TRANSCRICAO | `[09:23] Sofia` |
| **PRD-RNF-04** | `docs/PRD.md` | Requisito Não Funcional | Limite máximo de tamanho de payload de 64 KB | TRANSCRICAO | `[09:24] Diego` |
| **PRD-RNF-05** | `docs/PRD.md` | Requisito Não Funcional | Timeout estrito de 10 segundos por chamada HTTP | TRANSCRICAO | `[09:42] Diego` |
| **PRD-ESC-01** | `docs/PRD.md` | Restrição / Fora de Escopo | Dashboard visual no frontend fora do escopo (somente API) | TRANSCRICAO | `[09:40] Larissa` |
| **PRD-ESC-02** | `docs/PRD.md` | Restrição / Fora de Escopo | Envio automático de e-mails de alerta adiado para fases futuras | TRANSCRICAO | `[09:37] Larissa` |
| **PRD-RISCO-01** | `docs/PRD.md` | Risco | Instabilidade de clientes mitigada por retry 5x e DLQ | TRANSCRICAO | `[09:15] Diego` |
| **PRD-RISCO-02** | `docs/PRD.md` | Risco | Janela de revisão de segurança de 2 dias antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| **RFC-PROP-01** | `docs/RFC.md` | Arquitetura | Padrão Transactional Outbox no MySQL em vez de mensageria externa | TRANSCRICAO | `[09:06] Diego` |
| **RFC-ALT-01** | `docs/RFC.md` | Trade-off / Alternativa | Descarte de envio síncrono no service por travar transação SQL | TRANSCRICAO | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Trade-off / Alternativa | Descarte de Redis/RabbitMQ por overengineering para time pequeno | TRANSCRICAO | `[09:07] Diego` |
| **RFC-OPEN-01** | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída mantido aberto para observação | TRANSCRICAO | `[09:39] Larissa` |
| **RFC-OPEN-02** | `docs/RFC.md` | Questão em Aberto | Notificação de falhas por e-mail adiada para fase posterior | TRANSCRICAO | `[09:37] Larissa` |
| **FDD-FLUXO-01** | `docs/FDD.md` | Fluxo de Dados | Snapshot do payload formatado gerado na inserção da outbox | TRANSCRICAO | `[09:52] Larissa` |
| **FDD-FLUXO-02** | `docs/FDD.md` | Resiliência | Progressão de backoff exponencial: 1m, 5m, 30m, 2h, 12h | TRANSCRICAO | `[09:17] Diego` |
| **FDD-CONTRATO-01** | `docs/FDD.md` | Contrato HTTP | Headers enviados: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | `[09:44] Diego` |
| **FDD-ERRO-01** | `docs/FDD.md` | Padronização de Erro | Códigos de erro padronizados com prefixo WEBHOOK_* | TRANSCRICAO | `[09:29] Bruno` |
| **FDD-SEC-01** | `docs/FDD.md` | Segurança | Exigência da role ADMIN no endpoint de replay de DLQ | TRANSCRICAO | `[09:36] Sofia` |
| **ADR-001** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Uso do Padrão Outbox no MySQL em transação atômica | TRANSCRICAO | `[09:08] Larissa` |
| **ADR-002** | `docs/adrs/ADR-002-worker-em-processo-separado-em-polling.md` | Decisão | Polling de 2s e worker em processo isolado | TRANSCRICAO | `[09:10] Larissa` |
| **ADR-003** | `docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas de retry, backoff exponencial e tabela DLQ | TRANSCRICAO | `[09:17] Larissa` |
| **ADR-004** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256, secret por endpoint e grace period de 24h | TRANSCRICAO | `[09:22] Sofia` |
| **ADR-005** | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão | Entrega at-least-once com X-Event-Id para idempotência | TRANSCRICAO | `[09:26] Larissa` |
| **ADR-006** | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão | Reuso de AppError, Pino, Zod e estrutura modular | TRANSCRICAO | `[09:30] Larissa` |
| **COD-INT-01** | `docs/FDD.md` | Integração com Código | Extensão do método `changeStatus` em `OrderService` | CODIGO | `src/modules/orders/order.service.ts` |
| **COD-INT-02** | `docs/FDD.md` | Integração com Código | Adição dos novos modelos de webhook no arquivo de modelo Prisma | CODIGO | `prisma/schema.prisma` |
| **COD-INT-03** | `docs/FDD.md` | Integração com Código | Reuso das classes base e tratamento de erros do sistema | CODIGO | `src/shared/errors/app-error.ts` |
| **COD-INT-04** | `docs/FDD.md` | Integração com Código | Tratamento centralizado de erros `WEBHOOK_*` no middleware Express | CODIGO | `src/middlewares/error.middleware.ts` |
| **COD-INT-05** | `docs/FDD.md` | Integração com Código | Proteção RBAC com `requireRole(UserRole.ADMIN)` nas rotas | CODIGO | `src/middlewares/auth.middleware.ts` |
| **COD-INT-06** | `docs/FDD.md` | Integração com Código | Utilização do logger singleton Pino no processo worker | CODIGO | `src/shared/logger/index.ts` |

---

## Estatísticas de Cobertura da Rastreabilidade

* **Total de Itens Mapeados:** 41 itens
* **Itens com Fonte = TRANSCRICAO:** 35 itens (**85,4%** — Requisito do desafio: >= 70%)
* **Itens com Fonte = CODIGO:** 6 itens (**14,6%** — Requisito do desafio: >= 5 linhas)
* **Percentual de Itens Rastreáveis nos Docs:** **100%** (**100%** dos itens declarados possuem origem rastreável — Requisito do desafio: >= 80%)
