# ADR-004: Autenticação HMAC-SHA256 com Secret Única por Endpoint e Suporte a Rotação

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Como as notificações de webhook são enviadas através da internet aberta para endpoints HTTP pertencentes aos clientes B2B, existe o risco de ataques de falsificação (*spoofing*), interceptação ou adulteração do payload por terceiros (*man-in-the-middle*). 

Os clientes precisam ter um meio criptográfico confiável para validar que:
1. A requisição HTTP foi efetivamente gerada pelo nosso OMS.
2. O corpo (payload) da notificação não foi alterado no trajeto.

Além disso, requisitos de segurança exigem que o vazamento da chave de um cliente não comprometa a segurança de outros clientes, e que seja possível rotacionar chaves sem causar indisponibilidade no recebimento das notificações.

---

## Decisão

Decidimos implementar o padrão de **Assinatura Digital HMAC-SHA256 com Secret Criptográfica Única por Endpoint e Rotação com Grace Period**.

### 1. Algoritmo e Assinatura
* Cada requisição de webhook conterá o cabeçalho **`X-Signature`**.
* O valor do cabeçalho será o código hash hexadecimal gerado por HMAC-SHA256 do corpo em texto (stringified JSON) da requisição utilizando a *secret* do webhook do cliente:  
  `X-Signature = hmacHex(sha256, secret, payloadBody)`

### 2. Isolation & Secret por Endpoint
* A chave secreta (*secret*) será gerada aleatoriamente no momento do cadastro do webhook (com prefixo `whsec_`) e associada exclusivamente a **um único endpoint**.
* Não haverá *secret* global compartilhada da plataforma.

### 3. Rotação de Secret com Grace Period de 24 Horas
* Disponibilizaremos um endpoint para rotação da chave secreta.
* Quando uma chave for rotacionada, a chave antiga permanecerá válida por um **período de carência (grace period) de 24 horas**. Durante esse intervalo, o worker enviará ambas as assinaturas (ou o cliente poderá aceitar a validação por qualquer uma das duas chaves válidas). Após 24 horas, a chave antiga é revogada permanentemente.

### 4. Requisito de TLS Obligatório
* O cadastro de URLs de webhook exigirá estritamente o protocolo seguro **`https://`**. URLs utilizando `http://` serão rejeitadas na camada de validação de esquemas (Zod).

---

## Alternativas Consideradas

### 1. Secret Global da Plataforma
* **Descrição:** Utilizar uma única chave secreta para assinar as notificações de todos os clientes da plataforma OMS.
* **Motivo do Descarte:** Se a chave de um único cliente vazasse em seus logs de aplicação, toda a infraestrutura de notificações de todos os clientes B2B estaria comprometida.

### 2. Autenticação por Token Fixo no Header (Bearer Token / API Key Estática)
* **Descrição:** O cliente cadastra uma API Key estática e o OMS a reenvia no cabeçalho `Authorization: Bearer <key>`.
* **Motivo do Descarte:** O token estático exposto no cabeçalho pode ser interceptado e não garante a integridade do payload (qualquer invasor com o token poderia forjar um evento válido). O HMAC valida simultaneamente a autoria e a integridade da mensagem.

---

## Consequências

### Positivas
* **Segurança de Padrão de Mercado:** Segue as melhores práticas da indústria (utilizado por Stripe, GitHub, Shopify).
* **Defesa em Profundidade:** Garante a autenticidade e a integridade dos payloads enviados.
* **Zero Downtime na Rotação:** O *grace period* de 24h permite que os clientes atualizem a chave em seus servidores sem perder notificações nem apresentar falhas de validação.

### Negativas / Trade-offs
* **Complexidade no Cliente:** Os clientes B2B precisarão implementar o código de verificação HMAC-SHA256 no seu receptor HTTP (a ser facilitado por documentação e exemplos claros no portal de desenvolvedores).
* **Armazenamento Seguro de Secrets:** As secrets dos clientes devem ser armazenadas de forma segura no banco de dados e protegidas contra vazamentos em logs internos.
