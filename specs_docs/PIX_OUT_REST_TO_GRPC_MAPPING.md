# Mapeamento Completo: REST API → gRPC Proto - PIX OUT B2B

> **Versão:** 1.0
> **Data:** 2025-12-09
> **Baseado em:** API_LB.md (Seção 2 - Operações PIX OUT)

---

## Sumário Executivo

Este documento mapeia **todos os 7 tipos de pagamento PIX OUT** da REST API (conforme API_LB.md) para as definições gRPC Proto necessárias. Inclui payloads JSON completos, validações, respostas de sucesso/erro e a especificação Proto necessária para o serviço Payment.

**Status Atual:** ⚠️ **Payment service NÃO possui proto definitions**. Toda a API é REST pura (Fiber).

---

## 1. Tipos de Pagamento PIX OUT Suportados

| # | Tipo | Identificador | Formato | Endpoint REST | gRPC Method |
|---|------|---------------|---------|---------------|-------------|
| 1 | **CPF** | `tipo_chave: "cpf"` | 11 dígitos | `POST /v1/pix/payments` | `InitiatePixPaymentByKey` |
| 2 | **CNPJ** | `tipo_chave: "cnpj"` | 14 dígitos | `POST /v1/pix/payments` | `InitiatePixPaymentByKey` |
| 3 | **Email** | `tipo_chave: "email"` | RFC 5322 | `POST /v1/pix/payments` | `InitiatePixPaymentByKey` |
| 4 | **Telefone** | `tipo_chave: "telefone"` | +55 DDD #### | `POST /v1/pix/payments` | `InitiatePixPaymentByKey` |
| 5 | **EVP** | `tipo_chave: "evp"` | UUID v4 | `POST /v1/pix/payments` | `InitiatePixPaymentByKey` |
| 6 | **Conta Bancária** | Dados bancários | ISPB+Agência+Conta | `POST /v1/pix/payments` | `InitiatePixPaymentByAccount` |
| 7 | **QR Code** | BR Code | EMV String | `POST /v1/pix/qrcodes/pay` | `InitiatePixPaymentByQRCode` |

---

## 2. Mapeamento Detalhado por Tipo

### 2.1 Pagamento por CPF

#### REST API Request (JSON)
```json
{
  "valor": 1500.50,
  "descricao": "Pagamento fornecedor",
  "external_id": "INV-12345",
  "destinatario": {
    "chave_pix": "12345678900",
    "tipo_chave": "cpf"
  },
  "pagador": {
    "nome": "Empresa XYZ LTDA",
    "cpf": "12345678000190",
    "conta_id": "acc_abc123"
  }
}
```

#### Validações
- `chave_pix`: 11 dígitos numéricos, check digit válido
- Regex: `^\d{11}$`
- Não aceitar CPF sequencial (00000000000, 11111111111, etc)
- Validar check digit (Modulo 11)

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_abc123xyz",
  "end_to_end_id": "E12345678202501241030ABC123XYZ",
  "external_id": "INV-12345",
  "valor": 1500.50,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "chave_pix": "123.456.789-00",
    "tipo_chave": "cpf",
    "nome": "João Silva",
    "cpf": "123.456.789-00",
    "banco": {
      "ispb": "00000000",
      "nome": "Banco do Brasil",
      "codigo": "001"
    }
  },
  "pagador": {
    "nome": "Empresa XYZ LTDA",
    "cpf": "123.456.789-00",
    "conta_id": "acc_abc123"
  },
  "horario": {
    "solicitacao": "2025-01-24T10:30:45Z"
  },
  "criado_em": "2025-01-24T10:30:45Z"
}
```

#### Error Response (422 - Invalid Key)
```json
{
  "error": "invalid_key",
  "error_description": "Chave PIX não encontrada no DICT",
  "type": "https://api.lbpay.com.br/errors/invalid_key",
  "status": 422,
  "detail": "CPF 12345678900 não está registrado como chave PIX no DICT",
  "timestamp": "2025-01-24T10:30:45Z",
  "path": "/v1/pix/payments",
  "request_id": "req_abc123xyz"
}
```

#### gRPC Proto Mapping
```protobuf
message InitiatePixPaymentByKeyRequest {
  string amount = 1;                    // "1500.50"
  string pix_key = 2;                   // "12345678900"
  PixKeyType pix_key_type = 3;          // PIX_KEY_TYPE_CPF
  string description = 4;               // "Pagamento fornecedor"
  string external_id = 5;               // "INV-12345"
  PayerInfo payer_info = 6;
  string callback_url = 7;
  map<string, string> metadata = 8;
  string idempotency_key = 9;
}

enum PixKeyType {
  PIX_KEY_TYPE_CPF = 1;
  // ... outros tipos
}

message PayerInfo {
  string account_id = 1;                // "acc_abc123"
  string name = 2;                      // "Empresa XYZ LTDA"
  string document = 3;                  // "12345678000190"
}
```

---

### 2.2 Pagamento por CNPJ

#### REST API Request (JSON)
```json
{
  "valor": 25000.00,
  "descricao": "Pagamento B2B",
  "external_id": "CONTRACT-789",
  "destinatario": {
    "chave_pix": "12345678000190",
    "tipo_chave": "cnpj"
  },
  "pagador": {
    "nome": "Empresa ABC LTDA",
    "cpf": "98765432000109",
    "conta_id": "acc_xyz789"
  }
}
```

#### Validações
- `chave_pix`: 14 dígitos numéricos, check digit válido
- Regex: `^\d{14}$`
- Validar check digit (algoritmo CNPJ)

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_def456uvw",
  "end_to_end_id": "E12345678202501241035DEF456UVW",
  "external_id": "CONTRACT-789",
  "valor": 25000.00,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "chave_pix": "12.345.678/0001-90",
    "tipo_chave": "cnpj",
    "nome": "Fornecedor XYZ S.A.",
    "cnpj": "12.345.678/0001-90",
    "banco": {
      "ispb": "60701190",
      "nome": "Itaú Unibanco",
      "codigo": "341"
    }
  },
  "horario": {
    "solicitacao": "2025-01-24T10:35:00Z"
  },
  "criado_em": "2025-01-24T10:35:00Z"
}
```

#### gRPC Proto Mapping
```protobuf
// Mesmo InitiatePixPaymentByKeyRequest
// com pix_key_type = PIX_KEY_TYPE_CNPJ
```

---

### 2.3 Pagamento por Email

#### REST API Request (JSON)
```json
{
  "valor": 750.00,
  "descricao": "Pagamento freelancer",
  "external_id": "PROJ-456",
  "destinatario": {
    "chave_pix": "joao.silva@email.com",
    "tipo_chave": "email"
  },
  "pagador": {
    "cpf": "12345678000190",
    "conta_id": "acc_abc123"
  }
}
```

#### Validações
- `chave_pix`: Email RFC 5322
- Regex: `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- Max length: 77 caracteres
- Não aceitar espaços ou caracteres especiais não permitidos

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_ghi789rst",
  "end_to_end_id": "E12345678202501241040GHI789RST",
  "external_id": "PROJ-456",
  "valor": 750.00,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "chave_pix": "joao.silva@email.com",
    "tipo_chave": "email",
    "nome": "João da Silva",
    "cpf": "987.654.321-00"
  },
  "horario": {
    "solicitacao": "2025-01-24T10:40:00Z"
  }
}
```

#### gRPC Proto Mapping
```protobuf
// Mesmo InitiatePixPaymentByKeyRequest
// com pix_key_type = PIX_KEY_TYPE_EMAIL
```

---

### 2.4 Pagamento por Telefone

#### REST API Request (JSON)
```json
{
  "valor": 300.00,
  "descricao": "Pagamento delivery",
  "destinatario": {
    "chave_pix": "+5511999887766",
    "tipo_chave": "telefone"
  },
  "pagador": {
    "cpf": "12345678000190",
    "conta_id": "acc_abc123"
  }
}
```

#### Validações
- `chave_pix`: Formato +55 DDD Número
- Regex: `^\+55\d{10,11}$`
- 10 dígitos (fixo) ou 11 dígitos (celular com 9)
- Exemplo: +5511999887766 (celular SP)

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_jkl012mno",
  "end_to_end_id": "E12345678202501241045JKL012MNO",
  "valor": 300.00,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "chave_pix": "+55 11 99988-7766",
    "tipo_chave": "telefone",
    "nome": "Maria Santos"
  }
}
```

#### gRPC Proto Mapping
```protobuf
// Mesmo InitiatePixPaymentByKeyRequest
// com pix_key_type = PIX_KEY_TYPE_PHONE
```

---

### 2.5 Pagamento por EVP (Chave Aleatória)

#### REST API Request (JSON)
```json
{
  "valor": 5000.00,
  "descricao": "Pagamento via chave aleatória",
  "external_id": "TXN-999",
  "destinatario": {
    "chave_pix": "123e4567-e89b-12d3-a456-426614174000",
    "tipo_chave": "evp"
  },
  "pagador": {
    "cpf": "12345678000190",
    "conta_id": "acc_abc123"
  }
}
```

#### Validações
- `chave_pix`: UUID v4
- Regex: `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`
- Exatamente 36 caracteres (com hífens)
- Versão 4 do UUID (4 no terceiro grupo)

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_pqr345stu",
  "end_to_end_id": "E12345678202501241050PQR345STU",
  "external_id": "TXN-999",
  "valor": 5000.00,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "chave_pix": "123e4567-e89b-12d3-a456-426614174000",
    "tipo_chave": "evp",
    "nome": "José Oliveira",
    "cpf": "111.222.333-44"
  }
}
```

#### gRPC Proto Mapping
```protobuf
// Mesmo InitiatePixPaymentByKeyRequest
// com pix_key_type = PIX_KEY_TYPE_EVP
```

---

### 2.6 Pagamento por Conta Bancária

#### REST API Request (JSON)
```json
{
  "valor": 8500.00,
  "descricao": "Pagamento via dados bancários",
  "external_id": "BANK-TRANSFER-123",
  "destinatario": {
    "nome": "Maria Santos",
    "cpf": "98765432100",
    "banco": "001",
    "agencia": "1234",
    "conta": "567890",
    "tipo_conta": "corrente"
  },
  "pagador": {
    "nome": "Empresa XYZ LTDA",
    "cpf": "12345678000190",
    "conta_id": "acc_abc123"
  }
}
```

#### Validações
- `banco`: Código COMPE (3 dígitos) ou ISPB (8 dígitos)
- `agencia`: 4 dígitos
- `conta`: Alfanumérico, max 20 chars
- `tipo_conta`: Enum ["corrente", "poupança", "salario", "pagamento"]
- `cpf` ou `cnpj` do destinatário obrigatório

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_vwx678yza",
  "end_to_end_id": "E12345678202501241055VWX678YZA",
  "external_id": "BANK-TRANSFER-123",
  "valor": 8500.00,
  "status": "EM_PROCESSAMENTO",
  "destinatario": {
    "nome": "Maria Santos",
    "cpf": "987.654.321-00",
    "banco": {
      "codigo": "001",
      "nome": "Banco do Brasil",
      "ispb": "00000000"
    },
    "agencia": "1234",
    "conta": "567890-1",
    "tipo_conta": "corrente"
  },
  "horario": {
    "solicitacao": "2025-01-24T10:55:00Z"
  }
}
```

#### gRPC Proto Mapping
```protobuf
message InitiatePixPaymentByAccountRequest {
  string amount = 1;                    // "8500.00"
  string description = 2;               // "Pagamento via dados bancários"
  string external_id = 3;               // "BANK-TRANSFER-123"
  PayerInfo payer_info = 4;
  PayeeAccount payee_account = 5;
  string callback_url = 6;
  map<string, string> metadata = 7;
  string idempotency_key = 8;
}

message PayeeAccount {
  string ispb = 1;                      // "00000000" or bank code "001"
  string branch = 2;                    // "1234"
  string account = 3;                   // "567890"
  string account_type = 4;              // "corrente"
  string document = 5;                  // "98765432100"
  string name = 6;                      // "Maria Santos"
}
```

---

### 2.7 Pagamento por QR Code (BR Code)

#### REST API Request (JSON)
```json
{
  "brcode": "00020126580014br.gov.bcb.pix0136123e4567-e89b-12d3-a456-426614174000520400005303986540510.005802BR5913Merchant Name6009SAO PAULO62070503***6304ABCD",
  "pagador": {
    "cpf": "12345678900",
    "conta_id": "acc_abc123"
  },
  "external_id": "QR-PAYMENT-456"
}
```

#### Validações
- `brcode`: String EMV válida
- CRC válido (últimos 4 caracteres)
- Formato conforme especificação EMV QR Code
- Verificar expiração (se presente)

#### Success Response (201 Created)
```json
{
  "id": "pix_pay_qr_bcd901efg",
  "end_to_end_id": "E12345678202501241100BCD901EFG",
  "external_id": "QR-PAYMENT-456",
  "valor": 10.00,
  "status": "EM_PROCESSAMENTO",
  "qrcode": {
    "tipo": "ESTATICO",
    "chave_pix": "123e4567-e89b-12d3-a456-426614174000",
    "merchant_name": "Merchant Name",
    "merchant_city": "SAO PAULO"
  },
  "horario": {
    "solicitacao": "2025-01-24T11:00:00Z"
  }
}
```

#### gRPC Proto Mapping
```protobuf
message InitiatePixPaymentByQRCodeRequest {
  string qrcode = 1;                    // BR Code EMV string
  string amount = 2;                    // Optional if QR has fixed amount
  string description = 3;
  string external_id = 4;
  PayerInfo payer_info = 5;
  string callback_url = 6;
  map<string, string> metadata = 7;
  string idempotency_key = 8;
}
```

---

## 3. Erros Comuns - Todos os Tipos

### 3.1 Erros de Validação (400 Bad Request)

| Campo | Erro | Código | Mensagem |
|-------|------|--------|----------|
| `valor` | Negativo ou zero | `invalid_value` | "Valor deve ser maior que zero" |
| `valor` | Excede máximo | `value_too_high` | "Valor máximo permitido: R$ 500.000,00" |
| `chave_pix` | Formato inválido | `invalid_format` | "Formato de chave PIX inválido para tipo {tipo}" |
| `cpf` | Inválido | `invalid_cpf` | "CPF inválido (dígito verificador incorreto)" |
| `cnpj` | Inválido | `invalid_cnpj` | "CNPJ inválido" |
| `conta_id` | Ausente | `missing_field` | "Campo 'conta_id' é obrigatório" |

### 3.2 Erros de Negócio (422 Unprocessable Entity)

| Erro | Código | Descrição |
|------|--------|-----------|
| Chave não encontrada | `invalid_key` | Chave PIX não existe no DICT |
| Saldo insuficiente | `insufficient_balance` | Saldo insuficiente na conta |
| Limite excedido | `limit_exceeded` | Transação excede limite configurado |
| Conta bloqueada | `account_blocked` | Conta com restrição operacional |
| Titularidade inválida | `invalid_ownership` | CPF/CNPJ pagador ≠ titular da chave |

### 3.3 Erros de Autenticação (401 Unauthorized)

| Erro | Código | Descrição |
|------|--------|-----------|
| Token inválido | `authentication_failed` | Token de acesso inválido |
| Token expirado | `token_expired` | Access token expirado |
| Certificado inválido | `certificate_invalid` | mTLS certificate inválido |

---

## 4. Proto Completo para Payment Service

```protobuf
syntax = "proto3";

package payment.pixb2b;

import "google/protobuf/timestamp.proto";

option go_package = "github.com/london-bridge/money-moving/apps/payment/proto/pixb2b";

// =============================================================================
// PAYMENT SERVICE - PIX OUT B2B
// =============================================================================

service PixPaymentB2BService {
  // Payment initiation methods
  rpc InitiatePixPaymentByKey(InitiatePixPaymentByKeyRequest) returns (InitiatePixPaymentResponse);
  rpc InitiatePixPaymentByAccount(InitiatePixPaymentByAccountRequest) returns (InitiatePixPaymentResponse);
  rpc InitiatePixPaymentByQRCode(InitiatePixPaymentByQRCodeRequest) returns (InitiatePixPaymentResponse);

  // QR Code operations
  rpc DecodeQRCode(DecodeQRCodeRequest) returns (DecodeQRCodeResponse);

  // Query operations
  rpc GetPixPayment(GetPixPaymentRequest) returns (PixPaymentResponse);
  rpc GetPixPaymentByE2EId(GetPixPaymentByE2EIdRequest) returns (PixPaymentResponse);
  rpc ListPixPayments(ListPixPaymentsRequest) returns (ListPixPaymentsResponse);

  // Management
  rpc CancelPixPayment(CancelPixPaymentRequest) returns (CancelPixPaymentResponse);
}

// =============================================================================
// PAYMENT BY PIX KEY (CPF, CNPJ, Email, Phone, EVP)
// =============================================================================

message InitiatePixPaymentByKeyRequest {
  string amount = 1;                           // "1500.50" (BRL, 2 decimals)
  string pix_key = 2;                          // PIX key value
  PixKeyType pix_key_type = 3;                 // Key type enum
  string description = 4;                      // Optional, max 140 chars
  string external_id = 5;                      // Client reference ID
  PayerInfo payer_info = 6;                    // Payer details
  string callback_url = 7;                     // Webhook URL
  map<string, string> metadata = 8;            // Custom key-value pairs
  string idempotency_key = 9;                  // UUID for idempotency
}

enum PixKeyType {
  PIX_KEY_TYPE_UNSPECIFIED = 0;
  PIX_KEY_TYPE_CPF = 1;                        // 11 digits
  PIX_KEY_TYPE_CNPJ = 2;                       // 14 digits
  PIX_KEY_TYPE_EMAIL = 3;                      // RFC 5322 email
  PIX_KEY_TYPE_PHONE = 4;                      // +55DDNNNNNNNNN
  PIX_KEY_TYPE_EVP = 5;                        // UUID v4 (random key)
}

message PayerInfo {
  string account_id = 1;                       // LB Pay account ID
  string name = 2;                             // Payer name
  string document = 3;                         // CPF or CNPJ (digits only)
}

// =============================================================================
// PAYMENT BY BANK ACCOUNT
// =============================================================================

message InitiatePixPaymentByAccountRequest {
  string amount = 1;
  string description = 2;
  string external_id = 3;
  PayerInfo payer_info = 4;
  PayeeAccount payee_account = 5;
  string callback_url = 6;
  map<string, string> metadata = 7;
  string idempotency_key = 8;
}

message PayeeAccount {
  string ispb = 1;                             // 8-digit ISPB or 3-digit bank code
  string branch = 2;                           // Branch number (agência)
  string account = 3;                          // Account number
  string account_type = 4;                     // "corrente", "poupança", "salario", "pagamento"
  string document = 5;                         // CPF or CNPJ of account holder
  string name = 6;                             // Account holder name
}

// =============================================================================
// PAYMENT BY QR CODE
// =============================================================================

message InitiatePixPaymentByQRCodeRequest {
  string qrcode = 1;                           // BR Code EMV string
  string amount = 2;                           // Optional (if QR allows variable amount)
  string description = 3;
  string external_id = 4;
  PayerInfo payer_info = 5;
  string callback_url = 6;
  map<string, string> metadata = 7;
  string idempotency_key = 8;
}

// =============================================================================
// DECODE QR CODE
// =============================================================================

message DecodeQRCodeRequest {
  string qrcode = 1;                           // BR Code to decode
}

message DecodeQRCodeResponse {
  string qrcode = 1;
  string format = 2;                           // "emv"
  string type = 3;                             // "ESTATICO", "DINAMICO"
  QRCodeParsedData parsed_data = 4;
  QRCodeValidation validation = 5;
  QRCodePaymentInfo payment_info = 6;
}

message QRCodeParsedData {
  MerchantAccountInfo merchant_account_information = 1;
  string merchant_category_code = 2;
  string transaction_currency = 3;             // "986" (BRL)
  string transaction_amount = 4;
  string country_code = 5;                     // "BR"
  string merchant_name = 6;
  string merchant_city = 7;
  AdditionalData additional_data = 8;
  string crc = 9;
}

message MerchantAccountInfo {
  string gui = 1;                              // "br.gov.bcb.pix"
  string pix_key = 2;
  string url = 3;                              // For dynamic QR codes
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

// =============================================================================
// COMMON RESPONSE
// =============================================================================

message InitiatePixPaymentResponse {
  string payment_id = 1;                       // Internal payment ID
  PaymentStatus status = 2;
  google.protobuf.Timestamp created_at = 3;
  google.protobuf.Timestamp estimated_completion = 4;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_STATUS_PENDING = 1;                  // EM_PROCESSAMENTO
  PAYMENT_STATUS_VALIDATING = 2;
  PAYMENT_STATUS_PROCESSING = 3;
  PAYMENT_STATUS_COMPLETED = 4;                // REALIZADO
  PAYMENT_STATUS_FAILED = 5;                   // NAO_REALIZADO
  PAYMENT_STATUS_CANCELLED = 6;
}

// =============================================================================
// QUERY OPERATIONS
// =============================================================================

message GetPixPaymentRequest {
  string payment_id = 1;
}

message GetPixPaymentByE2EIdRequest {
  string e2e_id = 1;                           // End-to-End ID (32 chars)
}

message PixPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  string amount = 3;
  string payment_method = 4;                   // "pix_key", "account", "qrcode"

  // PIX key payment details
  string pix_key = 5;
  PixKeyType pix_key_type = 6;

  // Account payment details
  PayeeAccount payee_account = 7;

  // QR Code payment details
  string qrcode_parsed = 8;                    // JSON string

  string description = 9;
  string external_id = 10;

  PayerInfo payer_info = 11;
  PayeeInfo payee_info = 12;

  string e2e_id = 13;                          // End-to-End ID
  string txid = 14;                            // Transaction ID

  google.protobuf.Timestamp created_at = 15;
  google.protobuf.Timestamp processed_at = 16;
  google.protobuf.Timestamp completed_at = 17;

  map<string, string> metadata = 18;
  ValidationInfo validation = 19;
  PaymentError error = 20;
}

message PayeeInfo {
  string name = 1;
  string document = 2;                         // CPF or CNPJ
  string bank = 3;                             // Bank name
  string bank_code = 4;                        // COMPE code
  string ispb = 5;                             // ISPB
  string account_type = 6;
}

message ValidationInfo {
  bool same_ownership_required = 1;
  string same_ownership_result = 2;            // "valid", "invalid", "not_checked"
  string same_ownership_source = 3;            // "lbpay_internal", "dict"
  bool pix_key_validated = 4;
}

message PaymentError {
  string code = 1;                             // "invalid_key", "insufficient_balance", etc.
  string message = 2;
  string type = 3;                             // "validation_error", "business_error", etc.
  google.protobuf.Timestamp timestamp = 4;
}

// =============================================================================
// LIST OPERATIONS
// =============================================================================

message ListPixPaymentsRequest {
  string account_id = 1;
  PaymentStatus status_filter = 2;
  google.protobuf.Timestamp start_date = 3;
  google.protobuf.Timestamp end_date = 4;
  string pix_key_filter = 5;
  string e2e_id_filter = 6;
  string cursor = 7;
  int32 limit = 8;                             // Max 100
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

// =============================================================================
// CANCEL OPERATION
// =============================================================================

message CancelPixPaymentRequest {
  string payment_id = 1;
}

message CancelPixPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  google.protobuf.Timestamp cancelled_at = 3;
  string message = 4;
}
```

---

## 5. Tabela de Correspondência REST ↔ gRPC

| REST Endpoint | HTTP Method | gRPC Method | Request Message | Response Message |
|---------------|-------------|-------------|-----------------|------------------|
| `/v1/pix/payments` (key) | POST | `InitiatePixPaymentByKey` | `InitiatePixPaymentByKeyRequest` | `InitiatePixPaymentResponse` |
| `/v1/pix/payments` (account) | POST | `InitiatePixPaymentByAccount` | `InitiatePixPaymentByAccountRequest` | `InitiatePixPaymentResponse` |
| `/v1/pix/qrcodes/pay` | POST | `InitiatePixPaymentByQRCode` | `InitiatePixPaymentByQRCodeRequest` | `InitiatePixPaymentResponse` |
| `/v1/pix/qrcodes/decode` | POST | `DecodeQRCode` | `DecodeQRCodeRequest` | `DecodeQRCodeResponse` |
| `/v1/pix/payments/{id}` | GET | `GetPixPayment` | `GetPixPaymentRequest` | `PixPaymentResponse` |
| `/v1/pix/payments/e2e/{e2e_id}` | GET | `GetPixPaymentByE2EId` | `GetPixPaymentByE2EIdRequest` | `PixPaymentResponse` |
| `/v1/pix/payments` | GET | `ListPixPayments` | `ListPixPaymentsRequest` | `ListPixPaymentsResponse` |
| `/v1/pix/payments/{id}` | DELETE | `CancelPixPayment` | `CancelPixPaymentRequest` | `CancelPixPaymentResponse` |

---

## 6. Validações por Campo (Resumo)

| Campo | Tipo PIX | Validação | Regex/Constraint |
|-------|----------|-----------|------------------|
| `chave_pix` (CPF) | CPF | 11 dígitos + check digit | `^\d{11}$` |
| `chave_pix` (CNPJ) | CNPJ | 14 dígitos + check digit | `^\d{14}$` |
| `chave_pix` (Email) | Email | RFC 5322, max 77 | `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$` |
| `chave_pix` (Phone) | Telefone | +55 DDD #### | `^\+55\d{10,11}$` |
| `chave_pix` (EVP) | EVP | UUID v4 | `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$` |
| `valor` | Todos | >= 0.01, <= 500000.00 | 2 decimals |
| `descricao` | Todos | Max 140 chars | Alfanumérico + espaços |
| `external_id` | Todos | Max 50 chars | `^[a-zA-Z0-9\-_]*$` |

---

## 7. Status Flow

```
PENDING (EM_PROCESSAMENTO)
    ↓
VALIDATING (validando chave PIX, saldo, etc)
    ↓
PROCESSING (enviando para SPI/BACEN)
    ↓
    ├─→ COMPLETED (REALIZADO)
    └─→ FAILED (NAO_REALIZADO)

CANCELLED (usuário cancelou antes do processamento)
```

---

## 8. Próximos Passos para Implementação

1. **Criar diretório proto**:
   ```
   mkdir -p /apps/payment/proto
   ```

2. **Adicionar proto files**:
   - `payment.proto` (este arquivo)
   - `common.proto` (types compartilhados)

3. **Gerar código Go**:
   ```bash
   protoc --go_out=. --go-grpc_out=. proto/*.proto
   ```

4. **Implementar handlers gRPC**:
   - Reutilizar lógica existente de `/apps/payment/app/`
   - Criar camada de adaptação REST ↔ gRPC

5. **Atualizar Seamless PIX**:
   - Substituir chamadas REST por gRPC
   - Implementar mappers DTO → Proto

---

## Referências

- [API_LB.md](./API_LB.md) - Especificação completa REST API
- [BACEN - Manual do PIX](https://www.bcb.gov.br/estabilidadefinanceira/pix)
- [EMV QR Code Specification](https://www.emvco.com/)

---

**Documento criado em:** 2025-12-09
**Status:** Ready for Implementation
