# Especificação Completa: PIX OUT B2B API

> **Versão:** 2.0
> **Data:** 2025-12-09
> **Autor:** Equipe LB Pay
> **Status:** Draft - Aguardando Aprovação

---

## Sumário Executivo

Esta especificação descreve a implementação **completa da API PIX OUT B2B** para clientes **Pessoa Jurídica (PJ)**. A API oferece múltiplas formas de envio de PIX (chave, conta, QR Code), validação de mesma titularidade, consultas avançadas e webhooks robustos.

### Funcionalidades Completas

1. ✅ **Envio de PIX por Chave PIX**
2. ✅ **Envio de PIX por Dados Bancários** (ISPB + Conta)
3. ✅ **Envio de PIX com Validação de Mesma Titularidade** (tabela interna LBPay)
4. ✅ **Iniciar Pagamento por QR Code** (PIX Copia e Cola)
5. ✅ **Detalhar QR Code PIX** (decode BR Code)
6. ✅ **Consulta por ID da Transação** (payment_id)
7. ✅ **Consulta por End-to-End ID** (e2e_id)
8. ✅ **Listar Pagamentos** (paginação + filtros)
9. ✅ **Webhooks** (notificações + persistência)
10. ✅ **Cancelamento de Pagamento**

---

## 1. Arquitetura e Componentes

### 1.1 Fluxo Arquitetural

```
Cliente PJ B2B
    ↓
API Gateway (Kong)
    ↓
Seamless PIX REST API (:8081)
    ├─ POST /v1/pix/payments (por chave)
    ├─ POST /v1/pix/payments/account (por conta)
    ├─ POST /v1/pix/payments/qrcode (por QR Code)
    ├─ POST /v1/pix/qrcodes/decode (detalhar QR Code)
    ├─ GET /v1/pix/payments/{payment_id}
    ├─ GET /v1/pix/payments/e2e/{e2e_id}
    ├─ GET /v1/pix/payments (listar)
    ├─ DELETE /v1/pix/payments/{payment_id}
    └─ Webhook callbacks
    ↓
Payment gRPC Service (:50051)
    └─ PixPaymentB2BService
    ↓
    ├─→ DICT Service (validação de chaves)
    ├─→ Same Ownership Validator (tabela interna LBPay)
    ├─→ QR Code Parser (BR Code decoder)
    ├─→ Ledger v2 (reserva de saldo)
    └─→ SPI/BACEN (envio de pagamentos)
    ↓
Webhook Delivery Service
    └─ Persistência em webhook_deliveries table
```

---

## 2. Especificação Completa da REST API

### 2.1 Enviar PIX por Chave PIX

**Endpoint:** `POST /v1/pix/payments`

**Request Body:**
```json
{
  "amount": "1500.50",
  "pix_key": "12345678901",
  "pix_key_type": "cpf",
  "description": "Pagamento fornecedor - NF 12345",
  "payer_info": {
    "account_id": "acc_abc123xyz",
    "name": "Empresa XYZ LTDA",
    "document": "12345678000190"
  },
  "validate_same_ownership": true,
  "callback_url": "https://empresa.com.br/webhooks/pix",
  "metadata": {
    "invoice_number": "NF-12345"
  }
}
```

**Response: 202 Accepted**
```json
{
  "payment_id": "pix_abc123xyz",
  "status": "pending",
  "amount": "1500.50",
  "pix_key": "12345678901",
  "created_at": "2025-01-24T10:30:00Z",
  "estimated_completion": "2025-01-24T10:30:30Z"
}
```

---

### 2.2 Enviar PIX por Dados Bancários (Conta)

**Endpoint:** `POST /v1/pix/payments/account`

**Descrição:** Permite enviar PIX informando ISPB, agência, conta e tipo de conta ao invés de chave PIX.

**Request Body:**
```json
{
  "amount": "2500.00",
  "description": "Pagamento fornecedor via conta",
  "payer_info": {
    "account_id": "acc_abc123xyz",
    "name": "Empresa XYZ LTDA",
    "document": "12345678000190"
  },
  "payee_account": {
    "ispb": "00000000",
    "branch": "0001",
    "account": "12345678",
    "account_type": "checking",
    "document": "98765432109",
    "name": "João Silva"
  },
  "validate_same_ownership": false,
  "callback_url": "https://empresa.com.br/webhooks/pix",
  "metadata": {
    "contract_id": "CONT-456"
  }
}
```

**Campos `payee_account`:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `ispb` | string | Sim | ISPB do banco (8 dígitos) |
| `branch` | string | Sim | Agência (4 dígitos) |
| `account` | string | Sim | Número da conta |
| `account_type` | enum | Sim | `checking`, `savings`, `payment` |
| `document` | string | Sim | CPF/CNPJ do titular |
| `name` | string | Sim | Nome do titular |

**Response: 202 Accepted**
```json
{
  "payment_id": "pix_def456uvw",
  "status": "pending",
  "amount": "2500.00",
  "payee_account": {
    "ispb": "00000000",
    "branch": "0001",
    "account": "12345678",
    "account_type": "checking"
  },
  "created_at": "2025-01-24T10:35:00Z"
}
```

---

### 2.3 Enviar PIX por QR Code (PIX Copia e Cola)

**Endpoint:** `POST /v1/pix/payments/qrcode`

**Descrição:** Inicia pagamento a partir de um BR Code (QR Code PIX).

**Request Body:**
```json
{
  "qrcode": "00020126580014br.gov.bcb.pix0136a74e0c32-84e3-4e65-9d3e-57f8fcac7e9f52040000530398654041500.505802BR5913Fulano de Tal6008BRASILIA62070503***63041D3D",
  "amount": "1500.50",
  "description": "Pagamento via QR Code",
  "payer_info": {
    "account_id": "acc_abc123xyz",
    "name": "Empresa XYZ LTDA",
    "document": "12345678000190"
  },
  "callback_url": "https://empresa.com.br/webhooks/pix"
}
```

**Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `qrcode` | string | Sim | BR Code completo (string EMV) |
| `amount` | string | Condicional | Obrigatório se QR Code não tiver valor fixo |
| `description` | string | Não | Descrição do pagamento |
| `payer_info` | object | Sim | Dados do pagador |
| `callback_url` | string | Não | URL para webhook |

**Validações:**
- QR Code deve ser válido (formato EMV)
- Se QR Code tiver valor fixo, `amount` não deve ser informado
- Se QR Code permitir valor variável, `amount` é obrigatório
- QR Code não expirado (validade < 24h)

**Response: 202 Accepted**
```json
{
  "payment_id": "pix_ghi789rst",
  "status": "pending",
  "amount": "1500.50",
  "qrcode_parsed": {
    "payee_name": "Fulano de Tal",
    "payee_city": "Brasília",
    "pix_key": "a74e0c32-84e3-4e65-9d3e-57f8fcac7e9f",
    "txid": "***",
    "amount": "1500.50"
  },
  "created_at": "2025-01-24T10:40:00Z"
}
```

---

### 2.4 Detalhar QR Code PIX (Decode BR Code)

**Endpoint:** `POST /v1/pix/qrcodes/decode`

**Descrição:** Decodifica um BR Code (PIX Copia e Cola) e retorna os detalhes em JSON sem efetuar o pagamento.

**Request Body:**
```json
{
  "qrcode": "00020126580014br.gov.bcb.pix0136a74e0c32-84e3-4e65-9d3e-57f8fcac7e9f52040000530398654041500.505802BR5913Fulano de Tal6008BRASILIA62070503***63041D3D"
}
```

**Response: 200 OK**
```json
{
  "qrcode": "00020126580014br.gov.bcb.pix...",
  "format": "emv",
  "type": "dynamic",
  "parsed_data": {
    "merchant_account_information": {
      "gui": "br.gov.bcb.pix",
      "pix_key": "a74e0c32-84e3-4e65-9d3e-57f8fcac7e9f",
      "url": null
    },
    "merchant_category_code": "0000",
    "transaction_currency": "986",
    "transaction_amount": "1500.50",
    "country_code": "BR",
    "merchant_name": "Fulano de Tal",
    "merchant_city": "BRASILIA",
    "additional_data": {
      "txid": "***"
    },
    "crc": "1D3D"
  },
  "validation": {
    "is_valid": true,
    "crc_valid": true,
    "is_expired": false,
    "expiration_date": "2025-01-25T10:40:00Z"
  },
  "payment_info": {
    "amount_fixed": true,
    "amount": "1500.50",
    "payee_name": "Fulano de Tal",
    "payee_city": "Brasília",
    "can_change_amount": false
  }
}
```

**Campos `type`:**
- `static` - QR Code estático (valor fixo ou variável)
- `dynamic` - QR Code dinâmico (vinculado a cobrança)

**Erros Possíveis:**
- `400 Bad Request` - QR Code inválido (formato ou CRC)
- `422 Unprocessable Entity` - QR Code expirado

---

### 2.5 Consultar Pagamento por ID

**Endpoint:** `GET /v1/pix/payments/{payment_id}`

**Response: 200 OK**
```json
{
  "payment_id": "pix_abc123xyz",
  "status": "completed",
  "amount": "1500.50",
  "pix_key": "12345678901",
  "pix_key_type": "cpf",
  "description": "Pagamento fornecedor - NF 12345",
  "payer_info": {
    "account_id": "acc_abc123xyz",
    "name": "Empresa XYZ LTDA",
    "document": "12345678000190"
  },
  "payee_info": {
    "name": "João Silva",
    "document": "12345678901",
    "bank": "001 - Banco do Brasil",
    "ispb": "00000000",
    "account_type": "checking"
  },
  "e2e_id": "E12345678202501241030ABC123XYZ",
  "txid": "TXB123ABC456DEF789GHI012JKL345MNO",
  "created_at": "2025-01-24T10:30:00Z",
  "processed_at": "2025-01-24T10:30:15Z",
  "completed_at": "2025-01-24T10:30:27Z",
  "metadata": {
    "invoice_number": "NF-12345"
  },
  "validation": {
    "same_ownership_required": true,
    "same_ownership_result": "valid",
    "same_ownership_source": "lbpay_internal_table",
    "pix_key_validated": true
  },
  "error": null
}
```

---

### 2.6 Consultar Pagamento por End-to-End ID

**Endpoint:** `GET /v1/pix/payments/e2e/{e2e_id}`

**Descrição:** Consulta um pagamento usando o End-to-End ID do BACEN.

**Exemplo:** `GET /v1/pix/payments/e2e/E12345678202501241030ABC123XYZ`

**Response: 200 OK**
```json
{
  "payment_id": "pix_abc123xyz",
  "status": "completed",
  "amount": "1500.50",
  "e2e_id": "E12345678202501241030ABC123XYZ",
  "txid": "TXB123ABC456DEF789GHI012JKL345MNO",
  "payer_info": {
    "name": "Empresa XYZ LTDA",
    "document": "12345678000190"
  },
  "payee_info": {
    "name": "João Silva",
    "document": "12345678901",
    "bank": "001 - Banco do Brasil"
  },
  "created_at": "2025-01-24T10:30:00Z",
  "completed_at": "2025-01-24T10:30:27Z"
}
```

**Erros:**
- `404 Not Found` - E2E ID não encontrado

---

### 2.7 Listar Pagamentos (com Paginação e Filtros)

**Endpoint:** `GET /v1/pix/payments`

**Query Parameters:**

| Parâmetro | Tipo | Descrição | Exemplo |
|-----------|------|-----------|---------|
| `account_id` | string | Filtrar por conta | `acc_abc123xyz` |
| `status` | enum | Filtrar por status | `completed`, `failed`, `pending` |
| `start_date` | ISO 8601 | Data início | `2025-01-01T00:00:00Z` |
| `end_date` | ISO 8601 | Data fim | `2025-01-31T23:59:59Z` |
| `pix_key` | string | Filtrar por chave PIX | `12345678901` |
| `e2e_id` | string | Filtrar por E2E ID | `E123...XYZ` |
| `min_amount` | decimal | Valor mínimo | `100.00` |
| `max_amount` | decimal | Valor máximo | `10000.00` |
| `cursor` | string | Cursor de paginação | `eyJpZCI6IjEyMyJ9` |
| `limit` | int | Items por página (max 100) | `50` |
| `sort` | enum | Ordenação | `created_at:desc`, `amount:asc` |

**Exemplo:**
```
GET /v1/pix/payments?account_id=acc_abc123&status=completed&start_date=2025-01-01T00:00:00Z&limit=50
```

**Response: 200 OK**
```json
{
  "data": [
    {
      "payment_id": "pix_abc123xyz",
      "status": "completed",
      "amount": "1500.50",
      "pix_key": "12345678901",
      "description": "Pagamento fornecedor",
      "payee_name": "João Silva",
      "e2e_id": "E12345678202501241030ABC123XYZ",
      "created_at": "2025-01-24T10:30:00Z",
      "completed_at": "2025-01-24T10:30:27Z"
    },
    {
      "payment_id": "pix_def456uvw",
      "status": "failed",
      "amount": "2300.00",
      "pix_key": "maria@email.com",
      "description": "Pagamento fornecedor",
      "payee_name": null,
      "e2e_id": null,
      "created_at": "2025-01-24T11:15:00Z",
      "completed_at": null,
      "error": {
        "code": "invalid_pix_key",
        "message": "Chave PIX não encontrada"
      }
    }
  ],
  "pagination": {
    "total": 245,
    "limit": 50,
    "has_more": true,
    "next_cursor": "eyJpZCI6InBpeF9naGkifQ=="
  },
  "filters_applied": {
    "account_id": "acc_abc123",
    "status": "completed",
    "start_date": "2025-01-01T00:00:00Z"
  }
}
```

---

### 2.8 Cancelar Pagamento

**Endpoint:** `DELETE /v1/pix/payments/{payment_id}`

**Response: 200 OK**
```json
{
  "payment_id": "pix_abc123xyz",
  "status": "cancelled",
  "cancelled_at": "2025-01-24T10:31:00Z",
  "message": "Pagamento cancelado com sucesso"
}
```

---

## 3. Validação de Mesma Titularidade

### 3.1 Tabela Interna LBPay

**Tabela:** `same_ownership_registry`

```sql
CREATE TABLE same_ownership_registry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pix_key VARCHAR(77) UNIQUE NOT NULL,
    pix_key_type VARCHAR(10) NOT NULL,
    document VARCHAR(14) NOT NULL,
    name VARCHAR(200) NOT NULL,
    bank_ispb VARCHAR(8) NOT NULL,
    bank_name VARCHAR(100) NOT NULL,
    account_type VARCHAR(20),

    -- Validation
    validated_at TIMESTAMP WITH TIME ZONE NOT NULL,
    validated_by VARCHAR(50),
    validation_source VARCHAR(50), -- 'dict', 'manual', 'import'

    -- Status
    is_active BOOLEAN DEFAULT true,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_same_ownership_pix_key (pix_key),
    INDEX idx_same_ownership_document (document),
    INDEX idx_same_ownership_ispb (bank_ispb)
);
```

### 3.2 Fluxo de Validação

1. **Cliente solicita pagamento** com `validate_same_ownership: true`
2. **Worker valida chave no DICT** → obtém `document_beneficiario`
3. **Worker consulta tabela interna**:
   ```sql
   SELECT * FROM same_ownership_registry
   WHERE pix_key = :pix_key
     AND document = :payer_document
     AND is_active = true
   ```
4. **Se encontrado** → Validação OK, prossegue
5. **Se não encontrado** → Marca pagamento como `failed` com erro `same_ownership_failed`

### 3.3 Endpoint Admin para Gerenciar Tabela

**Endpoint:** `POST /v1/admin/same-ownership-registry`

**Request:**
```json
{
  "pix_key": "12345678901",
  "pix_key_type": "cpf",
  "document": "12345678901",
  "name": "João Silva",
  "bank_ispb": "00000000",
  "bank_name": "Banco do Brasil",
  "validation_source": "manual"
}
```

**Endpoint:** `GET /v1/admin/same-ownership-registry?document=12345678901`

**Endpoint:** `DELETE /v1/admin/same-ownership-registry/{pix_key}`

---

## 4. Webhooks

### 4.1 Estrutura de Webhook

**Eventos Disponíveis:**

| Evento | Quando Disparar |
|--------|-----------------|
| `pix.payment.initiated` | Pagamento criado |
| `pix.payment.validating` | Iniciou validação |
| `pix.payment.processing` | Enviando para SPI |
| `pix.payment.completed` | Concluído com sucesso |
| `pix.payment.failed` | Falhou |
| `pix.payment.cancelled` | Cancelado pelo usuário |

### 4.2 Payload do Webhook

```json
{
  "event": "pix.payment.completed",
  "webhook_id": "wh_abc123xyz",
  "timestamp": "2025-01-24T10:30:27Z",
  "data": {
    "payment_id": "pix_abc123xyz",
    "status": "completed",
    "amount": "1500.50",
    "pix_key": "12345678901",
    "description": "Pagamento fornecedor",
    "payee_name": "João Silva",
    "payee_document": "12345678901",
    "e2e_id": "E12345678202501241030ABC123XYZ",
    "txid": "TXB123ABC456DEF789GHI012JKL345MNO",
    "created_at": "2025-01-24T10:30:00Z",
    "completed_at": "2025-01-24T10:30:27Z",
    "metadata": {
      "invoice_number": "NF-12345"
    }
  }
}
```

### 4.3 Segurança do Webhook

**Headers enviados:**
```
Content-Type: application/json
X-Webhook-Signature: sha256=abc123def456...
X-Webhook-Timestamp: 1706097600
X-Webhook-ID: wh_abc123xyz
X-Webhook-Event: pix.payment.completed
```

**Cálculo da Signature:**
```javascript
const payload = timestamp + "." + JSON.stringify(data);
const signature = crypto.createHmac('sha256', webhook_secret)
                        .update(payload)
                        .digest('hex');
```

### 4.4 Retry Policy

- **5 tentativas**: 0s, 60s, 5min, 15min, 1h
- **Jitter**: 20% de aleatoriedade nos delays
- **Timeout**: 10 segundos por tentativa
- **Dead Letter Queue**: Após 5 falhas

### 4.5 Persistência de Webhooks

**Tabela:** `webhook_deliveries`

```sql
CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id VARCHAR(50) UNIQUE NOT NULL,
    payment_id VARCHAR(50) NOT NULL,
    event VARCHAR(50) NOT NULL,
    callback_url TEXT NOT NULL,

    -- Request
    request_payload JSONB NOT NULL,
    request_headers JSONB,

    -- Response
    status_code INTEGER,
    response_body TEXT,
    response_headers JSONB,

    -- Timing
    attempt_number INTEGER DEFAULT 1,
    max_attempts INTEGER DEFAULT 5,
    sent_at TIMESTAMP WITH TIME ZONE,
    responded_at TIMESTAMP WITH TIME ZONE,
    next_retry_at TIMESTAMP WITH TIME ZONE,

    -- Status
    delivery_status VARCHAR(20) NOT NULL, -- 'pending', 'delivered', 'failed', 'dead_letter'
    error_message TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    INDEX idx_webhook_deliveries_payment_id (payment_id),
    INDEX idx_webhook_deliveries_status (delivery_status),
    INDEX idx_webhook_deliveries_next_retry (next_retry_at)
);
```

### 4.6 Consultar Webhooks Enviados

**Endpoint:** `GET /v1/pix/payments/{payment_id}/webhooks`

**Response: 200 OK**
```json
{
  "payment_id": "pix_abc123xyz",
  "webhooks": [
    {
      "webhook_id": "wh_abc123xyz",
      "event": "pix.payment.completed",
      "delivery_status": "delivered",
      "attempt_number": 1,
      "status_code": 200,
      "sent_at": "2025-01-24T10:30:28Z",
      "responded_at": "2025-01-24T10:30:28Z"
    },
    {
      "webhook_id": "wh_def456uvw",
      "event": "pix.payment.processing",
      "delivery_status": "delivered",
      "attempt_number": 1,
      "status_code": 200,
      "sent_at": "2025-01-24T10:30:15Z",
      "responded_at": "2025-01-24T10:30:15Z"
    }
  ]
}
```

### 4.7 Reenviar Webhook Manualmente

**Endpoint:** `POST /v1/pix/payments/{payment_id}/webhooks/retry`

**Request:**
```json
{
  "webhook_id": "wh_abc123xyz"
}
```

**Response: 200 OK**
```json
{
  "webhook_id": "wh_abc123xyz",
  "status": "retrying",
  "message": "Webhook será reenviado em instantes"
}
```

---

## 5. Modelo de Dados Completo

### 5.1 Tabela Principal: pix_payments_b2b

```sql
CREATE TABLE pix_payments_b2b (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id VARCHAR(50) UNIQUE NOT NULL,
    account_id VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,

    -- Payment details
    amount NUMERIC(15, 2) NOT NULL,
    description TEXT,

    -- Payment method (key, account, or qrcode)
    payment_method VARCHAR(20) NOT NULL, -- 'pix_key', 'account', 'qrcode'

    -- PIX Key (if payment_method = 'pix_key')
    pix_key VARCHAR(77),
    pix_key_type VARCHAR(10),

    -- Account details (if payment_method = 'account')
    payee_ispb VARCHAR(8),
    payee_branch VARCHAR(4),
    payee_account VARCHAR(20),
    payee_account_type VARCHAR(20),

    -- QR Code (if payment_method = 'qrcode')
    qrcode TEXT,
    qrcode_parsed JSONB,

    -- Payer info
    payer_name VARCHAR(200) NOT NULL,
    payer_document VARCHAR(14) NOT NULL,

    -- Payee info (populated after validation)
    payee_name VARCHAR(200),
    payee_document VARCHAR(14),
    payee_bank VARCHAR(100),

    -- Transaction IDs
    e2e_id VARCHAR(32),
    txid VARCHAR(35),

    -- Validation
    validate_same_ownership BOOLEAN DEFAULT false,
    same_ownership_result VARCHAR(20),
    same_ownership_source VARCHAR(50), -- 'lbpay_internal', 'dict', 'not_required'
    pix_key_validated BOOLEAN DEFAULT false,

    -- Webhook
    callback_url TEXT,

    -- Metadata
    metadata JSONB,

    -- Error
    error_code VARCHAR(50),
    error_message TEXT,
    error_type VARCHAR(50),
    error_timestamp TIMESTAMP WITH TIME ZONE,

    -- Idempotency
    idempotency_key VARCHAR(50) UNIQUE,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,

    -- Indexes
    INDEX idx_pix_payments_payment_id (payment_id),
    INDEX idx_pix_payments_account_id (account_id),
    INDEX idx_pix_payments_status (status),
    INDEX idx_pix_payments_created_at (created_at DESC),
    INDEX idx_pix_payments_e2e_id (e2e_id),
    INDEX idx_pix_payments_payer_document (payer_document),
    INDEX idx_pix_payments_payee_document (payee_document),
    INDEX idx_pix_payments_pix_key (pix_key)
);

-- Constraints
ALTER TABLE pix_payments_b2b
ADD CONSTRAINT chk_payment_status
CHECK (status IN ('pending', 'validating', 'processing', 'completed', 'failed', 'cancelled'));

ALTER TABLE pix_payments_b2b
ADD CONSTRAINT chk_payment_method
CHECK (payment_method IN ('pix_key', 'account', 'qrcode'));
```

---

## 6. Especificação gRPC - Atualizada

### 6.1 Proto Completo

**Arquivo:** `/apps/payment/proto/pix_payment_b2b.proto`

```protobuf
syntax = "proto3";

package payment.pixb2b;

import "google/protobuf/timestamp.proto";

option go_package = "github.com/london-bridge/money-moving/apps/payment/proto/pixb2b";

service PixPaymentB2BService {
  // Payment methods
  rpc InitiatePixPaymentByKey(InitiatePixPaymentByKeyRequest) returns (InitiatePixPaymentResponse);
  rpc InitiatePixPaymentByAccount(InitiatePixPaymentByAccountRequest) returns (InitiatePixPaymentResponse);
  rpc InitiatePixPaymentByQRCode(InitiatePixPaymentByQRCodeRequest) returns (InitiatePixPaymentResponse);

  // QR Code
  rpc DecodeQRCode(DecodeQRCodeRequest) returns (DecodeQRCodeResponse);

  // Query
  rpc GetPixPayment(GetPixPaymentRequest) returns (PixPaymentResponse);
  rpc GetPixPaymentByE2EId(GetPixPaymentByE2EIdRequest) returns (PixPaymentResponse);
  rpc ListPixPayments(ListPixPaymentsRequest) returns (ListPixPaymentsResponse);

  // Management
  rpc CancelPixPayment(CancelPixPaymentRequest) returns (CancelPixPaymentResponse);

  // Webhooks
  rpc ListWebhookDeliveries(ListWebhookDeliveriesRequest) returns (ListWebhookDeliveriesResponse);
  rpc RetryWebhookDelivery(RetryWebhookDeliveryRequest) returns (RetryWebhookDeliveryResponse);
}

// Messages for payment by key
message InitiatePixPaymentByKeyRequest {
  string amount = 1;
  string pix_key = 2;
  PixKeyType pix_key_type = 3;
  string description = 4;
  PayerInfo payer_info = 5;
  bool validate_same_ownership = 6;
  string callback_url = 7;
  map<string, string> metadata = 8;
  string idempotency_key = 9;
}

// Messages for payment by account
message InitiatePixPaymentByAccountRequest {
  string amount = 1;
  string description = 2;
  PayerInfo payer_info = 3;
  PayeeAccount payee_account = 4;
  bool validate_same_ownership = 5;
  string callback_url = 6;
  map<string, string> metadata = 7;
  string idempotency_key = 8;
}

message PayeeAccount {
  string ispb = 1;
  string branch = 2;
  string account = 3;
  string account_type = 4; // checking, savings, payment
  string document = 5;
  string name = 6;
}

// Messages for payment by QR Code
message InitiatePixPaymentByQRCodeRequest {
  string qrcode = 1;
  string amount = 2; // optional if QR has fixed amount
  string description = 3;
  PayerInfo payer_info = 4;
  string callback_url = 5;
  map<string, string> metadata = 6;
  string idempotency_key = 7;
}

// Decode QR Code
message DecodeQRCodeRequest {
  string qrcode = 1;
}

message DecodeQRCodeResponse {
  string qrcode = 1;
  string format = 2; // "emv"
  string type = 3; // "static", "dynamic"
  QRCodeParsedData parsed_data = 4;
  QRCodeValidation validation = 5;
  QRCodePaymentInfo payment_info = 6;
}

message QRCodeParsedData {
  MerchantAccountInfo merchant_account_information = 1;
  string merchant_category_code = 2;
  string transaction_currency = 3;
  string transaction_amount = 4;
  string country_code = 5;
  string merchant_name = 6;
  string merchant_city = 7;
  AdditionalData additional_data = 8;
  string crc = 9;
}

message MerchantAccountInfo {
  string gui = 1;
  string pix_key = 2;
  string url = 3;
}

message AdditionalData {
  string txid = 1;
}

message QRCodeValidation {
  bool is_valid = 1;
  bool crc_valid = 2;
  bool is_expired = 3;
  google.protobuf.Timestamp expiration_date = 4;
}

message QRCodePaymentInfo {
  bool amount_fixed = 1;
  string amount = 2;
  string payee_name = 3;
  string payee_city = 4;
  bool can_change_amount = 5;
}

// Common messages
enum PixKeyType {
  PIX_KEY_TYPE_UNSPECIFIED = 0;
  PIX_KEY_TYPE_CPF = 1;
  PIX_KEY_TYPE_CNPJ = 2;
  PIX_KEY_TYPE_EMAIL = 3;
  PIX_KEY_TYPE_PHONE = 4;
  PIX_KEY_TYPE_EVP = 5;
}

message PayerInfo {
  string account_id = 1;
  string name = 2;
  string document = 3;
}

message PayeeInfo {
  string name = 1;
  string document = 2;
  string bank = 3;
  string account_type = 4;
  string ispb = 5;
}

message InitiatePixPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  google.protobuf.Timestamp created_at = 3;
  google.protobuf.Timestamp estimated_completion = 4;
}

// Get payment
message GetPixPaymentRequest {
  string payment_id = 1;
}

message GetPixPaymentByE2EIdRequest {
  string e2e_id = 1;
}

message PixPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  string amount = 3;
  string payment_method = 4; // pix_key, account, qrcode
  string pix_key = 5;
  PixKeyType pix_key_type = 6;
  string description = 7;
  PayerInfo payer_info = 8;
  PayeeInfo payee_info = 9;
  string e2e_id = 10;
  string txid = 11;
  google.protobuf.Timestamp created_at = 12;
  google.protobuf.Timestamp processed_at = 13;
  google.protobuf.Timestamp completed_at = 14;
  map<string, string> metadata = 15;
  ValidationInfo validation = 16;
  PaymentError error = 17;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_STATUS_PENDING = 1;
  PAYMENT_STATUS_VALIDATING = 2;
  PAYMENT_STATUS_PROCESSING = 3;
  PAYMENT_STATUS_COMPLETED = 4;
  PAYMENT_STATUS_FAILED = 5;
  PAYMENT_STATUS_CANCELLED = 6;
}

message ValidationInfo {
  bool same_ownership_required = 1;
  string same_ownership_result = 2;
  string same_ownership_source = 3;
  bool pix_key_validated = 4;
}

message PaymentError {
  string code = 1;
  string message = 2;
  string type = 3;
  google.protobuf.Timestamp timestamp = 4;
}

// List payments
message ListPixPaymentsRequest {
  string account_id = 1;
  PaymentStatus status_filter = 2;
  google.protobuf.Timestamp start_date = 3;
  google.protobuf.Timestamp end_date = 4;
  string pix_key_filter = 5;
  string e2e_id_filter = 6;
  string cursor = 7;
  int32 limit = 8;
}

message ListPixPaymentsResponse {
  repeated PixPaymentSummary payments = 1;
  PaginationInfo pagination = 2;
}

message PixPaymentSummary {
  string payment_id = 1;
  PaymentStatus status = 2;
  string amount = 3;
  string pix_key = 4;
  string description = 5;
  string payee_name = 6;
  string e2e_id = 7;
  google.protobuf.Timestamp created_at = 8;
  google.protobuf.Timestamp completed_at = 9;
  PaymentError error = 10;
}

message PaginationInfo {
  int32 total = 1;
  int32 limit = 2;
  bool has_more = 3;
  string next_cursor = 4;
}

// Cancel
message CancelPixPaymentRequest {
  string payment_id = 1;
}

message CancelPixPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  google.protobuf.Timestamp cancelled_at = 3;
  string message = 4;
}

// Webhooks
message ListWebhookDeliveriesRequest {
  string payment_id = 1;
}

message ListWebhookDeliveriesResponse {
  string payment_id = 1;
  repeated WebhookDelivery webhooks = 2;
}

message WebhookDelivery {
  string webhook_id = 1;
  string event = 2;
  string delivery_status = 3;
  int32 attempt_number = 4;
  int32 status_code = 5;
  google.protobuf.Timestamp sent_at = 6;
  google.protobuf.Timestamp responded_at = 7;
}

message RetryWebhookDeliveryRequest {
  string payment_id = 1;
  string webhook_id = 2;
}

message RetryWebhookDeliveryResponse {
  string webhook_id = 1;
  string status = 2;
  string message = 3;
}
```

---

## 7. Resumo das Funcionalidades

| # | Funcionalidade | Endpoint | Método | Prioridade |
|---|----------------|----------|--------|------------|
| 1 | Enviar PIX por Chave | `/v1/pix/payments` | POST | Alta |
| 2 | Enviar PIX por Conta | `/v1/pix/payments/account` | POST | Alta |
| 3 | Enviar PIX por QR Code | `/v1/pix/payments/qrcode` | POST | Alta |
| 4 | Detalhar QR Code | `/v1/pix/qrcodes/decode` | POST | Média |
| 5 | Consultar por ID | `/v1/pix/payments/{id}` | GET | Alta |
| 6 | Consultar por E2E ID | `/v1/pix/payments/e2e/{e2e}` | GET | Média |
| 7 | Listar com Filtros | `/v1/pix/payments` | GET | Alta |
| 8 | Cancelar Pagamento | `/v1/pix/payments/{id}` | DELETE | Média |
| 9 | Validação Titularidade | (interno) | - | Alta |
| 10 | Webhooks + Persistência | (worker) | - | Alta |
| 11 | Consultar Webhooks | `/v1/pix/payments/{id}/webhooks` | GET | Baixa |
| 12 | Reenviar Webhook | `/v1/pix/payments/{id}/webhooks/retry` | POST | Baixa |

---

## 8. Atualização no Planejamento

Com as funcionalidades adicionais, o planejamento deve ser ajustado:

**Nova Duração:** 6-7 Sprints (12-14 semanas / 3-3.5 meses)

**Épicos Adicionais:**
- Épico 6: QR Code (decode + payment) - 2 sprints
- Épico 7: Validação de Titularidade com tabela interna - 1 sprint
- Épico 8: Webhooks avançados (persistência + retry manual) - 1 sprint

---

## Apêndices

### Apêndice A: Referências

- [API LB PIX v1.0 Specification](./API_LB.md)
- [BACEN - Manual do PIX](https://www.bcb.gov.br/estabilidadefinanceira/pix)
- [EMV QR Code Specification](https://www.emvco.com/emv-technologies/qrcodes/)

### Apêndice B: Changelog

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 2025-12-09 | Versão inicial - PIX unitário |
| 2.0 | 2025-12-09 | **Versão completa** - Todas funcionalidades do escopo |

---

**Fim da Especificação Completa**
