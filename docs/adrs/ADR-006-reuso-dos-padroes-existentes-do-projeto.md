# ADR-006: Reuso dos Padrões Arquiteturais e Estruturais Existentes do Projeto

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

A aplicação base possui uma estrutura bem estabelecida e padronizada para modularização de código, tratamento de erros, validação de requisições, registro de logs e controle de acesso. 

Para manter a consistência, manutenibilidade e curva de aprendizado da equipe, a nova funcionalidade de **Sistema de Webhooks de Notificação de Pedidos** não deve introduzir novas bibliotecas redundantes ou abstrações conflitantes, mas sim se integrar harmonicamente com os padrões existentes na codebase.

---

## Decisão

Decidimos reaproveitar integralmente as abstrações, padrões de projeto e estruturas de pastas já utilizadas no projeto.

### 1. Estrutura Modular por Domínio
* Criaremos um novo módulo isolado em **`src/modules/webhooks/`**, mantendo a mesma separação de responsabilidades observada nos módulos existentes (`src/modules/orders/`, `src/modules/customers/`, `src/modules/auth/`):
  * `webhook.controller.ts`: Manipulação das requisições HTTP e respostas.
  * `webhook.service.ts`: Regras de negócio de webhooks.
  * `webhook.repository.ts`: Abstração do Prisma Client para entidades de webhook.
  * `webhook.routes.ts`: Definição de rotas Express.
  * `webhook.schemas.ts`: Schemas de validação Zod para payloads.
  * `webhook.worker.ts`: Processador de background chamado pelo ponto de entrada `src/worker.ts`.

### 2. Tratamento de Erros e Códigos com Prefixo `WEBHOOK_`
* Reaproveitaremos a classe base **`AppError`** localizada em [`src/shared/errors/app-error.ts`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts).
* Criaremos exceções de domínio estendendo `AppError` e adotaremos o padrão de códigos de erro em maiúsculo com o prefixo **`WEBHOOK_`** (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`).
* O middleware centralizado de erros em [`src/middlewares/error.middleware.ts`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts) capturará automaticamente essas exceções, garantindo respostas HTTP formatadas e estruturadas.

### 3. Log Estruturado com Pino
* Utilizaremos o logger singleton Pino configurado em [`src/shared/logger/index.ts`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts) tanto nas rotas da API quanto no processo worker (`src/worker.ts`), garantindo logs estruturados em JSON para observabilidade.

### 4. Autenticação e Controle de Acesso
* Reaproveitaremos os middlewares de autenticação JWT e autorização RBAC definidos em [`src/middlewares/auth.middleware.ts`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts) (`requireAuth` e `requireRole`).
* Os endpoints CRUD de webhook utilizarão `requireAuth`, enquanto rotas administrativas críticas (como o replay de DLQ) utilizarão `requireRole(UserRole.ADMIN)`.

### 5. Integração com Transações do Prisma
* Estenderemos a assinatura do método `changeStatus` em [`src/modules/orders/order.service.ts`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts) injetando a gravação na outbox dentro da transação `Prisma.TransactionClient` existente, mantendo o padrão de modelagem com UUIDs e convenção de tabelas `snake_case` do [`prisma/schema.prisma`](file:///Users/macbookpro/github/fullcycle-mba-ia-desafio-design-docs-com-ia/prisma/schema.prisma).

---

## Alternativas Consideradas

### 1. Introduzir uma Biblioteca Externa de Filas / Worker (ex: BullMQ / Agenda)
* **Descrição:** Adicionar bibliotecas de terceiros para gerenciar o agendamento de jobs.
* **Motivo do Descarte:** Adicionaria dependências pesadas e desnecessárias ao `package.json`, além de exigir um servidor Redis. O padrão de polling simples com `PrismaClient` atende perfeitamente os requisitos sem inflar o projeto.

---

## Consequências

### Positivas
* **Manutenibilidade e Consistência:** Desenvolvedores familiarizados com o modulo de `orders` ou `customers` entenderão imediatamente a estrutura do módulo `webhooks`.
* **Zero Código Duplicado:** Aproveitamento de tratamento de erros, validação Zod e logs existentes sem necessidade de criar novos middlewares ou bibliotecas.
* **Sem Regressões:** O código existente continua intacto e as novas funcionalidades se acoplam de forma transparente.

### Negativas / Trade-offs
* Nenhuma consequência negativa identificada. O alinhamento completo com os padrões da codebase atual é uma boa prática fundamental de engenharia de software.
