# ADR-008: Obrigatoriedade de URLs HTTPS via Validação Zod

* **Status:** Aceito
* **Data:** 2026-09-21
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior de Plataforma), Bruno (Engenheiro Pleno de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Os webhooks transmitem informações de negócios sobre pedidos de clientes B2B através da internet aberta. O tráfego de dados em conexões HTTP não criptografadas em texto claro (`http://`) expõe as notificações a riscos severos de interceptação de dados, vazamento de assinaturas HMAC e ataques do tipo *man-in-the-middle* (MITM).

Precisamos impor a obrigatoriedade do protocolo seguro SSL/TLS (HTTPS) para todos os endpoints de webhook cadastrados no OMS.

---

## Decisão

Decidimos instituir a **Obrigatoriedade Estrita de URLs com Esquema `https://`**, validada diretamente na camada de entrada via schemas **Zod**.

1. **Validação na Camada de Schema (Zod):** No momento do cadastro (`POST /webhooks`) ou atualização (`PATCH /webhooks/:id`), o esquema Zod (`webhook.schemas.ts`) aplicará a regra `.url().refine((url) => url.startsWith('https://'))`.
2. **Rejeição Imediata:** Caso o cliente tente cadastrar um endpoint utilizando o protocolo inseguro `http://`, a requisição será bloqueada imediatamente no middleware de validação com status **HTTP 400 Bad Request** e código de erro **`WEBHOOK_INVALID_URL`**.
3. **Sem Exceções em Produção:** Não serão permitidas exceções para URLs HTTP, garantindo a criptografia de ponta a ponta na transmissão dos webhooks.

---

## Alternativas Consideradas

### 1. Permitir HTTP com Aviso/Warning em Log
* **Descrição:** Permitir o cadastro de URLs `http://`, emitindo apenas um alerta de segurança nos logs do sistema.
* **Motivo do Descarte:** Violaria os requisitos básicos de segurança da informação (exigidos pela Engenheira de Segurança Sofia). Conexões HTTP desprotegidas permitiriam que bisbilhoteiros de rede interceptassem o cabeçalho `X-Signature` e o payload dos pedidos.

---

## Consequências

### Positivas
* **Garantia de Criptografia em Trânsito:** Assegura que 100% do tráfego de webhooks outbound seja transmitido sobre canais criptografados com TLS (HTTPS).
* **Bloqueio Precoce:** Impede que dados inseguros entrem no banco de dados, falhando imediatamente na camada de validação Zod antes de atingir o controller ou service.

### Negativas / Trade-offs
* **Requisito Técnico para o Cliente:** Exige que todos os clientes B2B possuam certificado SSL/TLS válido instalado em seus servidores receptores.
