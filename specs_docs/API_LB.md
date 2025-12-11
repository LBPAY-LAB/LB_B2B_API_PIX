# API LB PIX - Especificação

> **Versão:** 1.0
> **Data:** 2025-01-24
> **Base URL Produção:** `https://api.lbpay.com.br`
> **Base URL Sandbox:** `https://sandbox.lbpay.com.br`

Este documento apresenta a **Especificação da API LB PIX (v1.0)**, um serviço RESTful projetado para gerenciar todas as operações do sistema PIX, incluindo envio, recebimento, cobranças (COB/COBV), QR Codes e recorrências (PIX Automático). A arquitetura detalha **princípios de design REST** e requisitos rigorosos de **segurança (OAuth 2.0 e mTLS)**, além de políticas de resiliência como *rate limiting* e *circuit breaker*. Uma parte significativa do material é dedicada a **conformidade regulatória**, descrevendo como a API adere às regras do Banco Central (BACEN) e à Portaria SPA/ME nº 615/2024, especialmente em relação à **validação de mesma titularidade** e retenção obrigatória de logs por cinco anos. Estão descritas referências técnicas, incluindo **códigos de erro granulares**, esquemas JSON, e especificações de **webhooks** com políticas detalhadas de *retry* e segurança.

## Temas abordados

- Operações PIX completas (envio, recebimento, consultas)
- QR Codes estáticos e dinâmicos
- Cobranças (COB e COBV)
- PIX Automático (recorrências)
- Transferências internas
- Webhooks
- Split de pagamento
- Gestão de chaves PIX

## Temas a serem expandidos

- Portal do Desenvolvedor
  - credenciais de acesso, client secret, geração de token, integração com KeyCloak
- 21.3.5 Monitoramento de Segurança - SIEM
  - captura de estatísticas, logs, dashboard
- 17.2 Limites Temporais
  - limites de cada tipo de requisição (conforme contrato com o cliente)
- 18 SDKs e Bibliotecas
- 21.2.2 Retenção de Logs
- Repositório Auditoria

---

## Sumário Executivo

A **LB PIX API** é uma API RESTful moderna e completa para gestão de operações PIX, inspirada nas melhores práticas das APIs bancárias de mercado. Esta API foi projetada para atender às necessidades do LB Pay, oferecendo:

- **Operações PIX completas**: envio, recebimento, consultas e gestão de chaves
- **QR Codes**: estáticos e dinâmicos, com decodificação e pagamento
- **Recorrências**: PIX Automático e assinaturas
- **Transferências internas**: entre contas de mesma titularidade
- **Webhooks**: notificações assíncronas confiáveis
- **Split de pagamento**: divisão automática de valores
- **Conformidade regulatória**: SPA/ME nº 615/2024

---

## 1. Arquitetura e Princípios

### Modelo Integração

```mermaid
flowchart LR

  subgraph Cliente
    USR@{shape: circle, label: "Usuário"}
    CLI["ERP Cliente"]
  end

  AZI["Azion"]

  subgraph LB
    subgraph exposta
      AGW["API Gateway"]
      KEY["Keycloak"]
      POR["Portal Desenvolvedor"]
      CUS["Portal Customer"]
    end

    subgraph protegida
      API["API PIX"]
      BCK["Backend <br> (Orchestration)"]
      BFF["BFF Customer"]
    end
  end

  subgraph BACEN
    BCN["BACEN"]
  end

  CLI --> AZI --> AGW
  AGW --> API
  API --> BCK
  AGW --> KEY
  BCK --> BCN

  AZI -->  KEY
  POR --> KEY
  USR --> CLI
  USR --> CUS
  CUS --> BFF
  CUS --> KEY
  USR --> POR
  API -. postback webhook (LB Broker).-> CLI
  BFF --> BCK

```

```mermaid
    C4Component
    title Component diagram for LBPay PIX API - API Application

    Container(front, "Frontend", "javascript", "Provides all the functionality to customers via their web browser.")
    System_Ext(mbs, "Customer ERP", "Customer`s systems")
    Container(gateway, "Gateway")

    Container_Boundary(api, "API Application") {
        Component(token, "Token de acesso<br>/oauth/token", "Rest Controller", "Permite clientes obterem tokens de acesso")
        Component(health, "Health check<br>/health", "Rest Controller", "Saúde da API (kubernetes)")
        Component(status, "Status da API<br>/status", "Rest Controller", "Permite clientes consultarem status da API")
        Component(payments, "PIX enviados<br>/pix/payments", "Rest Controller")
        Component(receipts, "PIX recebidos<br>/pix/receipts", "Rest Controller")
        Component(qrcodes, "QR Codes<br>/pix/qrcodes", "Rest Controller")
        Component(charges, "COB e COBV<br>/pix/charges", "Rest Controller")
        Component(recurrences, "Recorrências<br>/pix/recurrences", "Rest Controller")
        Component(keys, "Chaves PIX<br>/pix/keys", "Rest Controller")
        Component(locations, "Locations<br>/pix/locations", "Rest Controller")
        Component(transfers, "Transferências<br>/transfers", "Rest Controller")
        Component(webhooks, "Webhooks<br>/webhooks", "Rest Controller")
        Component(splits, "Splits<br>/pix/split", "Rest Controller")
        Component(accounts, "Saldo, extrato, recibo<br>/accounts", "Rest Controller")
        Component(institutions, "Listar instituições<br>/institutions", "Rest Controller")
    }
```

Modelo base de requisições assíncronas (com processamento):

```mermaid
sequenceDiagram
  autonumber

  participant CLI as Cliente
  participant GTW as Gateway
  participant API as API PIX
  participant BCK as Backend
  participant PSR as Pulsar
  participant WHK as Webhook Worker

  par cliente realiza requisições
    CLI -> GTW: request
    GTW -> GTW: valida rate limit
    GTW -> GTW: valida headers
    GTW -> GTW: auth?

    GTW -> API: request
    API -> API: valida payload
    API ->> PSR: publica request
    API -> GTW: 202 ACCEPTED
    GTW -> CLI: 202 ACCEPTED
  end

  par requisições são processadas
    PSR -) BCK: consome requisição
    BCK -> BCK: valida business
    BCK -> BCK: processa
    BCK ->> PSR: publica response
  end

  par respostas são enviadas ao cliente
    PSR -) WHK: consome response
    WHK -> WHK: localiza url destino
    WHK -> CLI: response via webhook
  end
```

Modelo base de requisições síncronas (consultas):

```mermaid
sequenceDiagram
  autonumber

  participant CLI as Cliente
  participant GTW as Gateway
  participant API as API PIX
  participant BCK as Backend

  CLI -> GTW: request
  GTW -> GTW: valida rate limit
  GTW -> GTW: valida headers
  GTW -> GTW: auth?

  GTW -> API: request
  API -> API: valida payload
  API -> BCK: request
  BCK -> API: response
  
  API -> GTW: 200 OK
  GTW -> CLI: 200 OK
```

### 1.1 Princípios de Design REST

- **Recursos como substantivos**: URLs representam recursos, não ações
- **Métodos HTTP semânticos**: GET (leitura), POST (criação), PUT (atualização completa), PATCH (atualização parcial), DELETE (remoção)
- **Stateless**: cada requisição contém todas as informações necessárias
- **HATEOAS**: (Hypermedia as the Engine of Application State) respostas incluem links para recursos relacionados quando aplicável
- **Versionamento**: via URL (`/v1/`) para compatibilidade
- **Idempotência**: operações críticas (POST, PUT) suportam `idempotency-key`

### 1.2 Autenticação e Segurança

#### OAuth 2.0 + mTLS

```bash
POST /v1/oauth/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=pix.read pix.write
```

**Resposta:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "pix.read pix.write"
}
```

**Requisitos de Segurança:**

- OAuth 2.0 com Client Credentials Flow
- Certificado TLS mútuo (mTLS) obrigatório para operações PIX
- Certificado no formato P12 ou PEM conforme padrão BACEN
- JWT com expiração de 1 hora
- Renovação automática 5 minutos antes da expiração

**Headers obrigatórios em todas as requisições autenticadas:**

```html
Authorization: Bearer {access_token}
X-API-Version: 1.0
X-Idempotency-Key: {uuid} (opcional, recomendado para POST/PUT)
```

### 1.3 Rate Limiting

A API implementa rate limiting para garantir disponibilidade e fair usage:

**Limites padrão:**

- 100 requisições/minuto por client_id
- 1000 requisições/hora por client_id
- 10000 requisições/dia por client_id

**Headers de resposta:**

```html
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642694400
```

**Resposta quando limite excedido (429):**

```json
{
  "error": "rate_limit_exceeded",
  "error_description": "Taxa de requisições excedida. Tente novamente em 45 segundos.",
  "retry_after": 45
}
```

### 1.4 Paginação

Todas as listagens suportam paginação baseada em cursor:

**Parâmetros de query:**

- `limit`: Número de itens por página (padrão: 50, máximo: 100)
- `cursor`: Token do cursor para próxima página

**Exemplo de resposta paginada:**

```json
{
  "data": [...],
  "pagination": {
    "total": 250,
    "limit": 50,
    "has_more": true,
    "next_cursor": "eyJpZCI6MTIzNDU2fQ=="
  }
}
```

### 1.5 Códigos HTTP Padronizados

| Código | Significado | Uso |
|--------|-------------|-----|
| **200** | OK | Operação bem-sucedida (GET, PUT, PATCH, DELETE) |
| **201** | Created | Recurso criado com sucesso (POST) |
| **202** | Accepted | Requisição aceita para processamento assíncrono |
| **204** | No Content | Operação bem-sucedida sem corpo de resposta |
| **400** | Bad Request | Erro de validação nos dados enviados |
| **401** | Unauthorized | Token inválido ou ausente |
| **403** | Forbidden | Permissão insuficiente para a operação |
| **404** | Not Found | Recurso não encontrado |
| **409** | Conflict | Conflito (ex: duplicação, operação já processada) |
| **422** | Unprocessable Entity | Dados válidos mas regra de negócio impediu processamento |
| **429** | Too Many Requests | Rate limit excedido |
| **500** | Internal Server Error | Erro interno do servidor |
| **503** | Service Unavailable | Serviço temporariamente indisponível |

### 1.6 Estrutura de Erro Padronizada

```json
{
  "error": "invalid_request",
  "error_description": "O campo 'valor' deve ser maior que zero",
  "type": "https://api.lbpay.com.br/errors/invalid_request",
  "status": 400,
  "detail": "Valor informado: -10.50. O valor mínimo permitido é 0.01",
  "timestamp": "2025-01-24T10:30:45Z",
  "path": "/v1/pix/payments",
  "request_id": "req_abc123xyz",
  "validation_errors": [
    {
      "field": "valor",
      "message": "Deve ser maior que zero",
      "code": "invalid_value"
    }
  ]
}
```

---

## 2. Operações PIX - Envio e Pagamento (PIX OUT)

### 2.1 Enviar PIX

Realiza envio de PIX via chave, dados bancários ou telefone.

`POST /v1/pix/payments`

**Request Body:**

```json
{
  "valor": 100.50,
  "descricao": "Pagamento de serviço",
  "external_id": "pedido-12345",
  "destinatario": {
    "chave_pix": "email@exemplo.com",
    "tipo_chave": "email"
  },
  "pagador": {
    "nome": "João Silva",
    "cpf": "12345678900",
    "conta_id": "acc_abc123"
  }
}
```

**Alternativa - Envio por dados bancários:**

```json
{
  "valor": 100.50,
  "descricao": "Pagamento de serviço",
  "external_id": "pedido-12345",
  "destinatario": {
    "nome": "Maria Santos",
    "cpf": "98765432100",
    "banco": "001",
    "agencia": "1234",
    "conta": "567890",
    "tipo_conta": "corrente"
  },
  "pagador": {
    "cpf": "12345678900",
    "conta_id": "acc_abc123"
  }
}
```

**Response (201 Created):**

```json
{
  "id": "pix_pay_abc123xyz",
  "end_to_end_id": "E12345678202501241030ABC123XYZ",
  "external_id": "pedido-12345",
  "valor": 100.50,
  "status": "EM_PROCESSAMENTO",
  "descricao": "Pagamento de serviço",
  "destinatario": {
    "chave_pix": "email@exemplo.com",
    "tipo_chave": "email",
    "nome": "Maria Santos",
    "cpf": "987.654.321-00",
    "banco": {
      "ispb": "00000000",
      "nome": "Banco Exemplo",
      "codigo": "001"
    }
  },
  "pagador": {
    "nome": "João Silva",
    "cpf": "123.456.789-00",
    "conta_id": "acc_abc123"
  },
  "horario": {
    "solicitacao": "2025-01-24T10:30:45Z",
    "processamento": null
  },
  "criado_em": "2025-01-24T10:30:45Z",
  "atualizado_em": "2025-01-24T10:30:45Z"
}
```

**Validações:**

- Valor mínimo: R$ 0,01
- Valor máximo: conforme limites da conta
- Chave PIX válida no DICT
- Saldo suficiente
- Conta ativa e sem bloqueios
- CPF/CNPJ do pagador válido

**Fluxo de Processamento:**

1. Validação de autenticação e autorização
2. Validação de dados e limites
3. Consulta DICT (chave PIX)
4. Validação de saldo
5. Reserva de valor na conta
6. Envio para SPI (se PSP externo) ou liquidação interna
7. Processamento contábil
8. Notificação via webhook
9. Retorno do status

### 2.2 Enviar PIX - Mesma Titularidade

Envio de PIX com validação obrigatória de mesma titularidade.

`POST /v1/pix/payments/same-ownership`

**Request Body:**

```json
{
  "valor": 500.00,
  "descricao": "Transferência entre contas próprias",
  "external_id": "transfer-67890",
  "destinatario": {
    "chave_pix": "12345678900",
    "tipo_chave": "cpf"
  },
  "pagador": {
    "cpf": "12345678900",
    "conta_id": "acc_abc123"
  }
}
```

**Response (201 Created):**

```json
{
  "id": "pix_pay_def456uvw",
  "end_to_end_id": "E12345678202501241035DEF456UVW",
  "external_id": "transfer-67890",
  "valor": 500.00,
  "status": "REALIZADO",
  "mesma_titularidade": true,
  "liquidacao_interna": true,
  "descricao": "Transferência entre contas próprias",
  "destinatario": {
    "chave_pix": "123.456.789-00",
    "tipo_chave": "cpf",
    "nome": "João Silva",
    "cpf": "123.456.789-00"
  },
  "pagador": {
    "nome": "João Silva",
    "cpf": "123.456.789-00",
    "conta_id": "acc_abc123"
  },
  "horario": {
    "solicitacao": "2025-01-24T10:35:00Z",
    "liquidacao": "2025-01-24T10:35:01Z"
  },
  "criado_em": "2025-01-24T10:35:00Z",
  "atualizado_em": "2025-01-24T10:35:01Z"
}
```

**Validações adicionais:**

- CPF/CNPJ do pagador deve ser igual ao do titular da chave PIX no DICT
- Liquidação prioritária interna quando ambas contas são do LB Pay

### 2.3 Consultar PIX Enviado por ID

`GET /v1/pix/payments/{id}`

**Response (200 OK):**

```json
{
  "id": "pix_pay_abc123xyz",
  "end_to_end_id": "E12345678202501241030ABC123XYZ",
  "external_id": "pedido-12345",
  "valor": 100.50,
  "status": "REALIZADO",
  "descricao": "Pagamento de serviço",
  "destinatario": { ... },
  "pagador": { ... },
  "horario": {
    "solicitacao": "2025-01-24T10:30:45Z",
    "liquidacao": "2025-01-24T10:30:46Z"
  },
  "criado_em": "2025-01-24T10:30:45Z",
  "atualizado_em": "2025-01-24T10:30:46Z"
}
```

### 2.4 Consultar PIX Enviado por End-to-End ID

`GET /v1/pix/payments/e2e/{endToEndId}`

**Exemplo:**

`GET /v1/pix/payments/e2e/E12345678202501241030ABC123XYZ`

### 2.5 Listar PIX Enviados

`GET /v1/pix/payments?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&status=REALIZADO&limit=50&cursor=abc123`

**Parâmetros de Query:**

| Parâmetro | Obrigatório | Tipo | Descrição |
|-----------|-------------|------|-----------|
| `inicio` | Sim | DateTime (ISO 8601) | Data/hora inicial (máx 90 dias) |
| `fim` | Sim | DateTime (ISO 8601) | Data/hora final |
| `status` | Não | String | Filtro por status: EM_PROCESSAMENTO, REALIZADO, NAO_REALIZADO |
| `external_id` | Não | String | Filtro por ID externo |
| `limit` | Não | Integer | Itens por página (padrão: 50, máx: 100) |
| `cursor` | Não | String | Token de paginação |

**Response (200 OK):**

```json
{
  "data": [
    {
      "id": "pix_pay_abc123xyz",
      "end_to_end_id": "E12345678202501241030ABC123XYZ",
      "valor": 100.50,
      "status": "REALIZADO",
      "destinatario": {
        "nome": "Maria Santos",
        "chave_pix": "email@exemplo.com"
      },
      "horario": {
        "solicitacao": "2025-01-24T10:30:45Z",
        "liquidacao": "2025-01-24T10:30:46Z"
      }
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 50,
    "has_more": true,
    "next_cursor": "eyJpZCI6MTIzNDU2fQ=="
  }
}
```

### 2.6 Buscar PIX por Filtros

`GET /v1/pix/payments/search?end_to_end_id=E12345678&external_id=pedido-123`

---

## 3. Operações PIX - Recebimento (PIX IN)

### 3.1 Consultar PIX Recebido

`GET /v1/pix/receipts/{endToEndId}`

**Response (200 OK):**

```json
{
  "end_to_end_id": "E98765432202501241040XYZ789ABC",
  "txid": "abc123xyz789",
  "valor": 250.00,
  "chave_pix": "email@lbpay.com",
  "pagador": {
    "nome": "Carlos Oliveira",
    "cpf": "11122233344",
    "banco": {
      "ispb": "99999999",
      "nome": "Banco XYZ",
      "codigo": "999"
    }
  },
  "beneficiario": {
    "nome": "LB Pay Cliente",
    "cpf": "55566677788",
    "conta_id": "acc_xyz789"
  },
  "horario": "2025-01-24T10:40:30Z",
  "devolucoes": [],
  "info_adicional": "Pagamento de cobrança #12345"
}
```

### 3.2 Listar PIX Recebidos

`GET /v1/pix/receipts?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&limit=50`

### 3.3 Solicitar Devolução (Refund)

`POST /v1/pix/receipts/{endToEndId}/refunds`

**Request Body:**

```json
{
  "id_devolucao": "DEV123456",
  "valor": 50.00,
  "descricao": "Devolução parcial - produto defeituoso"
}
```

**Response (201 Created):**

```json
{
  "id": "refund_abc123",
  "id_devolucao": "DEV123456",
  "end_to_end_id": "E98765432202501241040XYZ789ABC",
  "rtrid": "D12345678202501241045DEV123XYZ",
  "valor": 50.00,
  "status": "EM_PROCESSAMENTO",
  "descricao": "Devolução parcial - produto defeituoso",
  "criado_em": "2025-01-24T10:45:00Z"
}
```

### 3.4 Consultar Devolução

`GET /v1/pix/receipts/{endToEndId}/refunds/{idDevolucao}`

---

## 4. QR Codes PIX

### 4.1 Criar QR Code Estático

`POST /v1/pix/qrcodes/static`

**Request Body:**

```json
{
  "chave_pix": "email@lbpay.com",
  "valor": 100.00,
  "descricao": "Pagamento de serviço",
  "merchant_name": "LB Pay Serviços",
  "merchant_city": "São Paulo",
  "txid": "qrcode123456789",
  "reutilizavel": false
}
```

**Response (201 Created):**

```json
{
  "id": "qr_static_abc123",
  "txid": "qrcode123456789",
  "chave_pix": "email@lbpay.com",
  "valor": 100.00,
  "descricao": "Pagamento de serviço",
  "brcode": "00020126580014br.gov.bcb.pix...",
  "qrcode_url": "https://api.lbpay.com.br/v1/pix/qrcodes/static/qr_static_abc123/image",
  "qrcode_base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUg...",
  "reutilizavel": false,
  "criado_em": "2025-01-24T11:00:00Z"
}
```

### 4.2 Criar QR Code Dinâmico

`POST /v1/pix/qrcodes/dynamic`

**Request Body:**

```json
{
  "chave_pix": "email@lbpay.com",
  "valor": 150.00,
  "descricao": "Cobrança #12345",
  "devedor": {
    "cpf": "12345678900",
    "nome": "João Silva"
  },
  "expiracao": 3600,
  "calendario": {
    "data_vencimento": "2025-01-25",
    "validade_apos_vencimento": 3
  },
  "valor_original": 150.00,
  "multa": {
    "modalidade": "PERCENTUAL",
    "valor": 2.00
  },
  "juros": {
    "modalidade": "VALOR_DIA",
    "valor": 1.50
  },
  "desconto": {
    "modalidade": "PERCENTUAL",
    "data_limite": "2025-01-24",
    "valor": 5.00
  }
}
```

**Response (201 Created):**

```json
{
  "id": "qr_dynamic_xyz789",
  "txid": "dyn_qrcode_abc123xyz",
  "location_id": "loc_abc123",
  "chave_pix": "email@lbpay.com",
  "valor": 150.00,
  "status": "ATIVA",
  "brcode": "00020126580014br.gov.bcb.pix...",
  "qrcode_url": "https://api.lbpay.com.br/v1/pix/qrcodes/dynamic/qr_dynamic_xyz789/image",
  "location_url": "https://api.lbpay.com.br/pix/cob/dyn_qrcode_abc123xyz",
  "criado_em": "2025-01-24T11:10:00Z",
  "expira_em": "2025-01-24T12:10:00Z"
}
```

### 4.3 Decodificar QR Code (BR Code)

`POST /v1/pix/qrcodes/decode`

**Request Body:**

```json
{
  "brcode": "00020126580014br.gov.bcb.pix0136email@exemplo.com520400005303986540510.005802BR5913Merchant Name6009Sao Paulo62070503***6304XXXX"
}
```

**Response (200 OK):**

```json
{
  "tipo": "ESTATICO",
  "chave_pix": "email@exemplo.com",
  "valor": 10.00,
  "merchant": {
    "name": "Merchant Name",
    "city": "Sao Paulo"
  },
  "txid": "***",
  "location_url": null,
  "calendar": null,
  "devedor": null,
  "valor_original": 10.00,
  "desconto": null,
  "juros": null,
  "multa": null
}
```

### 4.4 Pagar QR Code

`POST /v1/pix/qrcodes/pay`

**Request Body:**

```json
{
  "brcode": "00020126580014br.gov.bcb.pix...",
  "pagador": {
    "cpf": "12345678900",
    "conta_id": "acc_abc123"
  },
  "external_id": "pagamento-qr-12345"
}
```

**Response (201 Created):**

```json
{
  "id": "pix_pay_qr_abc123",
  "end_to_end_id": "E12345678202501241115ABC123XYZ",
  "external_id": "pagamento-qr-12345",
  "valor": 10.00,
  "status": "EM_PROCESSAMENTO",
  "qrcode": {
    "tipo": "ESTATICO",
    "chave_pix": "email@exemplo.com",
    "merchant_name": "Merchant Name"
  },
  "pagador": {
    "cpf": "123.456.789-00",
    "conta_id": "acc_abc123"
  },
  "horario": {
    "solicitacao": "2025-01-24T11:15:00Z"
  },
  "criado_em": "2025-01-24T11:15:00Z"
}
```

### 4.5 Obter Imagem do QR Code

`GET /v1/pix/qrcodes/static/{id}/image`

`GET /v1/pix/qrcodes/dynamic/{id}/image`

**Response:**

- Content-Type: `image/png`
- Retorna imagem PNG do QR Code

### 4.6 Atualizar QR Code Dinâmico

`PATCH /v1/pix/qrcodes/dynamic/{id}`

**Request Body:**

```json
{
  "valor": 200.00,
  "expiracao": 7200
}
```

### 4.7 Cancelar QR Code Dinâmico

`DELETE /v1/pix/qrcodes/dynamic/{id}`

---

## 5. Cobranças PIX (COB e COBV)

### 5.1 Criar Cobrança Imediata (COB)

`POST /v1/pix/charges`

`PUT /v1/pix/charges/{txid}`

**Request Body:**

```json
{
  "calendario": {
    "expiracao": 3600
  },
  "devedor": {
    "cpf": "12345678900",
    "nome": "João Silva"
  },
  "valor": {
    "original": "100.50"
  },
  "chave": "email@lbpay.com",
  "solicitacao_pagador": "Pagamento de serviço #12345",
  "info_adicionais": [
    {
      "nome": "Pedido",
      "valor": "12345"
    }
  ]
}
```

**Response (201 Created):**

```json
{
  "txid": "cob_abc123xyz789",
  "location": "qrcode.lbpay.com.br/v1/payload/cob_abc123xyz789",
  "location_id": "loc_abc123",
  "status": "ATIVA",
  "calendario": {
    "criacao": "2025-01-24T11:20:00Z",
    "expiracao": 3600
  },
  "devedor": {
    "cpf": "123.456.789-00",
    "nome": "João Silva"
  },
  "valor": {
    "original": "100.50"
  },
  "chave": "email@lbpay.com",
  "solicitacao_pagador": "Pagamento de serviço #12345",
  "brcode": "00020126580014br.gov.bcb.pix...",
  "qrcode_url": "https://api.lbpay.com.br/v1/pix/charges/cob_abc123xyz789/qrcode",
  "criado_em": "2025-01-24T11:20:00Z"
}
```

### 5.2 Criar Cobrança com Vencimento (COBV)

`POST /v1/pix/charges/due-date`

`PUT /v1/pix/charges/due-date/{txid}`

**Request Body:**

```json
{
  "calendario": {
    "data_vencimento": "2025-02-01",
    "validade_apos_vencimento": 30
  },
  "devedor": {
    "cpf": "12345678900",
    "nome": "João Silva",
    "logradouro": "Rua Exemplo, 123",
    "cidade": "São Paulo",
    "uf": "SP",
    "cep": "01000000"
  },
  "valor": {
    "original": "250.00",
    "multa": {
      "modalidade": "PERCENTUAL",
      "valor_perc": "2.00"
    },
    "juros": {
      "modalidade": "VALOR_DIA",
      "valor": "1.50"
    },
    "desconto": {
      "modalidade": "PERCENTUAL",
      "data_limite": "2025-01-30",
      "valor_perc": "5.00"
    }
  },
  "chave": "email@lbpay.com",
  "solicitacao_pagador": "Mensalidade Janeiro/2025"
}
```

**Response (201 Created):**

```json
{
  "txid": "cobv_xyz789abc123",
  "location": "qrcode.lbpay.com.br/v1/payload/cobv_xyz789abc123",
  "location_id": "loc_xyz789",
  "status": "ATIVA",
  "revisao": 0,
  "calendario": {
    "criacao": "2025-01-24T11:25:00Z",
    "data_vencimento": "2025-02-01",
    "validade_apos_vencimento": 30
  },
  "valor": {
    "original": "250.00",
    "final": "250.00"
  },
  "brcode": "00020126580014br.gov.bcb.pix...",
  "qrcode_url": "https://api.lbpay.com.br/v1/pix/charges/due-date/cobv_xyz789abc123/qrcode",
  "criado_em": "2025-01-24T11:25:00Z"
}
```

### 5.3 Consultar Cobrança

`GET /v1/pix/charges/{txid}`

`GET /v1/pix/charges/due-date/{txid}`

### 5.4 Listar Cobranças

`GET /v1/pix/charges?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&status=ATIVA`

`GET /v1/pix/charges/due-date?inicio=2025-01-01&fim=2025-01-31&status=ATIVA`

---

## 6. PIX Automático (Recorrências)

### 6.1 Criar Recorrência

`POST /v1/pix/recurrences`

**Request Body:**

```json
{
  "calendario": {
    "data_inicio": "2025-02-01",
    "recorrencia": "MENSAL",
    "quantidade": 12
  },
  "devedor": {
    "cpf": "12345678900",
    "nome": "João Silva"
  },
  "valor": {
    "original": "99.90",
    "permite_alteracao": false
  },
  "chave": "email@lbpay.com",
  "descricao": "Assinatura Premium - Anual"
}
```

**Response (201 Created):**

```json
{
  "id_rec": "rec_abc123xyz",
  "status": "ATIVA",
  "calendario": {
    "data_inicio": "2025-02-01",
    "recorrencia": "MENSAL",
    "quantidade": 12,
    "criacao": "2025-01-24T11:30:00Z"
  },
  "devedor": {
    "cpf": "123.456.789-00",
    "nome": "João Silva"
  },
  "valor": {
    "original": "99.90",
    "permite_alteracao": false
  },
  "chave": "email@lbpay.com",
  "descricao": "Assinatura Premium - Anual",
  "location": "qrcode.lbpay.com.br/v1/payload/rec_abc123xyz",
  "qrcode_url": "https://api.lbpay.com.br/v1/pix/recurrences/rec_abc123xyz/qrcode",
  "criado_em": "2025-01-24T11:30:00Z"
}
```

### 6.2 Criar Solicitação de Confirmação

`POST /v1/pix/recurrences/confirmation-requests`

### 6.3 Criar Cobrança Recorrente

`POST /v1/pix/recurrences/{idRec}/charges`

`PUT /v1/pix/recurrences/{idRec}/charges/{txid}`

### 6.4 Consultar Recorrência

`GET /v1/pix/recurrences/{idRec}`

### 6.5 Listar Recorrências

`GET /v1/pix/recurrences?inicio=2025-01-01T00:00:00Z&fim=2025-12-31T23:59:59Z&status=ATIVA`

### 6.6 Cancelar Recorrência

`PATCH /v1/pix/recurrences/{idRec}`

**Request Body:**

```json
{
  "status": "CANCELADA"
}
```

---

## 7. Chaves PIX

### 7.1 Listar Chaves PIX

`GET /v1/pix/keys`

**Response (200 OK):**

```json
{
  "chaves": [
    {
      "chave": "email@lbpay.com",
      "tipo": "email",
      "nome_titular": "LB Pay Cliente",
      "cpf_cnpj": "12345678900",
      "banco": {
        "ispb": "12345678",
        "nome": "LB Pay",
        "codigo": "999"
      },
      "conta": {
        "tipo": "corrente",
        "agencia": "0001",
        "numero": "123456"
      },
      "padrao": true,
      "criada_em": "2024-01-15T10:00:00Z"
    }
  ]
}
```

### 7.2 Criar Chave PIX Aleatória (EVP)

`POST /v1/pix/keys/random`

**Request Body:**

```json
{
  "conta_id": "acc_abc123"
}
```

**Response (201 Created):**

```json
{
  "chave": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tipo": "evp",
  "nome_titular": "LB Pay Cliente",
  "cpf_cnpj": "12345678900",
  "padrao": false,
  "criada_em": "2025-01-24T11:35:00Z"
}
```

### 7.3 Verificar Chave PIX

`POST /v1/pix/keys/check`

**Request Body:**

```json
{
  "chave": "email@exemplo.com",
  "tipo": "email"
}
```

**Response (200 OK):**

```json
{
  "chave": "email@exemplo.com",
  "tipo": "email",
  "existe": true,
  "nome_titular": "Maria Santos",
  "cpf_cnpj": "98765432100",
  "banco": {
    "ispb": "00000000",
    "nome": "Banco Exemplo",
    "codigo": "001"
  },
  "tipo_pessoa": "fisica",
  "data_criacao": "2024-05-20T14:30:00Z"
}
```

### 7.4 Definir Chave Padrão

`POST /v1/pix/keys/{chave}/set-default`

### 7.5 Deletar Chave PIX

`DELETE /v1/pix/keys/{chave}`

---

## 8. Transferências Internas

### 8.1 Transferência entre Contas (Mesma Titularidade)

`POST /v1/transfers/internal`

**Request Body:**

```json
{
  "valor": 1000.00,
  "conta_origem_id": "acc_owner_abc123",
  "conta_destino_id": "acc_transactional_xyz789",
  "tipo_transferencia": "OWNER_TO_TRANSACTIONAL",
  "descricao": "Reforço de saldo operacional",
  "external_id": "transfer-int-12345"
}
```

**Tipos de transferência:**

- `OWNER_TO_TRANSACTIONAL`: Conta Proprietária → Conta Transacional
- `TRANSACTIONAL_TO_OWNER`: Conta Transacional → Conta Proprietária
- `OWNER_TO_OPERATIONAL`: Conta Proprietária → Conta Operacional (tarifas)

**Response (201 Created):**

```json
{
  "id": "transfer_int_abc123",
  "external_id": "transfer-int-12345",
  "valor": 1000.00,
  "tipo_transferencia": "OWNER_TO_TRANSACTIONAL",
  "status": "REALIZADO",
  "conta_origem": {
    "id": "acc_owner_abc123",
    "tipo": "PROPRIETARIA",
    "titular": "João Silva",
    "cpf_cnpj": "12345678900"
  },
  "conta_destino": {
    "id": "acc_transactional_xyz789",
    "tipo": "TRANSACTIONAL",
    "titular": "João Silva",
    "cpf_cnpj": "12345678900"
  },
  "mesma_titularidade": true,
  "descricao": "Reforço de saldo operacional",
  "horario": {
    "solicitacao": "2025-01-24T11:40:00Z",
    "liquidacao": "2025-01-24T11:40:01Z"
  },
  "criado_em": "2025-01-24T11:40:00Z",
  "atualizado_em": "2025-01-24T11:40:01Z"
}
```

**Validações:**

- Mesma titularidade obrigatória (CPF/CNPJ)
- Saldo suficiente na conta origem
- Contas ativas e sem bloqueios
- Limites de transferência respeitados
- Conformidade com SPA/ME nº 615/2024

### 8.2 Consultar Transferência Interna

`GET /v1/transfers/internal/{id}`

### 8.3 Listar Transferências Internas

`GET /v1/transfers/internal?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&tipo=OWNER_TO_TRANSACTIONAL&limit=50`

**Parâmetros de Query:**

- `inicio`: Data/hora inicial (obrigatório)
- `fim`: Data/hora final (obrigatório)
- `tipo`: Tipo de transferência (opcional)
- `status`: Status da transferência (opcional)
- `conta_origem_id`: Filtro por conta origem (opcional)
- `conta_destino_id`: Filtro por conta destino (opcional)
- `limit`: Itens por página (padrão: 50)
- `cursor`: Token de paginação (opcional)

---

## 9. Webhooks

### 9.1 Configurar Webhook

`PUT /v1/webhooks/{chave}`

**Request Body:**

```json
{
  "url": "https://meusite.com/webhooks/pix",
  "eventos": [
    "pix.received",
    "pix.payment.completed",
    "pix.payment.failed",
    "charge.completed",
    "charge.expired"
  ],
  "autenticacao": {
    "tipo": "bearer",
    "token": "seu_token_secreto"
  }
}
```

**Response (200 OK):**

```json
{
  "chave": "email@lbpay.com",
  "url": "https://meusite.com/webhooks/pix",
  "eventos": [
    "pix.received",
    "pix.payment.completed",
    "pix.payment.failed",
    "charge.completed",
    "charge.expired"
  ],
  "autenticacao": {
    "tipo": "bearer"
  },
  "validado": true,
  "ativo": true,
  "criado_em": "2025-01-24T11:45:00Z",
  "atualizado_em": "2025-01-24T11:45:00Z"
}
```

**Validações do Webhook:**

1. URL deve ser HTTPS
2. TLS mútuo (mutual TLS) obrigatório
3. Teste de conectividade (handshake)
4. Timeout de 5 segundos para resposta
5. Espera-se status 200-299 no teste

**Eventos Disponíveis:**

- `pix.received`: PIX recebido
- `pix.payment.completed`: Pagamento PIX concluído
- `pix.payment.failed`: Pagamento PIX falhou
- `pix.refund.completed`: Devolução concluída
- `charge.completed`: Cobrança paga
- `charge.expired`: Cobrança expirada
- `recurrence.activated`: Recorrência ativada
- `recurrence.cancelled`: Recorrência cancelada
- `transfer.completed`: Transferência interna concluída
- `transfer.failed`: Transferência interna falhou

### 9.2 Consultar Webhook

`GET /v1/webhooks/{chave}`

**Response (200 OK):**

```json
{
  "chave": "email@lbpay.com",
  "url": "https://meusite.com/webhooks/pix",
  "eventos": ["pix.received", "pix.payment.completed"],
  "autenticacao": {
    "tipo": "bearer"
  },
  "validado": true,
  "ativo": true,
  "estatisticas": {
    "total_envios": 1234,
    "envios_sucesso": 1200,
    "envios_falha": 34,
    "ultima_falha": "2025-01-24T10:30:00Z",
    "ultimo_envio_sucesso": "2025-01-24T11:45:00Z"
  },
  "criado_em": "2025-01-20T09:00:00Z",
  "atualizado_em": "2025-01-24T11:45:00Z"
}
```

### 9.3 Listar Webhooks

`GET /v1/webhooks?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z`

**Response (200 OK):**

```json
{
  "data": [
    {
      "chave": "email@lbpay.com",
      "url": "https://meusite.com/webhooks/pix",
      "eventos": ["pix.received"],
      "ativo": true,
      "criado_em": "2025-01-20T09:00:00Z"
    }
  ],
  "pagination": {
    "total": 5,
    "limit": 50,
    "has_more": false
  }
}
```

### 9.4 Deletar Webhook

`DELETE /v1/webhooks/{chave}`

**Response (204 No Content)**

### 9.5 Reenviar Webhook

`POST /v1/webhooks/retry`

**Request Body:**

```json
{
  "evento_id": "evt_abc123xyz",
  "chave": "email@lbpay.com"
}
```

### 9.6 Estrutura de Payload do Webhook

**Exemplo - PIX Recebido:**

```json
{
  "evento": "pix.received",
  "evento_id": "evt_abc123xyz",
  "timestamp": "2025-01-24T11:50:00Z",
  "data": {
    "end_to_end_id": "E98765432202501241150XYZ789ABC",
    "txid": "cob_abc123xyz",
    "valor": 250.00,
    "chave_pix": "email@lbpay.com",
    "pagador": {
      "nome": "Carlos Oliveira",
      "cpf": "111.222.333-44"
    },
    "horario": "2025-01-24T11:50:00Z"
  },
  "signature": "sha256=a1b2c3d4e5f6..."
}
```

**Validação de Assinatura:**

```html
signature = HMAC-SHA256(payload_json, webhook_secret)
```

### 9.7 IPs dos Webhooks

`GET /v1/webhooks/ips`

**Response (200 OK):**

```json
{
  "ips": [
    "200.100.50.10",
    "200.100.50.11",
    "200.100.50.12"
  ],
  "ipv6": [
    "2001:db8:85a3::8a2e:370:7334"
  ]
}
```

---

## 10. Split de Pagamento

### 10.1 Criar Configuração de Split

`POST /v1/pix/split/configs`

**Request Body:**

```json
{
  "descricao": "Split para marketplace",
  "tipo": "DINAMICO",
  "lancamento": "IMEDIATO",
  "splits": [
    {
      "tipo": "PERCENTUAL",
      "valor": 10.00,
      "favorecido": {
        "cpf": "12345678900",
        "conta_id": "acc_marketplace_taxa"
      }
    },
    {
      "tipo": "PERCENTUAL",
      "valor": 90.00,
      "favorecido": {
        "cpf": "98765432100",
        "conta_id": "acc_vendedor"
      }
    }
  ]
}
```

**Response (201 Created):**

```json
{
  "id": "split_config_abc123",
  "descricao": "Split para marketplace",
  "tipo": "DINAMICO",
  "lancamento": "IMEDIATO",
  "status": "ATIVO",
  "splits": [
    {
      "tipo": "PERCENTUAL",
      "valor": 10.00,
      "favorecido": {
        "nome": "LB Marketplace",
        "cpf": "123.456.789-00",
        "conta_id": "acc_marketplace_taxa"
      }
    },
    {
      "tipo": "PERCENTUAL",
      "valor": 90.00,
      "favorecido": {
        "nome": "Vendedor XYZ",
        "cpf": "987.654.321-00",
        "conta_id": "acc_vendedor"
      }
    }
  ],
  "criado_em": "2025-01-24T12:00:00Z"
}
```

### 10.2 Vincular Split a Cobrança

`PUT /v1/pix/charges/{txid}/split/{splitConfigId}`

**Response (200 OK):**

```json
{
  "txid": "cob_abc123xyz",
  "split_config_id": "split_config_abc123",
  "vinculado_em": "2025-01-24T12:05:00Z"
}
```

### 10.3 Consultar Configuração de Split

`GET /v1/pix/split/configs/{id}`

### 10.4 Desvincular Split de Cobrança

`DELETE /v1/pix/charges/{txid}/split`

---

## 11. Contas e Saldo

### 11.1 Consultar Saldo

`GET /v1/accounts/{accountId}/balance`

**Response (200 OK):**

```json
{
  "conta_id": "acc_abc123",
  "saldo": {
    "disponivel": 5000.00,
    "bloqueado": 500.00,
    "total": 5500.00
  },
  "moeda": "BRL",
  "atualizado_em": "2025-01-24T12:10:00Z"
}
```

### 11.2 Consultar Extrato

`GET /v1/accounts/{accountId}/statements?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&tipo=all&limit=50`

**Parâmetros:**

- `tipo`: `all`, `pix-in`, `pix-out`, `transfer`, `fee` (opcional)
- `limit`: Itens por página (padrão: 50, máx: 100)

**Response (200 OK):**

```json
{
  "conta_id": "acc_abc123",
  "periodo": {
    "inicio": "2025-01-01T00:00:00Z",
    "fim": "2025-01-31T23:59:59Z"
  },
  "saldo_inicial": 3000.00,
  "saldo_final": 5000.00,
  "data": [
    {
      "id": "stmt_abc123",
      "tipo": "pix-in",
      "valor": 250.00,
      "descricao": "PIX recebido",
      "end_to_end_id": "E98765432202501241150XYZ789ABC",
      "data": "2025-01-24T11:50:00Z",
      "saldo_apos": 3250.00
    },
    {
      "id": "stmt_def456",
      "tipo": "pix-out",
      "valor": -100.50,
      "descricao": "Pagamento PIX",
      "end_to_end_id": "E12345678202501241030ABC123XYZ",
      "data": "2025-01-24T10:30:45Z",
      "saldo_apos": 3149.50
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 50,
    "has_more": true,
    "next_cursor": "eyJpZCI6MTIzNDU2fQ=="
  }
}
```

### 11.3 Gerar Recibo (PDF)

`GET /v1/accounts/{accountId}/receipts/{endToEndId}`

**Response:**

- Content-Type: `application/pdf`
- Retorna PDF do comprovante da transação

---

## 12. Locations (Payload)

### 12.1 Criar Location

`POST /v1/pix/locations`

**Request Body:**

```json
{
  "tipo_cob": "cob"
}
```

**Response (201 Created):**

```json
{
  "id": "loc_abc123",
  "location": "qrcode.lbpay.com.br/v1/payload/loc_abc123",
  "tipo_cob": "cob",
  "criacao": "2025-01-24T12:15:00Z"
}
```

### 12.2 Consultar Location

`GET /v1/pix/locations/{id}`

### 12.3 Listar Locations

`GET /v1/pix/locations?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z`

### 12.4 Desvincular txid de Location

`DELETE /v1/pix/locations/{id}/txid`

### 12.5 Gerar QR Code de Location

`GET /v1/pix/locations/{id}/qrcode`

---

## 13. Utilitários e Informações

### 13.1 Consultar Instituições Bancárias

`GET /v1/institutions?ispb=12345678&name=Banco&limit=50`

**Response (200 OK):**

```json
{
  "data": [
    {
      "ispb": "12345678",
      "nome": "Banco Exemplo S.A.",
      "codigo_compensacao": "001",
      "nome_reduzido": "BCO EXEMPLO"
    }
  ],
  "pagination": {
    "total": 1,
    "limit": 50,
    "has_more": false
  }
}
```

### 13.3 Health Check

`GET /v1/health`

**Response (200 OK):**

```json
{
  "status": "healthy",
  "timestamp": "2025-01-24T12:20:00Z",
  "version": "1.0.0",
  "services": {
    "database": "healthy",
    "spi": "healthy",
    "dict": "healthy"
  }
}
```

### 13.4 Status da API

`GET /v1/status`

**Response (200 OK):**

```json
{
  "api_version": "1.0.0",
  "environment": "production",
  "uptime": 864000,
  "rate_limits": {
    "minute": 100,
    "hour": 1000,
    "day": 10000
  }
}
```

---

## 14. Webhooks - Eventos Detalhados

### 14.1 Estrutura de Eventos

Todos os webhooks seguem esta estrutura base:

```json
{
  "evento": "string",
  "evento_id": "string (uuid)",
  "timestamp": "string (ISO 8601)",
  "version": "1.0",
  "data": { ... },
  "signature": "string (HMAC-SHA256)"
}
```

### 14.2 Evento: pix.received

```json
{
  "evento": "pix.received",
  "evento_id": "evt_abc123",
  "timestamp": "2025-01-24T11:50:00Z",
  "version": "1.0",
  "data": {
    "end_to_end_id": "E98765432202501241150XYZ789ABC",
    "txid": "cob_abc123xyz",
    "valor": 250.00,
    "chave_pix": "email@lbpay.com",
    "pagador": {
      "nome": "Carlos Oliveira",
      "cpf": "11122233344",
      "banco": {
        "ispb": "99999999",
        "nome": "Banco XYZ"
      }
    },
    "beneficiario": {
      "nome": "LB Pay Cliente",
      "cpf": "55566677788",
      "conta_id": "acc_xyz789"
    },
    "horario": "2025-01-24T11:50:00Z",
    "info_adicional": "Pagamento de cobrança #12345"
  },
  "signature": "sha256=..."
}
```

### 14.3 Evento: pix.payment.completed

```json
{
  "evento": "pix.payment.completed",
  "evento_id": "evt_def456",
  "timestamp": "2025-01-24T10:30:46Z",
  "version": "1.0",
  "data": {
    "id": "pix_pay_abc123xyz",
    "end_to_end_id": "E12345678202501241030ABC123XYZ",
    "external_id": "pedido-12345",
    "valor": 100.50,
    "status": "REALIZADO",
    "destinatario": {
      "nome": "Maria Santos",
      "chave_pix": "email@exemplo.com"
    },
    "horario": {
      "solicitacao": "2025-01-24T10:30:45Z",
      "liquidacao": "2025-01-24T10:30:46Z"
    }
  },
  "signature": "sha256=..."
}
```

### 14.4 Evento: transfer.completed

```json
{
  "evento": "transfer.completed",
  "evento_id": "evt_ghi789",
  "timestamp": "2025-01-24T11:40:01Z",
  "version": "1.0",
  "data": {
    "id": "transfer_int_abc123",
    "external_id": "transfer-int-12345",
    "valor": 1000.00,
    "tipo_transferencia": "OWNER_TO_TRANSACTIONAL",
    "status": "REALIZADO",
    "conta_origem_id": "acc_owner_abc123",
    "conta_destino_id": "acc_transactional_xyz789",
    "horario": {
      "solicitacao": "2025-01-24T11:40:00Z",
      "liquidacao": "2025-01-24T11:40:01Z"
    }
  },
  "signature": "sha256=..."
}
```

---

## 15. Referência Completa de Endpoints

### 15.1 Tabela Resumida

| Método | Endpoint | Descrição | Auth |
|--------|----------|-----------|------|
| **POST** | `/v1/oauth/token` | Obter token de acesso | Basic |
| **GET** | `/v1/health` | Health check | - |
| **GET** | `/v1/status` | Status da API | - |
| | | | |
| **POST** | `/v1/pix/payments` | Enviar PIX | Bearer |
| **POST** | `/v1/pix/payments/same-ownership` | Enviar PIX mesma titularidade | Bearer |
| **GET** | `/v1/pix/payments/{id}` | Consultar PIX enviado por ID | Bearer |
| **GET** | `/v1/pix/payments/e2e/{endToEndId}` | Consultar PIX por End-to-End | Bearer |
| **GET** | `/v1/pix/payments` | Listar PIX enviados | Bearer |
| **GET** | `/v1/pix/payments/search` | Buscar PIX por filtros | Bearer |
| | | | |
| **GET** | `/v1/pix/receipts/{endToEndId}` | Consultar PIX recebido | Bearer |
| **GET** | `/v1/pix/receipts` | Listar PIX recebidos | Bearer |
| **POST** | `/v1/pix/receipts/{endToEndId}/refunds` | Solicitar devolução | Bearer |
| **GET** | `/v1/pix/receipts/{endToEndId}/refunds/{id}` | Consultar devolução | Bearer |
| | | | |
| **POST** | `/v1/pix/qrcodes/static` | Criar QR Code estático | Bearer |
| **GET** | `/v1/pix/qrcodes/static/{id}` | Consultar QR Code estático | Bearer |
| **GET** | `/v1/pix/qrcodes/static/{id}/image` | Imagem QR Code estático | Bearer |
| **POST** | `/v1/pix/qrcodes/dynamic` | Criar QR Code dinâmico | Bearer |
| **GET** | `/v1/pix/qrcodes/dynamic/{id}` | Consultar QR Code dinâmico | Bearer |
| **PATCH** | `/v1/pix/qrcodes/dynamic/{id}` | Atualizar QR Code dinâmico | Bearer |
| **DELETE** | `/v1/pix/qrcodes/dynamic/{id}` | Cancelar QR Code dinâmico | Bearer |
| **GET** | `/v1/pix/qrcodes/dynamic/{id}/image` | Imagem QR Code dinâmico | Bearer |
| **POST** | `/v1/pix/qrcodes/decode` | Decodificar QR Code | Bearer |
| **POST** | `/v1/pix/qrcodes/pay` | Pagar QR Code | Bearer |
| | | | |
| **POST** | `/v1/pix/charges` | Criar cobrança imediata | Bearer |
| **PUT** | `/v1/pix/charges/{txid}` | Criar cobrança imediata c/ txid | Bearer |
| **GET** | `/v1/pix/charges/{txid}` | Consultar cobrança | Bearer |
| **GET** | `/v1/pix/charges` | Listar cobranças | Bearer |
| **POST** | `/v1/pix/charges/due-date` | Criar cobrança c/ vencimento | Bearer |
| **PUT** | `/v1/pix/charges/due-date/{txid}` | Criar COBV c/ txid | Bearer |
| **GET** | `/v1/pix/charges/due-date/{txid}` | Consultar COBV | Bearer |
| **GET** | `/v1/pix/charges/due-date` | Listar COBV | Bearer |
| | | | |
| **POST** | `/v1/pix/recurrences` | Criar recorrência | Bearer |
| **GET** | `/v1/pix/recurrences/{idRec}` | Consultar recorrência | Bearer |
| **GET** | `/v1/pix/recurrences` | Listar recorrências | Bearer |
| **PATCH** | `/v1/pix/recurrences/{idRec}` | Cancelar recorrência | Bearer |
| **POST** | `/v1/pix/recurrences/confirmation-requests` | Criar solicitação confirmação | Bearer |
| **POST** | `/v1/pix/recurrences/{idRec}/charges` | Criar cobrança recorrente | Bearer |
| **PUT** | `/v1/pix/recurrences/{idRec}/charges/{txid}` | Criar COBR c/ txid | Bearer |
| | | | |
| **GET** | `/v1/pix/keys` | Listar chaves PIX | Bearer |
| **POST** | `/v1/pix/keys/random` | Criar chave aleatória (EVP) | Bearer |
| **POST** | `/v1/pix/keys/check` | Verificar chave PIX | Bearer |
| **POST** | `/v1/pix/keys/{chave}/set-default` | Definir chave padrão | Bearer |
| **DELETE** | `/v1/pix/keys/{chave}` | Deletar chave PIX | Bearer |
| | | | |
| **POST** | `/v1/transfers/internal` | Transferência interna | Bearer |
| **GET** | `/v1/transfers/internal/{id}` | Consultar transferência | Bearer |
| **GET** | `/v1/transfers/internal` | Listar transferências | Bearer |
| | | | |
| **PUT** | `/v1/webhooks/{chave}` | Configurar webhook | Bearer |
| **GET** | `/v1/webhooks/{chave}` | Consultar webhook | Bearer |
| **GET** | `/v1/webhooks` | Listar webhooks | Bearer |
| **DELETE** | `/v1/webhooks/{chave}` | Deletar webhook | Bearer |
| **POST** | `/v1/webhooks/retry` | Reenviar webhook | Bearer |
| **GET** | `/v1/webhooks/ips` | IPs dos webhooks | Bearer |
| | | | |
| **POST** | `/v1/pix/split/configs` | Criar config split | Bearer |
| **GET** | `/v1/pix/split/configs/{id}` | Consultar config split | Bearer |
| **PUT** | `/v1/pix/charges/{txid}/split/{splitConfigId}` | Vincular split | Bearer |
| **DELETE** | `/v1/pix/charges/{txid}/split` | Desvincular split | Bearer |
| | | | |
| **GET** | `/v1/accounts/{accountId}/balance` | Consultar saldo | Bearer |
| **GET** | `/v1/accounts/{accountId}/statements` | Consultar extrato | Bearer |
| **GET** | `/v1/accounts/{accountId}/receipts/{e2eId}` | Gerar recibo PDF | Bearer |
| | | | |
| **POST** | `/v1/pix/locations` | Criar location | Bearer |
| **GET** | `/v1/pix/locations/{id}` | Consultar location | Bearer |
| **GET** | `/v1/pix/locations` | Listar locations | Bearer |
| **DELETE** | `/v1/pix/locations/{id}/txid` | Desvincular txid | Bearer |
| **GET** | `/v1/pix/locations/{id}/qrcode` | QR Code de location | Bearer |
| | | | |
| **GET** | `/v1/institutions` | Listar instituições | Bearer |

---

## 16. Ambientes e Configuração

### 16.1 Ambientes Disponíveis

| Ambiente | Base URL | Certificado | Propósito |
|----------|----------|-------------|-----------|
| **Sandbox** | `https://sandbox.lbpay.com.br` | Certificado de homologação | Testes e desenvolvimento |
| **Produção** | `https://api.lbpay.com.br` | Certificado de produção | Operações reais |

### 16.2 Obtendo Credenciais

1. Acesse o portal do desenvolvedor: `https://developer.lbpay.com.br`
2. Crie uma aplicação
3. Obtenha `client_id` e `client_secret`
4. Gere ou faça upload do certificado P12/PEM
5. Configure os scopes necessários

### 16.3 Scopes Disponíveis

| Scope | Descrição |
|-------|-----------|
| `pix.read` | Leitura de operações PIX |
| `pix.write` | Criação e modificação de operações PIX |
| `pix.charges.read` | Leitura de cobranças |
| `pix.charges.write` | Criação de cobranças |
| `pix.payments.write` | Envio de pagamentos PIX |
| `pix.keys.read` | Leitura de chaves PIX |
| `pix.keys.write` | Gestão de chaves PIX |
| `transfers.write` | Transferências internas |
| `webhooks.write` | Configuração de webhooks |
| `accounts.read` | Leitura de contas e saldo |

---

## 17. Limites e Quotas

### 17.1 Limites de Transação

| Operação | Limite |
|----------|--------|
| Valor mínimo PIX | R$ 0,01 |
| Valor máximo PIX (por transação) | Conforme limite da conta |
| QR Codes ativos por chave | 100 |
| Webhooks por chave | 1 |
| Split - número de favorecidos | 20 |
| Recorrências ativas por conta | 100 |

### 17.2 Limites Temporais

| Período | Máximo Dias |
|---------|-------------|
| Intervalo de consulta (listagens) | 90 dias |
| Validade QR Code dinâmico | 365 dias |
| Validade de cobrança | 365 dias |
| Expiração mínima de cobrança | 60 segundos |

---

## 18. SDKs e Bibliotecas

> TODO SDKs internos e externos?

---

## 19. Suporte e Documentação

> TODO: os links abaixo ainda não existem.

- [Documentação completa](https://docs.lbpay.com.br)
- [Portal do desenvolvedor](https://developer.lbpay.com.br)
- [Status da API](https://status.lbpay.com.br)
- [Suporte técnico](api-support@lbpay.com.br)
- [Comunidade](https://community.lbpay.com.br)

---

## 20. Changelog e Versionamento

A API segue versionamento semântico (SemVer):

- **Major** (v1, v2): Mudanças incompatíveis com versões anteriores
- **Minor** (v1.1, v1.2): Adição de funcionalidades compatíveis
- **Patch** (v1.0.1, v1.0.2): Correções de bugs compatíveis

---

## 21. Conformidade Regulatória e Segurança

### 21.1 Portaria SPA/ME nº 615/2024

A Portaria Normativa SPA/MF nº 615, de 16 de abril de 2024, estabelece regras específicas para transações de pagamento envolvendo apostas de quota fixa e operações financeiras. A LB PIX API garante conformidade total com esses requisitos.

#### 21.1.1 Requisitos de Mesma Titularidade

**Validação Obrigatória:**

- Todos os pagamentos de prêmios e saques devem ser realizados exclusivamente para contas de mesma titularidade
- Validação de CPF/CNPJ do pagador contra o titular da chave PIX no DICT
- Validação de CPF/CNPJ em transferências internas entre contas

**Conta Cadastrada:**

- Definida como conta de depósito ou pagamento pré-paga
- De titularidade do apostador/cliente
- Mantida em instituição financeira ou de pagamento autorizada pelo Banco Central
- Usada como origem de contribuições financeiras e destino de prêmios

**Meios de Pagamento Permitidos:**

- PIX (Pagamento Instantâneo)
- TED (Transferência Eletrônica Disponível)
- Cartão de débito ou pré-pago
- Transferência escritural (contas na mesma instituição)

#### 21.1.2 Fluxo de Validação de Titularidade

> TODO

```text
1. Receber solicitação de pagamento/transferência
2. Extrair CPF/CNPJ do pagador da conta origem
3. Consultar DICT para obter dados do titular da chave PIX destino
4. Comparar CPF/CNPJ: origem == destino
5. Se diferentes: Rejeitar com erro 422 (invalid_ownership)
6. Se iguais: Prosseguir com validações adicionais
7. Registrar validação no log de auditoria
```

#### 21.1.3 Documentação para Auditorias

**Campos Obrigatórios em Logs:**

- Data e hora da operação (ISO 8601)
- ID da transação (idEnvio, e2eId)
- CPF/CNPJ do pagador
- CPF/CNPJ do beneficiário
- Resultado da validação de titularidade (APROVADO/REJEITADO)
- Chave PIX utilizada
- Valor da transação
- Status final da operação

### 21.2 Requisitos BACEN para PIX

#### 21.2.1 Resolução BCB nº 1/2020 - Arranjo de Pagamentos PIX

**Campos Obrigatórios em Transações PIX:**

- `endToEndId`: Identificador único de ponta a ponta (32 caracteres)
- `txid`: Identificador da transação de cobrança
- Valor em formato decimal com 2 casas decimais
- Dados completos do pagador (nome, CPF/CNPJ)
- Dados completos do beneficiário (nome, CPF/CNPJ, instituição)
- Horário da solicitação e da liquidação

**Formato do End-to-End ID:**

```text
E{ISPB}{YYYYMMDD}{HHMMss}{uniqueId}
Exemplo: E12345678202501241030ABC123XYZ
```

Onde:

- E: Identificador fixo
- ISPB: 8 dígitos do ISPB da instituição
- YYYYMMDD: Data da transação
- HHMMss: Hora da transação
- uniqueId: Identificador único (11 caracteres alfanuméricos)

#### 21.2.2 Retenção de Logs (5 Anos Mínimo)

**Política de Retenção:**

- Logs de transações PIX: 5 anos mínimos
- Logs de autenticação e autorização: 5 anos
- Logs de alterações em configurações: 5 anos
- Logs de acessos de auditores: 5 anos
- Logs de incidentes de segurança: 5 anos

**Armazenamento:**

- Formato: JSON estruturado
- Compressão: Gzip para logs com mais de 30 dias
- Localização: Storage redundante em múltiplas regiões
- Backup: Diário com retenção de 7 anos
- Criptografia: AES-256 em repouso

#### 21.2.3 Rastreabilidade End-to-End

Todas as transações PIX devem permitir rastreamento completo através do endToEndId:

`GET /v1/pix/trace/{endToEndId}`

```json
{
  "endToEndId": "E12345678202501241030ABC123XYZ",
  "traceId": "trace_abc123xyz",
  "timeline": [
    {
      "timestamp": "2025-01-24T10:30:45.123Z",
      "event": "REQUEST_RECEIVED",
      "details": "Solicitação de PIX recebida via API"
    },
    {
      "timestamp": "2025-01-24T10:30:45.234Z",
      "event": "AUTHENTICATION_SUCCESS",
      "details": "Cliente autenticado com sucesso"
    },
    {
      "timestamp": "2025-01-24T10:30:45.345Z",
      "event": "DICT_QUERY",
      "details": "Consulta DICT para chave email@exemplo.com"
    },
    {
      "timestamp": "2025-01-24T10:30:45.456Z",
      "event": "OWNERSHIP_VALIDATED",
      "details": "Titularidade validada: CPF 12345678900"
    },
    {
      "timestamp": "2025-01-24T10:30:45.567Z",
      "event": "BALANCE_RESERVED",
      "details": "Valor R$ 100,50 reservado"
    },
    {
      "timestamp": "2025-01-24T10:30:45.678Z",
      "event": "SPI_SENT",
      "details": "Mensagem PACS.008 enviada ao SPI"
    },
    {
      "timestamp": "2025-01-24T10:30:46.123Z",
      "event": "SPI_RESPONSE",
      "details": "PACS.002 recebido: Liquidação confirmada"
    },
    {
      "timestamp": "2025-01-24T10:30:46.234Z",
      "event": "ACCOUNTING_PROCESSED",
      "details": "Lançamentos contábeis realizados"
    },
    {
      "timestamp": "2025-01-24T10:30:46.345Z",
      "event": "WEBHOOK_SENT",
      "details": "Notificação webhook enviada"
    },
    {
      "timestamp": "2025-01-24T10:30:46.456Z",
      "event": "COMPLETED",
      "details": "Transação concluída com sucesso"
    }
  ],
  "status": "REALIZADO",
  "valor": 100.50,
  "pagador": {
    "cpf": "123.456.789-00"
  },
  "beneficiario": {
    "cpf": "987.654.321-00"
  }
}
```

#### 21.2.4 Horários de Funcionamento do SPI

O Sistema de Pagamentos Instantâneos (SPI) opera 24x7x365, mas com janelas de manutenção programadas:

**Disponibilidade Normal:**

- 24 horas por dia
- 7 dias por semana
- 365 dias por ano

**Janelas de Manutenção:**

- Horário: 01:00 - 06:00 (horário de Brasília)
- Frequência: Primeira segunda-feira de cada mês
- Duração máxima: 4 horas
- Notificação prévia: 7 dias

**Tratamento de Indisponibilidade:**

```json
{
  "error": "spi_unavailable",
  "error_description": "Sistema PIX temporariamente indisponível",
  "type": "https://api.lbpay.com.br/errors/spi_unavailable",
  "status": 503,
  "detail": "O SPI está em manutenção programada. Tente novamente após 06:00 BRT.",
  "retry_after": 3600,
  "maintenance_window": {
    "inicio": "2025-02-03T01:00:00-03:00",
    "fim": "2025-02-03T06:00:00-03:00"
  }
}
```

### 21.3 Segurança Cibernética (Circular 3.909/2018)

A Circular BACEN nº 3.909/2018 estabelece requisitos de segurança cibernética para instituições de pagamento.

#### 21.3.1 Criptografia de Dados em Trânsito

**TLS 1.3 Obrigatório:**

- Todas as comunicações devem usar TLS 1.3
- Suporte a TLS 1.2 apenas para retrocompatibilidade (será descontinuado)
- Cipher suites permitidos:
  - TLS_AES_256_GCM_SHA384
  - TLS_AES_128_GCM_SHA256
  - TLS_CHACHA20_POLY1305_SHA256

**Configuração mTLS:**

> TODO

```text
# Certificado do cliente (obrigatório para operações PIX)
Client-Certificate: -----BEGIN CERTIFICATE-----...-----END CERTIFICATE-----

# Validação do certificado
- Emitido por CA homologada pelo BACEN
- Formato: P12 ou PEM
- Validade mínima: 1 ano
- Revogação: Lista CRL atualizada diariamente
```

**Headers de Segurança:**

```html
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

#### 21.3.2 Criptografia de Dados em Repouso

**Algoritmos Utilizados:**

- AES-256-GCM para dados sensíveis
- ChaCha20-Poly1305 para dados de alto volume
- RSA-4096 para chaves assimétricas

**Dados Criptografados:**

- Dados bancários completos (agência, conta)
- Dados pessoais (CPF, RG, endereço completo)
- Senhas e tokens de autenticação (hashing com Argon2id)
- Certificados digitais
- Logs contendo dados sensíveis

**Gestão de Chaves:**

- HSM (Hardware Security Module) para chaves mestras
- Rotação de chaves: 90 dias
- Backup de chaves: Criptografado e armazenado em cofre separado
- Acesso a chaves: Autenticação multifator obrigatória

#### 21.3.3 Gestão de Certificados Digitais

**Ciclo de Vida:**

> TODO

```text
1. Geração: HSM com chave privada não exportável
2. Requisição: CSR enviado para CA homologada
3. Emissão: Certificado válido por 1-2 anos
4. Instalação: Deploy automatizado com validação
5. Monitoramento: Alertas 60, 30 e 7 dias antes do vencimento
6. Renovação: Automática 30 dias antes do vencimento
7. Revogação: Processo emergencial com propagação em 4 horas
```

**Endpoint de Validação:**

`GET /v1/certificates/status`

```json
{
  "certificates": [
    {
      "type": "mtls_client",
      "subject": "CN=LB Pay API Client,O=LB Pay,C=BR",
      "issuer": "CN=BACEN CA,O=Banco Central,C=BR",
      "serial": "ABC123XYZ789",
      "valid_from": "2024-01-15T00:00:00Z",
      "valid_until": "2026-01-15T23:59:59Z",
      "days_until_expiry": 365,
      "status": "ACTIVE",
      "thumbprint": "sha256:a1b2c3d4e5f6..."
    },
    {
      "type": "tls_server",
      "subject": "CN=*.lbpay.com.br,O=LB Pay,C=BR",
      "issuer": "CN=DigiCert,O=DigiCert Inc,C=US",
      "serial": "DEF456UVW012",
      "valid_from": "2024-06-01T00:00:00Z",
      "valid_until": "2025-06-01T23:59:59Z",
      "days_until_expiry": 128,
      "status": "ACTIVE",
      "thumbprint": "sha256:g7h8i9j0k1l2..."
    }
  ]
}
```

#### 21.3.4 Política de Senhas e Tokens

**Requisitos de Senha (usuários administrativos):**

- Comprimento mínimo: 12 caracteres
- Complexidade: Letras maiúsculas, minúsculas, números e símbolos
- Histórico: Não reutilizar últimas 12 senhas
- Validade: 90 dias
- Tentativas: Bloqueio após 5 tentativas falhas
- Hash: Argon2id com salt único

**Tokens OAuth 2.0:**

- Access Token: JWT com validade de 1 hora
- Refresh Token: Validade de 30 dias (rotação obrigatória)
- Algoritmo: RS256 (RSA com SHA-256)
- Renovação: Automática 5 minutos antes da expiração
- Revogação: Endpoint dedicado com propagação imediata

**Exemplo de Token JWT:**

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key_2025_01"
  },
  "payload": {
    "iss": "https://api.lbpay.com.br",
    "sub": "client_abc123",
    "aud": "https://api.lbpay.com.br",
    "exp": 1706097645,
    "iat": 1706094045,
    "jti": "jwt_xyz789",
    "scope": "pix.read pix.write pix.payments.write",
    "client_id": "cli_abc123xyz",
    "organization_id": "org_def456uvw"
  }
}
```

#### 21.3.5 Monitoramento de Segurança - SIEM (Security Information and Event Management)

**Eventos Monitorados:**

- Tentativas de autenticação (sucesso/falha)
- Acessos a dados sensíveis
- Modificações em configurações críticas
- Anomalias de tráfego (rate limit, padrões suspeitos)
- Falhas de validação de certificados
- Transações de alto valor (acima de R$ 10.000)
- Múltiplas transações em curto período
- Acessos de IPs não autorizados

**Alertas em Tempo Real:**

```json
{
  "alert_id": "alert_sec_001",
  "severity": "HIGH",
  "category": "AUTHENTICATION_FAILURE",
  "timestamp": "2025-01-24T14:30:45Z",
  "description": "5 tentativas de autenticação falhas em 2 minutos",
  "source_ip": "203.0.113.42",
  "client_id": "cli_suspected",
  "action_taken": "IP_BLOCKED_TEMPORARILY",
  "block_duration": "1800s",
  "requires_review": true
}
```

**Dashboard de Segurança:**

`GET /v1/security/dashboard`

```json
{
  "period": "last_24h",
  "summary": {
    "total_requests": 1250000,
    "blocked_requests": 245,
    "authentication_failures": 127,
    "rate_limit_exceeded": 89,
    "suspicious_patterns": 18,
    "active_incidents": 2
  },
  "incidents": [
    {
      "id": "inc_001",
      "severity": "MEDIUM",
      "type": "BRUTE_FORCE_ATTEMPT",
      "status": "MITIGATED",
      "detected_at": "2025-01-24T12:15:00Z",
      "mitigated_at": "2025-01-24T12:16:30Z"
    }
  ],
  "top_blocked_ips": [
    {"ip": "203.0.113.42", "count": 45, "reason": "rate_limit"},
    {"ip": "198.51.100.23", "count": 32, "reason": "invalid_certificate"}
  ]
}
```

#### 21.3.6 Resposta a Incidentes

**Processo de Resposta:**

1. **Detecção** (T+0min): Sistema SIEM detecta anomalia
2. **Classificação** (T+5min): Equipe de segurança avalia severidade
3. **Contenção** (T+15min): Bloqueio automático/manual conforme necessário
4. **Investigação** (T+30min): Análise de logs e identificação de causa raiz
5. **Erradicação** (T+2h): Remoção da ameaça e correção de vulnerabilidades
6. **Recuperação** (T+4h): Restauração de serviços afetados
7. **Pós-Incidente** (T+24h): Relatório detalhado e ações preventivas

**Endpoint de Notificação de Incidentes:**

`POST /v1/security/incidents`

```json
{
  "type": "SECURITY_BREACH_ATTEMPT",
  "severity": "CRITICAL",
  "description": "Tentativa de acesso não autorizado detectada",
  "affected_systems": ["authentication", "pix_payments"],
  "detection_timestamp": "2025-01-24T15:45:00Z",
  "source": {
    "ip": "203.0.113.42",
    "user_agent": "curl/7.68.0",
    "client_id": "unknown"
  },
  "evidence": {
    "logs": ["log_001", "log_002"],
    "traces": ["trace_xyz"]
  }
}
```

### 21.4 LGPD (Lei Geral de Proteção de Dados)

#### 21.4.1 Dados Pessoais Coletados e Finalidade

**Dados Coletados:**

| Categoria | Dados | Finalidade | Base Legal |
|-----------|-------|------------|------------|
| **Identificação** | Nome completo, CPF/CNPJ, RG, data de nascimento | Identificação do usuário, conformidade regulatória | Obrigação legal (Art. 7º, II) |
| **Contato** | Email, telefone, endereço | Comunicação, notificações, envio de documentos | Execução de contrato (Art. 7º, V) |
| **Financeiros** | Dados bancários, transações PIX, saldo | Prestação de serviços financeiros | Execução de contrato (Art. 7º, V) |
| **Navegação** | IP, user agent, timestamps, geolocalização | Segurança, prevenção a fraudes | Legítimo interesse (Art. 7º, IX) |
| **Autenticação** | Credenciais, tokens, certificados | Autenticação e autorização | Execução de contrato (Art. 7º, V) |

#### 21.4.2 Direitos do Titular

**Direitos Garantidos (Art. 18, LGPD):**

1. Confirmação da existência de tratamento
2. Acesso aos dados
3. Correção de dados incompletos, inexatos ou desatualizados
4. Anonimização, bloqueio ou eliminação
5. Portabilidade dos dados
6. Eliminação dos dados tratados com consentimento
7. Informação sobre compartilhamento
8. Informação sobre possibilidade de não fornecer consentimento
9. Revogação do consentimento

#### 21.4.3 Endpoints para Exercício de Direitos LGPD

`GET /v1/users/{userId}/data-export`

Exporta todos os dados pessoais do usuário em formato legível (JSON ou PDF).

**Request:**

```html
GET /v1/users/usr_abc123/data-export?format=json
Authorization: Bearer {access_token}
```

**Response (200 OK):**

```json
{
  "export_id": "export_xyz789",
  "user_id": "usr_abc123",
  "generated_at": "2025-01-24T16:00:00Z",
  "format": "json",
  "data": {
    "identificacao": {
      "nome": "João Silva",
      "cpf": "123.456.789-00",
      "data_nascimento": "1985-03-15",
      "rg": "12.345.678-9"
    },
    "contato": {
      "email": "joao.silva@exemplo.com",
      "telefone": "+55 11 98765-4321",
      "endereco": {
        "logradouro": "Rua Exemplo, 123",
        "cidade": "São Paulo",
        "uf": "SP",
        "cep": "01000-000"
      }
    },
    "dados_financeiros": {
      "contas": [
        {
          "conta_id": "acc_abc123",
          "tipo": "TRANSACTIONAL",
          "saldo": 5000.00,
          "criada_em": "2024-01-15T10:00:00Z"
        }
      ],
      "chaves_pix": [
        "joao.silva@exemplo.com",
        "+5511987654321"
      ],
      "transacoes_ultimos_90_dias": 150,
      "volume_transacionado": 45000.00
    },
    "consentimentos": [
      {
        "tipo": "termos_uso",
        "data_aceite": "2024-01-15T10:00:00Z",
        "versao": "2.0"
      },
      {
        "tipo": "politica_privacidade",
        "data_aceite": "2024-01-15T10:00:00Z",
        "versao": "1.5"
      }
    ],
    "historico_acessos": [
      {
        "data": "2025-01-24T15:30:00Z",
        "ip": "203.0.113.1",
        "acao": "LOGIN"
      }
    ]
  },
  "download_url": "https://api.lbpay.com.br/v1/exports/export_xyz789/download",
  "expires_at": "2025-01-31T16:00:00Z"
}
```

`DELETE /v1/users/{userId}/data-deletion`

Solicita a exclusão de dados pessoais (direito ao esquecimento).

**Request:**

```html
DELETE /v1/users/usr_abc123/data-deletion
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "confirmation": "CONFIRMO_EXCLUSAO",
  "reason": "Não utilizo mais o serviço",
  "keep_minimal_data_for_legal_compliance": true
}
```

**Response (202 Accepted):**

```json
{
  "deletion_request_id": "del_req_xyz789",
  "user_id": "usr_abc123",
  "status": "PENDING_VALIDATION",
  "requested_at": "2025-01-24T16:15:00Z",
  "estimated_completion": "2025-02-08T16:15:00Z",
  "validation_steps": [
    {
      "step": "verify_no_pending_transactions",
      "status": "IN_PROGRESS"
    },
    {
      "step": "verify_no_legal_holds",
      "status": "PENDING"
    },
    {
      "step": "anonymize_historical_data",
      "status": "PENDING"
    }
  ],
  "important_note": "Dados necessários para conformidade legal (logs de transações, auditoria) serão mantidos de forma anonimizada por 5 anos conforme exigência do BACEN.",
  "callback_url": "https://api.lbpay.com.br/v1/users/usr_abc123/data-deletion/status/del_req_xyz789"
}
```

`GET /v1/users/{userId}/data-correction`

Solicita correção de dados pessoais.

`PATCH /v1/users/{userId}`

Atualiza dados pessoais do usuário.

**Request:**

```html
PATCH /v1/users/usr_abc123
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "email": "joao.silva.novo@exemplo.com",
  "telefone": "+55 11 91234-5678",
  "endereco": {
    "logradouro": "Rua Nova, 456",
    "cidade": "São Paulo",
    "uf": "SP",
    "cep": "01001-000"
  }
}
```

**Response (200 OK):**

```json
{
  "user_id": "usr_abc123",
  "updated_at": "2025-01-24T16:30:00Z",
  "changes": [
    {
      "field": "email",
      "old_value": "joao.silva@exemplo.com",
      "new_value": "joao.silva.novo@exemplo.com",
      "requires_verification": true
    },
    {
      "field": "telefone",
      "old_value": "+55 11 98765-4321",
      "new_value": "+55 11 91234-5678",
      "requires_verification": true
    },
    {
      "field": "endereco",
      "updated": true,
      "requires_verification": false
    }
  ],
  "verification_emails_sent": true
}
```

#### 21.4.4 Registro de Operações com Dados Pessoais

Todas as operações envolvendo dados pessoais são registradas para auditoria:

`GET /v1/users/{userId}/data-access-log`

```json
{
  "user_id": "usr_abc123",
  "period": {
    "inicio": "2025-01-01T00:00:00Z",
    "fim": "2025-01-31T23:59:59Z"
  },
  "operations": [
    {
      "timestamp": "2025-01-24T10:30:00Z",
      "operation": "DATA_ACCESS",
      "operator": "system",
      "purpose": "Processamento de transação PIX",
      "data_categories": ["identificacao", "dados_financeiros"],
      "legal_basis": "execucao_contrato"
    },
    {
      "timestamp": "2025-01-24T16:00:00Z",
      "operation": "DATA_EXPORT",
      "operator": "usr_abc123",
      "purpose": "Exercício de direito LGPD - Acesso aos dados",
      "data_categories": ["all"],
      "legal_basis": "exercicio_direitos"
    },
    {
      "timestamp": "2025-01-24T16:30:00Z",
      "operation": "DATA_UPDATE",
      "operator": "usr_abc123",
      "purpose": "Correção de dados cadastrais",
      "data_categories": ["contato"],
      "legal_basis": "exercicio_direitos",
      "changes": ["email", "telefone", "endereco"]
    }
  ],
  "total_operations": 3
}
```

#### 21.4.5 DPO e Canal de Contato

**Informações do Encarregado de Dados (DPO):**

```json
{
  "dpo": {
    "nome": "Maria Santos Silva",
    "email": "dpo@lbpay.com.br",
    "telefone": "+55 11 4000-1234",
    "endereco": "Av. Paulista, 1000 - São Paulo/SP - CEP 01310-100"
  },
  "canais_atendimento": [
    {
      "tipo": "email",
      "contato": "privacidade@lbpay.com.br",
      "prazo_resposta": "5 dias úteis"
    },
    {
      "tipo": "formulario_web",
      "url": "https://www.lbpay.com.br/lgpd/solicitacao",
      "prazo_resposta": "5 dias úteis"
    },
    {
      "tipo": "telefone",
      "contato": "0800-123-4567",
      "horario": "Segunda a sexta, 9h às 18h",
      "prazo_resposta": "Imediato para esclarecimentos"
    }
  ],
  "politica_privacidade_url": "https://www.lbpay.com.br/privacidade",
  "ultima_atualizacao": "2024-12-01"
}
```

**Endpoint de Informações LGPD:**

`GET /v1/lgpd/info`

```json
{
  "controlador": {
    "razao_social": "LB Pay Instituição de Pagamento S.A.",
    "cnpj": "12.345.678/0001-90",
    "endereco": "Av. Paulista, 1000 - São Paulo/SP",
    "contato": "contato@lbpay.com.br"
  },
  "dpo": {
    "nome": "Maria Santos Silva",
    "email": "dpo@lbpay.com.br"
  },
  "finalidades_tratamento": [
    "Prestação de serviços de pagamento PIX",
    "Prevenção a fraudes e lavagem de dinheiro",
    "Cumprimento de obrigações legais e regulatórias",
    "Melhoria de serviços e experiência do usuário"
  ],
  "bases_legais": [
    "Execução de contrato (Art. 7º, V)",
    "Cumprimento de obrigação legal (Art. 7º, II)",
    "Legítimo interesse (Art. 7º, IX)"
  ],
  "compartilhamento": [
    {
      "destinatario": "Banco Central do Brasil",
      "finalidade": "Cumprimento de obrigações regulatórias",
      "dados_compartilhados": ["Dados de transações PIX"]
    },
    {
      "destinatario": "Instituições participantes do SPI",
      "finalidade": "Liquidação de transações PIX",
      "dados_compartilhados": ["Dados mínimos para processamento"]
    }
  ],
  "retencao": "Dados armazenados pelo prazo necessário para cumprimento de obrigações legais (mínimo 5 anos para dados transacionais)",
  "direitos_titular": [
    "Acesso aos dados",
    "Correção de dados",
    "Portabilidade",
    "Exclusão (com ressalvas legais)",
    "Revogação de consentimento",
    "Informação sobre compartilhamento"
  ]
}
```

### 21.5 Auditoria e Logs

#### 21.5.1 Estrutura de Logs Obrigatórios

Todos os logs seguem formato JSON estruturado para facilitar análise e auditoria:

```json
{
  "timestamp": "2025-01-24T10:30:45.123Z",
  "log_id": "log_abc123xyz789",
  "level": "INFO",
  "service": "pix-payments",
  "environment": "production",
  "version": "1.0.0",
  "trace_id": "trace_def456uvw012",
  "request_id": "req_ghi789rst345",
  "correlation_id": "corr_jkl012mno678",
  "event_type": "PIX_PAYMENT_SENT",
  "user": {
    "id": "usr_abc123",
    "ip": "203.0.113.1",
    "user_agent": "LBPay-SDK/1.0.0"
  },
  "transaction": {
    "id": "pix_pay_abc123xyz",
    "end_to_end_id": "E12345678202501241030ABC123XYZ",
    "external_id": "pedido-12345",
    "type": "PIX_PAYMENT",
    "valor": 100.50,
    "status": "REALIZADO"
  },
  "parties": {
    "pagador": {
      "cpf_hash": "sha256:a1b2c3...",
      "conta_id": "acc_abc123"
    },
    "beneficiario": {
      "cpf_hash": "sha256:d4e5f6...",
      "chave_pix": "email@exemplo.com",
      "banco_ispb": "00000000"
    }
  },
  "compliance": {
    "ownership_validated": true,
    "aml_check": "PASSED",
    "fraud_score": 15
  },
  "performance": {
    "duration_ms": 1234,
    "dict_lookup_ms": 145,
    "spi_roundtrip_ms": 890,
    "accounting_ms": 78
  },
  "metadata": {
    "server_id": "srv-prod-01",
    "region": "sa-east-1"
  }
}
```

#### 21.5.2 Campos Obrigatórios por Tipo de Operação

**PIX Payment (Envio):**

- `timestamp` (ISO 8601 com timezone)
- `end_to_end_id` (32 caracteres)
- `external_id` (opcional, mas recomendado)
- `valor` (decimal, 2 casas)
- `pagador.cpf_hash` (SHA-256)
- `beneficiario.cpf_hash` (SHA-256)
- `ownership_validated` (boolean)
- `status` (EM_PROCESSAMENTO, REALIZADO, NAO_REALIZADO)
- `chave_pix` ou dados bancários completos

**PIX Receipt (Recebimento):**

- `timestamp`
- `end_to_end_id`
- `txid` (se cobrança)
- `valor`
- `pagador.cpf_hash`
- `beneficiario.cpf_hash`
- `chave_pix`

**Internal Transfer:**

- `timestamp`
- `transfer_id`
- `tipo_transferencia` (OWNER_TO_TRANSACTIONAL, etc.)
- `valor`
- `conta_origem_id`
- `conta_destino_id`
- `ownership_validated` (deve ser sempre true)
- `status`

**Authentication/Authorization:**

- `timestamp`
- `user_id` ou `client_id`
- `ip_address`
- `action` (LOGIN, TOKEN_ISSUED, TOKEN_RENEWED, TOKEN_REVOKED)
- `success` (boolean)
- `failure_reason` (se applicable)
- `scopes_granted` (se applicable)

#### 21.5.3 Centralização e Retenção

**Arquitetura de Logs:**

> TODO

```text
Application Servers
    ↓ (log streaming)
Log Aggregator (Fluentd/Logstash)
    ↓ (parsing & enrichment)
SIEM Platform (Splunk/Elastic)
    ↓ (storage)
Hot Storage (Last 90 days) - Elasticsearch
    ↓ (archival)
Warm Storage (91-365 days) - S3 Standard
    ↓ (long-term)
Cold Storage (1-5 years) - S3 Glacier
    ↓ (compliance)
Archive (5+ years) - S3 Glacier Deep Archive
```

**Políticas de Retenção:**

| Tipo de Log | Hot Storage | Warm Storage | Cold Storage | Archive | Total |
|-------------|-------------|--------------|--------------|---------|-------|
| Transações PIX | 90 dias | 275 dias | 4 anos | 5+ anos | 10 anos |
| Autenticação | 90 dias | 275 dias | 4 anos | 5+ anos | 10 anos |
| Auditoria | 90 dias | 275 dias | 4 anos | 5+ anos | 10 anos |
| Performance | 30 dias | 60 dias | 275 dias | - | 1 ano |
| Errors | 90 dias | 275 dias | 4 anos | - | 5 anos |

#### 21.5.4 Acesso de Auditores

**Endpoint para Auditores BACEN:**

`GET /v1/compliance/audit-logs`

```html
GET /v1/compliance/audit-logs?inicio=2025-01-01T00:00:00Z&fim=2025-01-31T23:59:59Z&type=PIX_PAYMENT&limit=1000
Authorization: Bearer {auditor_token}
X-Audit-Request-ID: audit_req_xyz789
X-Auditor-ID: bacen_auditor_123
```

**Response (200 OK):**

```json
{
  "audit_request_id": "audit_req_xyz789",
  "period": {
    "inicio": "2025-01-01T00:00:00Z",
    "fim": "2025-01-31T23:59:59Z"
  },
  "filters": {
    "type": "PIX_PAYMENT"
  },
  "total_records": 45678,
  "returned_records": 1000,
  "data": [
    {
      "timestamp": "2025-01-24T10:30:45.123Z",
      "end_to_end_id": "E12345678202501241030ABC123XYZ",
      "valor": 100.50,
      "status": "REALIZADO",
      "pagador_cpf_hash": "sha256:a1b2c3...",
      "beneficiario_cpf_hash": "sha256:d4e5f6...",
      "ownership_validated": true
    }
  ],
  "pagination": {
    "limit": 1000,
    "has_more": true,
    "next_cursor": "eyJpZCI6MTIzNDU2fQ=="
  },
  "export_url": "https://api.lbpay.com.br/v1/compliance/audit-logs/export/audit_req_xyz789"
}
```

**Controles de Acesso:**

- Autenticação multifator obrigatória
- Certificado digital de auditor
- Whitelist de IPs do BACEN
- Rate limit: 100 requisições/hora
- Todas as consultas são registradas
- Dados sensíveis são fornecidos em hash (reversível apenas com ordem judicial)

#### 21.5.5 Exemplo Completo de Log Entry

```json
{
  "timestamp": "2025-01-24T10:30:45.123456Z",
  "log_id": "log_20250124_103045_abc123xyz789",
  "level": "INFO",
  "service": "pix-payments-api",
  "environment": "production",
  "version": "1.0.0",
  "region": "sa-east-1",
  "availability_zone": "sa-east-1a",
  "server_id": "srv-prod-pix-01",
  "container_id": "container_abc123",

  "trace": {
    "trace_id": "trace_def456uvw012",
    "span_id": "span_ghi789rst345",
    "parent_span_id": "span_jkl012mno678",
    "trace_flags": "01"
  },

  "request": {
    "request_id": "req_pqr456stu901",
    "correlation_id": "corr_vwx234yz012",
    "method": "POST",
    "path": "/v1/pix/payments",
    "query_params": {},
    "headers": {
      "user_agent": "LBPay-SDK/1.0.0 (Java/17.0.1)",
      "x_forwarded_for": "203.0.113.1",
      "content_type": "application/json"
    }
  },

  "event": {
    "type": "PIX_PAYMENT_COMPLETED",
    "category": "transaction",
    "action": "create",
    "outcome": "success"
  },

  "user": {
    "id": "usr_abc123def456",
    "client_id": "cli_xyz789uvw012",
    "organization_id": "org_mno345pqr678",
    "ip_address": "203.0.113.1",
    "country": "BR",
    "city": "São Paulo",
    "authenticated": true,
    "authentication_method": "oauth2_client_credentials"
  },

  "transaction": {
    "id": "pix_pay_abc123xyz789",
    "end_to_end_id": "E12345678202501241030ABC123XYZ",
    "external_id": "pedido-12345",
    "type": "PIX_PAYMENT",
    "subtype": "CHAVE_PIX",
    "valor": 100.50,
    "moeda": "BRL",
    "status": "REALIZADO",
    "status_anterior": "EM_PROCESSAMENTO",
    "horario_solicitacao": "2025-01-24T10:30:45.000Z",
    "horario_liquidacao": "2025-01-24T10:30:46.123Z"
  },

  "parties": {
    "pagador": {
      "id": "usr_abc123def456",
      "nome_hash": "sha256:aabbcc...",
      "cpf_hash": "sha256:ddeeff...",
      "conta_id": "acc_ghi789jkl012"
    },
    "beneficiario": {
      "nome_hash": "sha256:112233...",
      "cpf_hash": "sha256:445566...",
      "chave_pix": "email@exemplo.com",
      "chave_tipo": "email",
      "banco": {
        "ispb": "00000000",
        "nome": "Banco Exemplo",
        "codigo": "001"
      }
    }
  },

  "compliance": {
    "ownership_validated": false,
    "ownership_validation_required": false,
    "aml_check_status": "PASSED",
    "fraud_score": 15,
    "fraud_threshold": 75,
    "high_value_transaction": false,
    "sanction_list_check": "CLEAR",
    "pep_check": "NOT_PEP"
  },

  "performance": {
    "total_duration_ms": 1234,
    "breakdown": {
      "authentication_ms": 45,
      "authorization_ms": 23,
      "validation_ms": 67,
      "dict_lookup_ms": 145,
      "balance_check_ms": 34,
      "fraud_check_ms": 89,
      "spi_roundtrip_ms": 723,
      "accounting_ms": 78,
      "notification_ms": 30
    }
  },

  "spi": {
    "message_type": "PACS.008",
    "message_id": "msg_spi_abc123",
    "response_type": "PACS.002",
    "response_status": "ACCP",
    "sent_at": "2025-01-24T10:30:45.500Z",
    "received_at": "2025-01-24T10:30:46.123Z"
  },

  "financial": {
    "saldo_anterior": 5000.00,
    "saldo_posterior": 4899.50,
    "tarifa": 0.00,
    "valor_liquido": 100.50
  },

  "security": {
    "tls_version": "1.3",
    "cipher_suite": "TLS_AES_256_GCM_SHA384",
    "certificate_validated": true,
    "mtls_enabled": true
  },

  "metadata": {
    "webhook_sent": true,
    "webhook_url_hash": "sha256:778899...",
    "webhook_response_status": 200,
    "idempotency_key": "idem_aabbccddee",
    "retry_count": 0
  },

  "tags": [
    "pix",
    "payment",
    "outbound",
    "production",
    "high_priority"
  ]
}
```

### 21.6 Compliance Endpoints

Endpoints específicos para atender requisitos de conformidade e auditoria.

#### 21.6.1 GET /v1/compliance/audit-logs

Consulta logs de auditoria com filtros avançados (documentado em 21.5.4).

#### 21.6.2 GET /v1/compliance/transactions/{id}/audit-trail

Retorna trilha de auditoria completa de uma transação específica.

**Request:**

```html
GET /v1/compliance/transactions/pix_pay_abc123xyz/audit-trail
Authorization: Bearer {auditor_token}
```

**Response (200 OK):**

```json
{
  "transaction_id": "pix_pay_abc123xyz",
  "end_to_end_id": "E12345678202501241030ABC123XYZ",
  "audit_trail": [
    {
      "timestamp": "2025-01-24T10:30:45.000Z",
      "event": "TRANSACTION_CREATED",
      "actor": "usr_abc123",
      "actor_type": "USER",
      "ip_address": "203.0.113.1",
      "details": {
        "valor": 100.50,
        "external_id": "pedido-12345"
      }
    },
    {
      "timestamp": "2025-01-24T10:30:45.123Z",
      "event": "AUTHENTICATION_VERIFIED",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "method": "oauth2",
        "scopes": ["pix.payments.write"]
      }
    },
    {
      "timestamp": "2025-01-24T10:30:45.234Z",
      "event": "DICT_QUERY_SUCCESS",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "chave_pix": "email@exemplo.com",
        "titular_encontrado": true
      }
    },
    {
      "timestamp": "2025-01-24T10:30:45.345Z",
      "event": "BALANCE_RESERVED",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "conta_id": "acc_ghi789",
        "valor_reservado": 100.50,
        "saldo_anterior": 5000.00
      }
    },
    {
      "timestamp": "2025-01-24T10:30:45.500Z",
      "event": "SPI_MESSAGE_SENT",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "message_type": "PACS.008",
        "message_id": "msg_spi_abc123"
      }
    },
    {
      "timestamp": "2025-01-24T10:30:46.123Z",
      "event": "SPI_RESPONSE_RECEIVED",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "response_type": "PACS.002",
        "status": "ACCP",
        "liquidacao_confirmada": true
      }
    },
    {
      "timestamp": "2025-01-24T10:30:46.234Z",
      "event": "ACCOUNTING_PROCESSED",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "lancamentos": 2,
        "saldo_final": 4899.50
      }
    },
    {
      "timestamp": "2025-01-24T10:30:46.345Z",
      "event": "TRANSACTION_COMPLETED",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "status_final": "REALIZADO",
        "duracao_total_ms": 1345
      }
    },
    {
      "timestamp": "2025-01-24T10:30:46.456Z",
      "event": "WEBHOOK_NOTIFICATION_SENT",
      "actor": "system",
      "actor_type": "SYSTEM",
      "details": {
        "webhook_url_hash": "sha256:778899...",
        "response_status": 200
      }
    }
  ],
  "total_events": 9,
  "compliance_checks": {
    "ownership_validated": false,
    "aml_passed": true,
    "fraud_score": 15,
    "sanction_check": "CLEAR"
  }
}
```

#### 21.6.3 GET /v1/compliance/regulatory-reports

Gera relatórios regulatórios para envio ao BACEN.

**Request:**

```html
GET /v1/compliance/regulatory-reports?type=PIX_TRANSACTIONS&period=2025-01&format=csv
Authorization: Bearer {compliance_officer_token}
```

**Response (200 OK):**

```json
{
  "report_id": "report_bacen_202501_pix",
  "type": "PIX_TRANSACTIONS",
  "period": "2025-01",
  "format": "csv",
  "generated_at": "2025-02-01T08:00:00Z",
  "status": "COMPLETED",
  "summary": {
    "total_transactions": 125000,
    "total_volume": 45000000.00,
    "transactions_sent": 67000,
    "transactions_received": 58000,
    "average_transaction_value": 360.00
  },
  "download_url": "https://api.lbpay.com.br/v1/compliance/reports/report_bacen_202501_pix/download",
  "expires_at": "2025-02-08T08:00:00Z",
  "sha256_hash": "aabbccddeeff...",
  "file_size_bytes": 1234567
}
```

**Tipos de Relatórios Disponíveis:**

- `PIX_TRANSACTIONS`: Transações PIX mensais
- `AML_ALERTS`: Alertas de prevenção à lavagem de dinheiro
- `FRAUD_INCIDENTS`: Incidentes de fraude
- `SYSTEM_AVAILABILITY`: Disponibilidade do sistema
- `API_USAGE`: Uso da API por cliente
- `SECURITY_INCIDENTS`: Incidentes de segurança

#### 21.6.4 POST /v1/compliance/suspicious-activity-report

Permite reportar atividades suspeitas ao compliance.

**Request:**

```html
POST /v1/compliance/suspicious-activity-report
Authorization: Bearer {compliance_officer_token}
Content-Type: application/json

{
  "type": "UNUSUAL_TRANSACTION_PATTERN",
  "severity": "MEDIUM",
  "user_id": "usr_suspicious_123",
  "transactions": [
    "pix_pay_abc123",
    "pix_pay_def456",
    "pix_pay_ghi789"
  ],
  "description": "Múltiplas transações de alto valor em curto período de tempo",
  "indicators": [
    "10 transações em 5 minutos",
    "Valor total: R$ 50.000,00",
    "Destinatários diversos sem relação aparente"
  ],
  "reported_by": "compliance_officer_xyz",
  "additional_information": "Cliente não possui histórico de transações dessa magnitude"
}
```

**Response (201 Created):**

```json
{
  "report_id": "sar_2025_001234",
  "status": "UNDER_INVESTIGATION",
  "created_at": "2025-01-24T17:00:00Z",
  "assigned_to": "compliance_team",
  "next_steps": [
    "Análise detalhada do perfil do cliente",
    "Verificação de origem dos fundos",
    "Possível reporte ao COAF se confirmado"
  ],
  "investigation_deadline": "2025-01-31T17:00:00Z"
}
```

---

## 22. Performance, Escalabilidade e Observabilidade

### 22.1 Estratégias de Caching

O caching é fundamental para reduzir latência e carga nos sistemas backend.

**Cache de Consultas DICT:** TTL de 5 minutos, Redis Cluster, estratégia cache-aside
**Cache de Instituições Bancárias:** TTL de 24 horas, atualização diária
**Cache de Configurações:** TTL infinito com invalidação explícita

**Headers de Cache:**

```html
X-Cache-Status: HIT | MISS | VALIDATED
Cache-Control: private, max-age=300
ETag: "v1-resource-timestamp"
Age: 55
```

### 22.2 Rate Limiting Avançado

**Limits por Tier de Cliente:**

- Free: 100 req/min (burst: 120)
- Professional: 500 req/min (burst: 600)
- Business: 2000 req/min (burst: 2500)
- Enterprise: 10000 req/min (burst: 12000)

**Headers Expandidos:**

```html
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 487
X-RateLimit-Reset: 1706097600
X-RateLimit-Policy: 500-per-minute
X-RateLimit-Tier: professional
X-RateLimit-Burst-Limit: 600
```

**Rate Limiting por IP:** 60 req/min global, blacklist automática após 3 violações

### 22.3 Circuit Breaker e Resiliência

**Estados:** CLOSED (normal) → OPEN (bloqueado após falhas) → HALF_OPEN (testando) → CLOSED

**Thresholds por Dependência:**

- SPI: 5 falhas em 10s, timeout 30s
- DICT: 10 falhas em 30s, timeout 60s
- Database: 3 falhas em 5s, timeout 10s

**Timeouts por Operação:**

- DICT Lookup: 7s total
- SPI Message: 15s total
- Database Query: 6s total
- Webhook Call: 10s total

**Retry Policy:** 3 tentativas máximas, exponential backoff (1s, 2s, 4s) com jitter 20%

**Fallback Strategies:**

- DICT indisponível: Cache local (até 24h desatualizado)
- SPI indisponível: Fila assíncrona com retry
- Database down: Read-only mode com replicas

### 22.4 Observabilidade

**Métricas Prometheus:**

> TODO

```text
GET /v1/metrics

http_requests_total{method,endpoint,status}
http_request_duration_seconds{endpoint}
pix_transactions_total{type,status}
pix_transaction_value_brl{type}
circuit_breaker_state{name}
redis_cache_hit_rate
```

**Distributed Tracing (OpenTelemetry):**

```text
traceparent: 00-{trace-id}-{span-id}-{flags}
X-Request-ID: req_abc123xyz789
X-Trace-ID: trace_0af7651916cd
X-Correlation-ID: corr_client_123
```

**Logs Estruturados:** JSON format com trace_id, request_id, correlation_id (ver seção 21.5)

### 22.5 SLOs, SLIs e SLAs

**Service Level Objectives (SLOs):**

- Disponibilidade: 99.95%
- Latência p95: < 300ms
- Taxa de Erro: < 0.1%
- Taxa de Sucesso PIX: > 99.5%

**Service Level Agreements (SLAs) por Tier:**

- Professional: 99% disponibilidade, compensação 10%
- Business: 99.5% disponibilidade, compensação 25%
- Enterprise: 99.9% disponibilidade, compensação 50%
- Custom: 99.95% disponibilidade, compensação 100%

**Error Budget:** Com 99.95% SLO = 21.6 min downtime/mês permitido

**Janelas de Manutenção:** Primeira segunda-feira do mês, 02:00-04:00 BRT, notificação 7 dias antes

### 22.6 Arquitetura de Alta Disponibilidade

**Multi-Region:**

- Primary: sa-east-1 (São Paulo)
- Secondary: sa-east-2 (São Paulo - zona diferente)
- DR: us-east-1 (Virgínia)

**Load Balancing:** Layer 7 ALB, algoritmo Least Outstanding Requests, health checks a cada 10s

**Database Replication:**

- 1 Primary (writes + reads)
- 3 Replicas (reads only)
- Replication lag < 1s
- Failover automático em 30s

**Disaster Recovery:**

- RPO: < 5 minutos
- RTO: < 30 minutos
- Backups: Full diário, incremental horário, PITR 35 dias

**Zero-Downtime Deployment:** Blue-Green strategy com canary releases graduais

### 22.7 Performance Benchmarks

**Throughput (capacidade demonstrada):**

- 10,000 transações PIX/segundo
- 100,000 requisições API/segundo
- 50,000 concurrent connections/instância

**Latência (p95):**

- POST /v1/pix/payments: 287ms
- GET /v1/pix/payments/{id}: 67ms
- POST /v1/pix/qrcodes/dynamic: 134ms
- GET /v1/pix/qrcodes/{id}: 34ms

**Database:** Query p95 < 50ms, 100 connections/instância

**Ponto de Operação Recomendado:** 75% da capacidade máxima (75,000 RPS) para margem de segurança

---

## 23. Completude Técnica - JSON Schemas e Códigos de Erro

### 23.1 JSON Schemas Completos

Schemas detalhados para validação de requisições seguindo JSON Schema Draft 2020-12.

#### 23.1.1 POST /v1/pix/payments (Enviar PIX)

**Request Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://api.lbpay.com.br/schemas/pix-payment-request.json",
  "type": "object",
  "required": ["valor", "destinatario", "pagador"],
  "additionalProperties": false,
  "properties": {
    "valor": {
      "type": "number",
      "format": "decimal",
      "minimum": 0.01,
      "maximum": 500000.00,
      "multipleOf": 0.01,
      "description": "Valor em BRL (reais)",
      "examples": [100.50, 1500.00, 50.25]
    },
    "descricao": {
      "type": "string",
      "minLength": 1,
      "maxLength": 140,
      "pattern": "^[a-zA-Z0-9\\s\\-\\.\\,\\(\\)]*$",
      "description": "Descrição da transação (alfanumérico)",
      "examples": ["Pagamento de serviço", "Compra produto XYZ"]
    },
    "external_id": {
      "type": "string",
      "minLength": 1,
      "maxLength": 50,
      "pattern": "^[a-zA-Z0-9\\-_]*$",
      "description": "ID externo para idempotência",
      "examples": ["pedido-12345", "invoice_abc123"]
    },
    "destinatario": {
      "type": "object",
      "required": ["chave_pix", "tipo_chave"],
      "additionalProperties": false,
      "properties": {
        "chave_pix": {
          "type": "string",
          "description": "Chave PIX do destinatário",
          "examples": ["email@exemplo.com", "12345678900", "+5511999999999", "123e4567-e89b-12d3-a456-426614174000"]
        },
        "tipo_chave": {
          "type": "string",
          "enum": ["email", "cpf", "cnpj", "telefone", "evp"],
          "description": "Tipo da chave PIX"
        }
      }
    },
    "pagador": {
      "type": "object",
      "required": ["cpf", "conta_id"],
      "additionalProperties": false,
      "properties": {
        "nome": {
          "type": "string",
          "minLength": 3,
          "maxLength": 100,
          "pattern": "^[a-zA-ZÀ-ÿ\\s]*$",
          "description": "Nome completo do pagador",
          "examples": ["João Silva", "Maria Santos"]
        },
        "cpf": {
          "type": "string",
          "pattern": "^\\d{11}$",
          "description": "CPF do pagador (apenas números)",
          "examples": ["12345678900"]
        },
        "conta_id": {
          "type": "string",
          "pattern": "^acc_[a-zA-Z0-9]{10,}$",
          "description": "ID da conta na plataforma LB Pay",
          "examples": ["acc_abc123xyz789"]
        }
      }
    }
  }
}
```

**Response Schema (201 Created):**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://api.lbpay.com.br/schemas/pix-payment-response.json",
  "type": "object",
  "required": ["id", "end_to_end_id", "valor", "status", "horario"],
  "properties": {
    "id": {
      "type": "string",
      "pattern": "^pix_pay_[a-zA-Z0-9]{10,}$",
      "description": "ID interno da transação",
      "examples": ["pix_pay_abc123xyz789"]
    },
    "end_to_end_id": {
      "type": "string",
      "pattern": "^E\\d{8}\\d{14}[a-zA-Z0-9]{10}$",
      "minLength": 32,
      "maxLength": 32,
      "description": "Identificador E2E do PIX",
      "examples": ["E12345678202501241030ABC123XYZ"]
    },
    "external_id": {
      "type": "string",
      "description": "ID externo fornecido na requisição"
    },
    "valor": {
      "type": "number",
      "format": "decimal"
    },
    "status": {
      "type": "string",
      "enum": ["EM_PROCESSAMENTO", "REALIZADO", "NAO_REALIZADO"],
      "description": "Status atual do pagamento"
    },
    "horario": {
      "type": "object",
      "required": ["solicitacao"],
      "properties": {
        "solicitacao": {
          "type": "string",
          "format": "date-time",
          "description": "Horário da solicitação (ISO 8601)"
        },
        "liquidacao": {
          "type": "string",
          "format": "date-time",
          "description": "Horário da liquidação (se aplicável)"
        }
      }
    },
    "pagador": {
      "type": "object",
      "properties": {
        "nome": {"type": "string"},
        "cpf": {"type": "string"}
      }
    },
    "destinatario": {
      "type": "object",
      "properties": {
        "nome": {"type": "string"},
        "cpf": {"type": "string"},
        "chave_pix": {"type": "string"},
        "banco": {
          "type": "object",
          "properties": {
            "ispb": {"type": "string"},
            "nome": {"type": "string"}
          }
        }
      }
    }
  }
}
```

#### 23.1.2 POST /v1/pix/qrcodes/dynamic (QR Code Dinâmico)

**Request Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["valor", "chave_pix"],
  "properties": {
    "valor": {
      "type": "number",
      "minimum": 0.01,
      "maximum": 500000.00,
      "multipleOf": 0.01
    },
    "chave_pix": {
      "type": "string",
      "description": "Chave PIX do recebedor"
    },
    "descricao": {
      "type": "string",
      "maxLength": 140
    },
    "expiracao_em_segundos": {
      "type": "integer",
      "minimum": 60,
      "maximum": 86400,
      "default": 3600,
      "description": "Tempo de validade do QR Code (60s a 24h)"
    },
    "devedor": {
      "type": "object",
      "properties": {
        "cpf": {"type": "string", "pattern": "^\\d{11}$"},
        "cnpj": {"type": "string", "pattern": "^\\d{14}$"},
        "nome": {"type": "string", "maxLength": 100}
      }
    },
    "valor_adicional": {
      "type": "object",
      "properties": {
        "modalidade": {
          "type": "string",
          "enum": ["PERMITE", "NAO_PERMITE"]
        }
      }
    }
  }
}
```

#### 23.1.3 POST /v1/pix/receipts/{endToEndId}/refunds (Devolução)

**Request Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["valor"],
  "properties": {
    "valor": {
      "type": "number",
      "minimum": 0.01,
      "description": "Valor a ser devolvido (deve ser <= valor original)"
    },
    "motivo": {
      "type": "string",
      "enum": [
        "ERRO_OPERACIONAL",
        "FRAUDE",
        "SOLICITACAO_PAGADOR",
        "DEVOLUCAO_PARCIAL",
        "OUTROS"
      ],
      "description": "Motivo da devolução"
    },
    "descricao": {
      "type": "string",
      "maxLength": 140,
      "description": "Descrição adicional do motivo"
    }
  }
}
```

#### 23.1.4 POST /v1/transfers/internal (Transferência Interna)

**Request Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["valor", "conta_origem_id", "conta_destino_id", "tipo"],
  "properties": {
    "valor": {
      "type": "number",
      "minimum": 0.01,
      "multipleOf": 0.01
    },
    "conta_origem_id": {
      "type": "string",
      "pattern": "^acc_[a-zA-Z0-9]{10,}$"
    },
    "conta_destino_id": {
      "type": "string",
      "pattern": "^acc_[a-zA-Z0-9]{10,}$"
    },
    "tipo": {
      "type": "string",
      "enum": [
        "OWNER_TO_TRANSACTIONAL",
        "TRANSACTIONAL_TO_OWNER",
        "OWNER_TO_OPERATIONAL",
        "OPERATIONAL_TO_OWNER"
      ],
      "description": "Tipo de transferência interna"
    },
    "descricao": {
      "type": "string",
      "maxLength": 200
    },
    "external_id": {
      "type": "string",
      "maxLength": 50,
      "pattern": "^[a-zA-Z0-9\\-_]*$"
    }
  }
}
```

#### 23.1.5 PUT /v1/webhooks/{chave} (Configurar Webhook)

**Request Schema:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["webhookUrl"],
  "properties": {
    "webhookUrl": {
      "type": "string",
      "format": "uri",
      "pattern": "^https://",
      "description": "URL HTTPS para receber notificações",
      "examples": ["https://seu-sistema.com.br/webhooks/pix"]
    },
    "mtls_required": {
      "type": "boolean",
      "default": true,
      "description": "Exigir TLS mútuo"
    },
    "events": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": [
          "PIX_RECEIVED",
          "PIX_SENT_COMPLETED",
          "PIX_SENT_FAILED",
          "REFUND_RECEIVED",
          "REFUND_COMPLETED"
        ]
      },
      "minItems": 1,
      "uniqueItems": true,
      "description": "Eventos a serem notificados"
    }
  }
}
```

### 23.2 Códigos de Erro Granulares

Tabela completa de códigos de erro seguindo padrões ISO 20022 e RFC 7807.

| Código | HTTP | Categoria | Descrição | Ação Sugerida | Retry |
|--------|------|-----------|-----------|---------------|-------|
| `invalid_request` | 400 | Validação | Requisição malformada ou inválida | Verificar formato JSON e campos obrigatórios | Não |
| `invalid_value` | 400 | Validação | Valor de campo não atende validações | Corrigir valor do campo específico | Não |
| `missing_field` | 400 | Validação | Campo obrigatório ausente | Adicionar campo obrigatório | Não |
| `invalid_json` | 400 | Formato | JSON inválido ou malformado | Validar sintaxe JSON | Não |
| `invalid_content_type` | 400 | Headers | Content-Type deve ser application/json | Definir header correto | Não |
| `field_too_long` | 400 | Validação | Campo excede comprimento máximo | Reduzir tamanho do campo | Não |
| `field_too_short` | 400 | Validação | Campo abaixo do comprimento mínimo | Aumentar tamanho do campo | Não |
| `invalid_format` | 400 | Validação | Formato do campo inválido (regex) | Corrigir formato conforme padrão | Não |
| `authentication_failed` | 401 | Autenticação | Token inválido ou expirado | Renovar token OAuth | Não |
| `token_expired` | 401 | Autenticação | Access token expirado | Usar refresh token para renovar | Não |
| `token_revoked` | 401 | Autenticação | Token foi revogado | Autenticar novamente | Não |
| `invalid_credentials` | 401 | Autenticação | Client ID ou Secret inválidos | Verificar credenciais | Não |
| `certificate_invalid` | 401 | Autenticação | Certificado mTLS inválido | Atualizar certificado | Não |
| `certificate_expired` | 401 | Autenticação | Certificado mTLS expirado | Renovar certificado | Não |
| `insufficient_permissions` | 403 | Autorização | Scope necessário não presente | Solicitar scope adequado | Não |
| `forbidden` | 403 | Autorização | Operação não permitida para este cliente | Contatar suporte | Não |
| `ip_not_whitelisted` | 403 | Segurança | IP não autorizado | Adicionar IP ao whitelist | Não |
| `account_suspended` | 403 | Conta | Conta suspensa por violação | Contatar compliance | Não |
| `resource_not_found` | 404 | Recurso | Recurso não encontrado | Verificar ID | Não |
| `endpoint_not_found` | 404 | Rota | Endpoint não existe | Verificar URL | Não |
| `pix_not_found` | 404 | PIX | Transação PIX não encontrada | Verificar endToEndId | Não |
| `qrcode_not_found` | 404 | QR Code | QR Code não encontrado | Verificar ID do QR Code | Não |
| `account_not_found` | 404 | Conta | Conta não encontrada | Verificar ID da conta | Não |
| `method_not_allowed` | 405 | HTTP | Método HTTP não permitido | Usar método correto (GET/POST/etc) | Não |
| `conflict` | 409 | Estado | Conflito de estado do recurso | Verificar estado atual | Não |
| `duplicate_transaction` | 409 | Idempotência | external_id já usado | Usar novo external_id | Não |
| `duplicate_qrcode` | 409 | QR Code | QR Code já existe para este txid | Usar novo txid | Não |
| `request_timeout` | 408 | Timeout | Requisição excedeu tempo limite | Tentar novamente | Sim |
| `insufficient_balance` | 422 | Saldo | Saldo insuficiente para transação | Adicionar fundos | Não |
| `invalid_key` | 422 | DICT | Chave PIX não existe no DICT | Verificar chave | Não |
| `invalid_ownership` | 422 | Compliance | CPF/CNPJ de titularidade diferente | Usar conta de mesma titularidade | Não |
| `account_blocked` | 422 | Conta | Conta com restrição operacional | Contatar suporte | Não |
| `limit_exceeded` | 422 | Limites | Transação acima do limite permitido | Reduzir valor ou aumentar limite | Não |
| `daily_limit_exceeded` | 422 | Limites | Limite diário excedido | Aguardar próximo dia | Não |
| `value_too_low` | 422 | Validação | Valor abaixo do mínimo (R$ 0,01) | Aumentar valor | Não |
| `value_too_high` | 422 | Validação | Valor acima do máximo | Reduzir valor | Não |
| `invalid_cpf` | 422 | Validação | CPF inválido (dígito verificador) | Corrigir CPF | Não |
| `invalid_cnpj` | 422 | Validação | CNPJ inválido (dígito verificador) | Corrigir CNPJ | Não |
| `qrcode_expired` | 422 | QR Code | QR Code expirado | Gerar novo QR Code | Não |
| `pix_already_refunded` | 422 | Devolução | PIX já foi devolvido | Não é possível devolver novamente | Não |
| `refund_value_exceeded` | 422 | Devolução | Valor de devolução > valor original | Reduzir valor de devolução | Não |
| `webhook_url_invalid` | 422 | Webhook | URL de webhook inválida ou inacessível | Corrigir URL | Não |
| `mtls_handshake_failed` | 422 | Webhook | Falha no handshake TLS mútuo | Verificar certificado do servidor | Não |
| `rate_limit_exceeded` | 429 | Rate Limit | Muitas requisições | Aguardar e retry com backoff | Sim |
| `rate_limit_ip_exceeded` | 429 | Rate Limit | Rate limit por IP excedido | Aguardar desbloqueio | Sim |
| `internal_error` | 500 | Servidor | Erro interno no processamento | Tentar novamente mais tarde | Sim |
| `database_error` | 500 | Servidor | Erro no banco de dados | Tentar novamente mais tarde | Sim |
| `spi_unavailable` | 503 | Dependência | Sistema PIX (SPI) indisponível | Aguardar disponibilidade | Sim |
| `dict_unavailable` | 503 | Dependência | Serviço DICT indisponível | Retry com backoff exponencial | Sim |
| `service_unavailable` | 503 | Servidor | Serviço temporariamente indisponível | Aguardar e tentar novamente | Sim |
| `gateway_timeout` | 504 | Timeout | Timeout na comunicação com SPI/DICT | Tentar novamente | Sim |
| `circuit_breaker_open` | 503 | Resiliência | Circuit breaker aberto para dependência | Aguardar fechamento automático | Sim |

**Exemplo de Resposta de Erro (RFC 7807):**

```json
{
  "type": "https://api.lbpay.com.br/errors/insufficient_balance",
  "title": "Saldo Insuficiente",
  "status": 422,
  "detail": "A conta origem não possui saldo suficiente para realizar esta transação. Saldo disponível: R$ 50,00. Valor solicitado: R$ 100,50",
  "instance": "/v1/pix/payments/pix_pay_abc123",
  "trace_id": "trace_def456uvw012",
  "request_id": "req_ghi789rst345",
  "timestamp": "2025-01-24T10:30:45.123Z",
  "errors": [
    {
      "field": "valor",
      "code": "insufficient_balance",
      "message": "Saldo disponível (R$ 50,00) é menor que o valor solicitado (R$ 100,50)"
    }
  ],
  "saldo_disponivel": 50.00,
  "valor_solicitado": 100.50,
  "sugestao": "Adicione R$ 50,50 ou mais à conta antes de tentar novamente"
}
```

**Exemplo com Múltiplos Erros de Validação:**

```json
{
  "type": "https://api.lbpay.com.br/errors/invalid_request",
  "title": "Requisição Inválida",
  "status": 400,
  "detail": "A requisição contém 3 erros de validação",
  "instance": "/v1/pix/payments",
  "trace_id": "trace_abc123",
  "request_id": "req_xyz789",
  "timestamp": "2025-01-24T10:30:45.123Z",
  "errors": [
    {
      "field": "valor",
      "code": "invalid_value",
      "message": "Valor deve ser maior que R$ 0,01",
      "received": -10.50,
      "expected": "> 0.01"
    },
    {
      "field": "destinatario.chave_pix",
      "code": "missing_field",
      "message": "Campo obrigatório ausente"
    },
    {
      "field": "pagador.cpf",
      "code": "invalid_format",
      "message": "CPF deve conter exatamente 11 dígitos numéricos",
      "received": "123.456.789-00",
      "expected_pattern": "^\\d{11}$"
    }
  ]
}
```

---

## 24. Validações de Campos e HTTP Headers

### 24.1 Validações de Campos Críticos

#### 24.1.1 CPF (Pessoa Física)

**Formato:** 11 dígitos numéricos
**Validação:** Dígito verificador (algoritmo módulo 11)
**Regex:** `^\d{11}$`
**Exemplo válido:** `12345678900`
**Exemplos inválidos:** `123.456.789-00` (com formatação), `12345678901` (dígito verificador inválido)

**Algoritmo de Validação:**

> TODO

```text
1. Verificar se tem 11 dígitos
2. Rejeitar sequências iguais (00000000000, 11111111111, etc.)
3. Calcular primeiro dígito verificador (posições 1-9)
4. Calcular segundo dígito verificador (posições 1-10)
5. Comparar com dígitos fornecidos
```

#### 24.1.2 CNPJ (Pessoa Jurídica)

**Formato:** 14 dígitos numéricos
**Validação:** Dígito verificador (algoritmo específico)
**Regex:** `^\d{14}$`
**Exemplo válido:** `12345678000190`
**Exemplos inválidos:** `12.345.678/0001-90` (com formatação)

#### 24.1.3 Chave PIX - Email

**Formato:** Email válido RFC 5322
**Max length:** 77 caracteres
**Regex:** `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
**Exemplo válido:** `usuario@exemplo.com.br`
**Exemplos inválidos:** `usuario@exemplo`, `@exemplo.com`, `usuario@.com`

#### 24.1.4 Chave PIX - Telefone

**Formato:** +55 + DDD (2 dígitos) + Número (8 ou 9 dígitos)
**Regex:** `^\+55\d{10,11}$`
**Exemplo válido:** `+5511999999999` (celular) ou `+551133334444` (fixo)
**Exemplos inválidos:** `11999999999` (sem +55), `+55 11 99999-9999` (com formatação)

#### 24.1.5 Chave PIX - EVP (Chave Aleatória)

**Formato:** UUID v4
**Regex:** `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`
**Exemplo válido:** `123e4567-e89b-12d3-a456-426614174000`
**Nota:** Sempre lowercase, versão 4 (quarto grupo começa com '4')

#### 24.1.6 End-to-End ID (E2E)

**Formato:** E + ISPB (8) + Data (8) + Hora (6) + uniqueId (10)
**Length:** Exatamente 32 caracteres
**Pattern:** `^E\d{8}\d{8}\d{6}[a-zA-Z0-9]{10}$`
**Exemplo válido:** `E12345678202501241030ABC123XYZ0`
**Componentes:**

- `E`: Fixo
- `12345678`: ISPB da instituição (8 dígitos)
- `20250124`: Data YYYYMMDD
- `103045`: Hora HHMMss
- `ABC123XYZ0`: Identificador único (10 caracteres alfanuméricos)

#### 24.1.7 TxID (Transaction ID)

**Formato:** 26-35 caracteres alfanuméricos
**Pattern:** `^[a-zA-Z0-9]{26,35}$`
**Exemplo válido:** `ABCDEFGHIJKLMNOPQRSTUVWXYZ`
**Regras:**

- Deve ser único por chave PIX do recebedor
- Apenas letras e números
- Não pode conter caracteres especiais

#### 24.1.8 Valores Monetários

**Tipo:** Decimal com 2 casas decimais
**Mínimo:** R$ 0,01
**Máximo:** R$ 500.000,00 (limite padrão, configurável)
**Format:** Usar ponto como separador decimal
**Exemplos válidos:** `100.50`, `1500.00`, `0.01`
**Exemplos inválidos:** `100,50` (vírgula), `100.5` (1 casa decimal), `0.001` (3 casas)

### 24.2 HTTP Headers Completos

#### 24.2.1 Request Headers

**Obrigatórios:**

```html
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Recomendados:**

```html
X-Idempotency-Key: {uuid}
X-Correlation-ID: {uuid}
User-Agent: {client-name}/{version}
Accept: application/json
Accept-Encoding: gzip, deflate, br
Accept-Language: pt-BR,pt;q=0.9,en;q=0.8
```

**Opcionais (para TLS mútuo):**

```html
X-Client-Certificate-Thumbprint: {sha256-hash}
```

**Exemplo Completo:**

```html
POST /v1/pix/payments HTTP/1.1
Host: api.lbpay.com.br
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json; charset=utf-8
X-API-Version: 1.0
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7
User-Agent: LBPay-SDK/1.0.0 (Java/17.0.1; Linux/x86_64)
Accept: application/json
Accept-Encoding: gzip, deflate, br
Accept-Language: pt-BR
Content-Length: 256
```

#### 24.2.2 Response Headers

**Sempre presentes:**

```html
Content-Type: application/json; charset=utf-8
X-Request-ID: req_{uuid}
X-Response-Time: {duration_ms}ms
Date: {RFC7231-date}
```

**Rate Limiting:**

```html
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 487
X-RateLimit-Reset: 1706097600
X-RateLimit-Policy: 500-per-minute
X-RateLimit-Tier: professional
```

**Segurança:**

```html
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

**Caching:**

```html
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache
Expires: 0
```

(Para operações sensíveis. Para consultas, pode usar caching apropriado)

**Tracing:**

```html
X-Trace-ID: trace_{uuid}
X-Correlation-ID: {client-provided-correlation-id}
X-Server-ID: srv-prod-pix-01
X-Region: sa-east-1
```

**Exemplo Completo (Sucesso):**

```html
HTTP/1.1 201 Created
Date: Fri, 24 Jan 2025 13:30:45 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 512
X-Request-ID: req_abc123xyz789
X-Trace-ID: trace_def456uvw012
X-Correlation-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Response-Time: 1234ms
X-Server-ID: srv-prod-pix-01
X-Region: sa-east-1
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 487
X-RateLimit-Reset: 1706097600
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Cache-Control: no-store
Location: /v1/pix/payments/pix_pay_abc123xyz789
```

**Exemplo de Resposta de Erro:**

```html
HTTP/1.1 422 Unprocessable Entity
Date: Fri, 24 Jan 2025 13:30:45 GMT
Content-Type: application/problem+json; charset=utf-8
Content-Length: 324
X-Request-ID: req_xyz789abc123
X-Trace-ID: trace_uvw456def789
X-Response-Time: 45ms
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
```

### 24.3 Webhooks - Políticas de Retry Completas

#### 24.3.1 Retry Policy Detalhada

**Configuração:**

- Total de tentativas: 5
- Algoritmo: Exponential backoff with jitter
- Timeout por tentativa: 10 segundos
- Jitter: 20% (aleatoriedade)

**Schedule de Tentativas:**

| Tentativa | Delay Base | Jitter | Delay Real (Range) | Timeout |
|-----------|------------|--------|-------------------|---------|
| 1 | 0s (imediato) | N/A | 0s | 10s |
| 2 | 60s | ±20% | 48s - 72s | 10s |
| 3 | 300s (5min) | ±20% | 240s - 360s | 10s |
| 4 | 900s (15min) | ±20% | 720s - 1080s | 10s |
| 5 | 3600s (1h) | ±20% | 2880s - 4320s | 10s |

**Códigos HTTP que acionam retry:**

- 408 Request Timeout
- 425 Too Early
- 429 Too Many Requests
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout
- Network errors (connection refused, timeout, etc.)

**Códigos HTTP que NÃO acionam retry:**

- 200-299 Success
- 400 Bad Request
- 401 Unauthorized
- 403 Forbidden
- 404 Not Found
- 410 Gone

#### 24.3.2 Dead Letter Queue (DLQ)

**Critérios para DLQ:**

- 5 tentativas falharam
- Webhook URL retorna 410 Gone (removed)
- Erro de validação de certificado TLS

**Acesso à DLQ:**

`GET /v1/webhooks/failed-events`

```json
{
  "failed_events": [
    {
      "event_id": "evt_failed_abc123",
      "webhook_url_hash": "sha256:aabbccdd...",
      "event_type": "PIX_RECEIVED",
      "attempts": 5,
      "first_attempt": "2025-01-24T10:30:45Z",
      "last_attempt": "2025-01-24T12:30:45Z",
      "last_error": {
        "code": "connection_timeout",
        "message": "Connection timeout after 10s"
      },
      "payload": {
        "end_to_end_id": "E12345678202501241030ABC123XYZ",
        "valor": 100.50
      },
      "expires_at": "2025-01-31T12:30:45Z"
    }
  ],
  "pagination": {
    "limit": 50,
    "has_more": false
  }
}
```

**Retry Manual:**

`POST /v1/webhooks/retry`

```json
{
  "event_id": "evt_failed_abc123",
  "override_url": "https://novo-endpoint.com/webhooks/pix"
}
```

#### 24.3.3 Webhook Security

**Signature Validation (HMAC-SHA256):**

Cada webhook inclui header de assinatura:

```html
X-Webhook-Signature: sha256=9a0b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b
X-Webhook-Timestamp: 1706097645
X-Webhook-ID: evt_abc123xyz789
```

**Validação no Cliente:**

```python
import hmac
import hashlib

def validate_webhook_signature(payload, signature, secret, timestamp):
    # Rejeitar se timestamp muito antigo (> 5 minutos)
    if abs(time.time() - timestamp) > 300:
        return False

    # Construir mensagem: timestamp + payload
    message = f"{timestamp}.{payload}"

    # Calcular HMAC-SHA256
    expected = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()

    # Comparação segura (timing-safe)
    return hmac.compare_digest(signature, f"sha256={expected}")
```

**TLS Mútuo (mTLS):**

- Certificado do cliente deve ser válido
- Emitido por CA confiável
- Não revogado (check CRL)
- Common Name (CN) deve estar na whitelist

**IP Whitelist (Opcional):**

```json
{
  "webhook_config": {
    "url": "https://seu-sistema.com/webhooks/pix",
    "ip_whitelist": [
      "203.0.113.0/24",
      "198.51.100.42"
    ],
    "require_mtls": true,
    "signature_algorithm": "hmac-sha256"
  }
}
```

---

## Conclusão

A **LB PIX API v1.0** oferece uma solução completa, moderna e robusta para operações PIX, combinando as melhores práticas das principais APIs do mercado bancário com recursos adicionais específicos para as necessidades do LB Pay.

**Destaques:**

- Design RESTful consistente e intuitivo
- Segurança OAuth2 + mTLS
- Rate limiting e proteção contra abuso
- Paginação eficiente baseada em cursor
- Webhooks confiáveis com assinatura
- Suporte completo a PIX Automático
- Split de pagamento flexível
- Conformidade regulatória (SPA/ME nº 615/2024)
- Documentação completa e exemplos práticos
- SDKs em múltiplas linguagens

A API está desenhada para atender desde pequenas integrações até operações de alto volume, mantendo performance, segurança e confiabilidade em todos os cenários de uso.

---

## FAQ

### 1. Como o usuário irá conseguir se autenticar nessa Api?

A gestão de acesso e autenticação na **LB PIX API** segue padrões de segurança modernos, combinando o fluxo OAuth 2.0 com requisitos rigorosos de mTLS (TLS mútuo). O método de autenticação principal exigido é o **OAuth 2.0 com Client Credentials Flow**.

### 2. Ele utilizará uma chave de api já gerenciada pela LB?

1. **Credenciais de Acesso (Chaves de API):** O cliente deve obter um `client_id` e um `client_secret`. O processo de obtenção dessas credenciais envolve acessar o **portal do desenvolvedor** (`https://developer.lbpay.com.br`) e **criar uma aplicação**.
2. **Obtenção do Token:** O cliente usará essas credenciais (`client_id` e `client_secret`) para solicitar um **JWT** (JSON Web Token) através do endpoint `/v1/oauth/token`, utilizando autenticação Basic.
3. **Validade do Token:** O Access Token (JWT) terá uma **expiração de 1 hora** e será necessária a renovação automática 5 minutos antes da expiração.
4. **Segurança Adicional (mTLS):** Para operações PIX, o **Certificado TLS mútuo (mTLS) é obrigatório**. Este certificado deve estar no formato P12 ou PEM, conforme o padrão BACEN.

### 3. Ele terá uma forma de gerar nova(s) chaves se quiser?

- **Geração/Gerenciamento de Chaves de API (Client ID/Secret):** O usuário deve criar uma aplicação e obter o `client_id` e `client_secret` através do **portal do desenvolvedor**. A especificação não detalha se o usuário pode gerar múltiplos pares de `client_id`/`client_secret` diretamente, mas o processo começa com a criação da aplicação no portal.
- **Gerenciamento de Certificados (Chave mTLS):** O cliente deve **gerar ou fazer o upload do certificado P12/PEM** necessário para o mTLS no portal do desenvolvedor.

### 4. Vai existir alguma tela onde ele irá gerenciar suas chaves e webhooks se for o caso?

O **portal do desenvolvedor** (`https://developer.lbpay.com.br`), onde as credenciais (`client_id`, `client_secret` e certificado) são obtidas e configuradas.

Embora o gerenciamento de webhooks e chaves PIX possa ser feito via API, os endpoints sugerem que esses recursos são controláveis pelo usuário:

- **Webhooks:** Existem endpoints específicos para o gerenciamento programático de webhooks:
  - **Configurar** um webhook (`PUT /v1/webhooks/{chave}`).
  - **Consultar** ou **Listar** os webhooks existentes.
  - **Deletar** um webhook.
  - O cliente pode configurar apenas 1 webhook por chave (limite padrão).
- **Chaves PIX:** O cliente pode gerenciar suas Chaves PIX através de endpoints, o que implica autonomia sobre esses recursos:
  - **Listar** Chaves PIX.
  - **Criar** Chave PIX Aleatória (EVP).
  - **Deletar** Chave PIX.

Ainda é necessário business definir se irá existir uma "tela" centralizada de gerenciamento no Portal do Desenvolvedor ou no Front atual para *configurar* chaves PIX e/ou *visualizar* webhooks.

### 5. O fluxo de gestão de acesso a essa api será novo ou não?

Sim, o **portal do desenvolvedor** (`https://developer.lbpay.com.br`).

### 6. Se o cliente usa a api pix já terá acesso a essa api também?

Precisamos avaliar, com pessoal de segurança, quais seriam as melhores práticas nesse cenário. Pois o fluxo de gestão de acesso a esta API é novo, padronizado com: **OAuth 2.0 com gestão de escopos e mTLS obrigatório**, que não necessariamente está contemplado na implementação atual (usando KeyCloak?).

- **Escopos (Permissões):** O acesso é controlado por escopos granulares que precisam ser configurados. Exemplos de escopos incluem `pix.read` (Leitura de operações PIX), `pix.write` (Criação e modificação de operações PIX), e `webhooks.write` (Configuração de webhooks). O cliente deve configurar os scopes necessários no portal do desenvolvedor.
- **Reaproveitamento de Credenciais:** As fontes não confirmam que um cliente que "usa a api pix" (anterior) **automaticamente** terá acesso a esta nova **LB PIX API v1.0**. Visto que esta é uma nova especificação que exige a criação de uma aplicação, obtenção de `client_id`, `client_secret` e upload/geração de certificados mTLS, o fluxo de integração e gestão de acesso deve ser redefinido e configurado pelo portal do desenvolvedor para esta nova API.

A autenticação é uma combinação de chaves de API (`client_id` + `client_secret`) para obter um token JWT, e um certificado mTLS obrigatório para a segurança das transações PIX, com o ponto de partida sendo o Portal do Desenvolvedor para gerenciamento de credenciais.

### 7. Ao iniciar um pix-out (envio de pix), como o cliente irá acompanhar a solicitação dele?

O cliente terá duas formas principais e complementares para acompanhar a solicitação de um PIX Out (envio de PIX) na **LB PIX API**: através de consultas síncronas (polling) utilizando endpoints específicos e via notificações assíncronas (webhooks).

#### 7.1. Acompanhamento via Consultas Síncronas (Polling)

O cliente pode verificar o status de uma transação enviada a qualquer momento utilizando o ID interno da transação ou o End-to-End ID (E2E ID), que é o identificador único de ponta a ponta.

Os endpoints disponíveis para consulta são:

1. **Consultar PIX Enviado por ID:** O cliente pode usar o ID interno da transação para consultar o status específico. O endpoint correspondente é `GET /v1/pix/payments/{id}`.
2. **Consultar PIX Enviado por End-to-End ID:** Para rastrear a transação usando o identificador universal do PIX, o End-to-End ID, o endpoint é `GET /v1/pix/payments/e2e/{endToEndId}`.
3. **Listar PIX Enviados e Filtrar por Status:** O cliente pode listar as transações enviadas em um determinado período e aplicar filtros por status, o que permite o acompanhamento em lote. Os filtros de status disponíveis são:

- **EM_PROCESSAMENTO**
- **REALIZADO**
- **NAO_REALIZADO**

A listagem suporta paginação baseada em cursor para gerenciar grandes volumes de dados.

Além disso, após a conclusão da transação, é possível **Gerar Recibo (PDF)** de uma transação pelo endpoint `GET /v1/accounts/{accountId}/receipts/{e2eId}`.

#### 7.2. Acompanhamento via Notificações Assíncronas (Webhooks)

O método mais recomendado para acompanhar o status final do envio é através de **webhooks**, que fornecem notificações assíncronas e confiáveis.

O fluxo de processamento de um PIX Out, após as validações e o envio para o SPI (se for PSP externo), inclui a **Notificação via webhook** antes do retorno do status.

O cliente deve configurar um webhook (limitado a **1 webhook por chave**) para receber os eventos relevantes, como:

- `pix.payment.completed`: Evento que notifica que o **Pagamento PIX foi concluído** com sucesso.
- `pix.payment.failed`: Evento que notifica que o **Pagamento PIX falhou**.

Para garantir a segurança, a URL do webhook deve ser **HTTPS** e o **TLS mútuo (mTLS) é obrigatório** para a sua configuração. Se o servidor do cliente estiver indisponível temporariamente, a API tem uma política de *retry* (tentativa de reenvio) detalhada.

| Método de Acompanhamento | Natureza | Vantagem |
| :--- | :--- | :--- |
| **Webhooks** | Assíncrono (Push) | Notificação em tempo real do status final (`completed` ou `failed`). |
| **Consultas GET** | Síncrono (Pull) | Permite verificar o status a qualquer momento ou obter o extrato completo de envios. |

### 8. E se ocorrer um erro no processo?

A **LB PIX API** foi projetada para lidar com erros de forma padronizada e granular, utilizando códigos HTTP semânticos e uma estrutura de erro consistente.

Se ocorrer um erro no processo de envio (PIX Out), o cliente será notificado de forma imediata (erros síncronos) ou assíncrona (falhas de processamento final).

Aqui está o detalhamento de como os erros são tratados:

#### 8.1. Respostas Síncronas (Códigos HTTP e Estrutura de Erro)

O primeiro ponto de falha é a própria chamada da API. A resposta utilizará códigos HTTP padronizados e uma estrutura de erro que detalha a causa da falha.

| Código HTTP | Categoria de Erro | Significado | Exemplo de Código Granular |
| :--- | :--- | :--- | :--- |
| **400 Bad Request** | Validação | Erro no formato ou conteúdo da requisição. | `invalid_request`, `missing_field`, `invalid_json`. |
| **401 Unauthorized** | Autenticação | Credenciais inválidas (token expirado ou ausente). | `token_expired`, `authentication_failed`, `certificate_invalid` (mTLS). |
| **403 Forbidden** | Autorização | Permissão insuficiente (escopo ausente) ou IP não autorizado. | `insufficient_permissions`, `ip_not_whitelisted`. |
| **422 Unprocessable Entity** | Regra de Negócio | Os dados são válidos, mas uma regra de negócio impediu a transação. | `insufficient_balance`, `invalid_key`, `invalid_ownership`, `limit_exceeded`. |
| **429 Too Many Requests** | Rate Limit | O cliente excedeu os limites de requisições por minuto ou por hora. | `rate_limit_exceeded`. |
| **500 Internal Server Error** | Servidor | Erro interno inesperado no sistema. | `internal_error`, `database_error`. |
| **503 Service Unavailable** | Dependência/Resiliência | Um serviço externo crítico (como SPI ou DICT) está indisponível. | `spi_unavailable`, `dict_unavailable`, `circuit_breaker_open`. |

##### 8.1.1 Erros Críticos no PIX Out (422)

Para o envio de PIX, os erros mais importantes que impedem a transação por regras de negócio incluem:

- **`insufficient_balance`**: Indica que o saldo na conta é insuficiente para cobrir o valor da transação.
- **`invalid_key`**: A chave PIX informada pelo cliente não existe no DICT.
- **`invalid_ownership`**: O CPF/CNPJ do pagador é diferente do titular da conta de destino, violando o requisito de **mesma titularidade** estabelecido pela Portaria SPA/ME nº 615/2024 para certas transações.
- **`limit_exceeded`**: A transação ultrapassou o valor máximo permitido para a conta ou o limite diário (`daily_limit_exceeded`).

#### 8.2. Notificação Assíncrona de Falha (Webhooks)

Se a requisição inicial for aceita (retorno **201 Created** ou **202 Accepted**), mas a transação falhar durante o processamento subsequente (por exemplo, falha na liquidação junto ao SPI), o cliente será notificado de forma assíncrona.

O evento de webhook `pix.payment.failed` é enviado para a URL configurada pelo cliente, indicando que o Pagamento PIX falhou.

#### 8.3. Ações Sugeridas e Política de Retry

A documentação da API fornece orientações sobre se o erro deve ser retentado:

- **Não Retentar (Non-Retryable):** Erros de validação (400, 422) e de autenticação/autorização (401, 403) geralmente são **erros do cliente** ou de regra de negócio e **não devem ser retentados**.
- **Retentar (Retryable):** Erros de timeout (408), Rate Limit (429) e erros de Servidor/Dependência (5xx, como 500 ou 503) sugerem indisponibilidade temporária e **devem ser retentados**.

A API implementa políticas de resiliência interna, incluindo **Circuit Breaker** e **Retry Policy** com *exponential backoff* para lidar com falhas em dependências externas, como o SPI e o DICT. Se o servidor do cliente não puder receber uma notificação de webhook de falha, há uma **Política de Retry Detalhada** que tenta reenvio até 5 vezes, com atrasos crescentes (por exemplo, 60 segundos na 2ª tentativa, até 3600 segundos na 5ª).

### 9. E se enviar a mesma solicitação, como é o tratamento idempotente?

A **LB PIX API** implementa o princípio da **idempotência** para operações críticas, como o envio de um PIX (`POST`), garantindo que a mesma solicitação processada mais de uma vez não cause efeitos colaterais adicionais no sistema.

Este tratamento é fundamental para lidar com cenários de falha de rede ou *timeouts*, permitindo que o cliente retente com segurança.

#### 9.1. Tratamento Idempotente de Solicitações

A idempotência em operações críticas (`POST`, `PUT`) é suportada através de uma chave de idempotência (`idempotency-key`).

##### Cenário 1: Solicitação JÁ Processada com Sucesso

Se o cliente enviar uma solicitação de PIX (envio) que já foi processada com sucesso anteriormente, a API irá identificar a duplicidade e retornar um erro, em vez de processar a transação novamente.

- **Resposta Esperada:** O sistema deve retornar o código **409 Conflict**.
- **Código de Erro Granular:** O código específico de erro é `duplicate_transaction`.
- **Detalhe:** Este erro ocorre porque o `external_id` (ID externo fornecido pelo cliente) já foi utilizado para uma transação.

##### Cenário 2: Retentar após um Erro Sistêmico (Timeout/5xx)

Se a primeira solicitação resultou em um erro que é **passível de retentativa** (como um *timeout* ou um erro de servidor 5xx), o cliente deve retentar a solicitação utilizando o **mesmo `external_id`** (chave de idempotência).

- Se o sistema conseguir processar a transação na retentativa, ou se a transação original tiver sido concluída mesmo após a falha de comunicação, o sistema deve garantir que o PIX seja liquidado apenas uma vez.
- Se o `external_id` for enviado novamente e a transação já estiver `REALIZADO`, a API retornará o `409 Conflict` (`duplicate_transaction`), informando ao cliente que a ação já foi concluída, e não executará o pagamento novamente.

#### 9.2. Tratamento de Erros de Negócio e Sistêmicos

A API distingue claramente entre erros sistêmicos (que podem ser retentados) e erros de negócio (que exigem correção por parte do cliente). O cliente é notificado através de códigos HTTP e códigos de erro granulares que sugerem a ação adequada.

##### Erros de Negócio (Não Retentáveis - 422 Unprocessable Entity)

Estes erros indicam que a transação viola uma regra de negócio ou está impedida pelo estado da conta, e não devem ser retentados sem a correção dos dados ou do status da conta.

- **Saldo Insuficiente:** Se o cliente não tiver fundos, será retornado o código **422 Unprocessable Entity** com o erro granular **`insufficient_balance`**. A ação sugerida é **Adicionar fundos**.
- **Conta Bloqueada:** Uma conta com restrição operacional resulta no erro **422** com o código **`account_blocked`**. O cliente deve **Contatar suporte**.
- **Limite Excedido:** Se a transação estiver acima do limite diário permitido, o código é **422** com `limit_exceeded` ou `daily_limit_exceeded`.
- **Titularidade Inválida (Compliance):** Ocorre o erro **422** com o código **`invalid_ownership`** se o CPF/CNPJ de titularidade for diferente do esperado, exigindo o uso de uma conta de mesma titularidade.

##### Erros Sistêmicos/Temporários (Retentáveis - 4xx e 5xx)

Estes erros indicam falhas temporárias ou de dependência, e a ação sugerida é **Tentar novamente**.

- **Rate Limit Excedido (429):** Indica excesso de requisições e a ação sugerida é **Aguardar e retry com backoff**.
- **Request Timeout (408):** Indica que a requisição excedeu o tempo limite e deve ser retentada.
- **Serviço Indisponível (503):** Falhas em dependências externas, como o SPI (`spi_unavailable`) ou o DICT (`dict_unavailable`), devem ser retentadas. A API implementa uma **Retry Policy** interna para dependências com *exponential backoff* e *jitter*.
