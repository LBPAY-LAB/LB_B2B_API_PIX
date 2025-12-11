# PIX OUT B2B - Especificação de Escopo de Implementação

**Data**: 2025-12-09
**Versão**: 1.0
**Objetivo**: Validar o escopo de implementação dos 6 pontos principais do PIX OUT B2B

---

## 1. RESUMO EXECUTIVO

### 1.1 Escopo Simplificado

A implementação consiste em **estender** a REST API Seamless PIX existente e implementar handlers gRPC no Payment service que chamam a lógica de negócio já existente (DICT, SPI, Ledger v2).

**NÃO vamos construir do zero**. Vamos aproveitar:
- ✅ Cliente gRPC para Payment service (já existe no monorepo)
- ✅ Integração com DICT/SPI (já está funcionando)
- ✅ Integração com Ledger v2 (já está funcionando)
- ✅ Database PostgreSQL (já está funcionando)
- ✅ Padrão arquitetural existente (COB/COBV como referência)

### 1.2 Arquitetura Atual (Descoberta)

```
┌─────────────────┐      HTTP REST       ┌──────────────────┐
│                 │  ───────────────────> │                  │
│  Seamless PIX   │                       │  Payment Service │
│  (REST API)     │  <─────────────────── │  (gRPC Server)   │
│                 │      gRPC Response     │                  │
└─────────────────┘                       └──────────────────┘
        │                                          │
        │                                          │
        v                                          v
  app/pix/pix.go                          app/pix/*.go
  - GenerateQrCode()                      - Business logic
                                          - Calls DICT/SPI
                                          - Calls Ledger v2
```

**Descoberta Importante**: A arquitetura atual do Seamless PIX usa **HTTP REST** para chamar o Payment service (não gRPC diretamente). Exemplo:

```go
// seamless/app/pix/pix.go
body, err := a.request.Http.Request(ctx,
    http.MethodPost,
    a.paymentUrl+"/v1/qrcode",  // HTTP REST call
    headers,
    statusCodes,
    false,
    paymentPayload, nil)
```

**Implicação**: Precisamos decidir se vamos:
- **Opção A**: Manter padrão HTTP REST (Seamless → Payment REST API)
- **Opção B**: Migrar para gRPC (Seamless → Payment gRPC)

---

## 2. OS 6 PONTOS DE ESCOPO

### 2.1 Enviar PIX por Chave PIX
- **Endpoint**: `POST /v1/pix/payments`
- **Método**: Pagamento por chave PIX (CPF, CNPJ, Email, Phone, EVP)
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `SendPixByKey()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.2 Enviar PIX por Dados Bancários
- **Endpoint**: `POST /v1/pix/payments/account`
- **Método**: Pagamento por conta bancária (ISPB + Branch + Account)
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `SendPixByAccount()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.3 Enviar PIX por QR Code
- **Endpoint**: `POST /v1/pix/payments/qrcode`
- **Método**: Pagamento por QR Code (BR Code / PIX Copia e Cola)
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `SendPixByQRCode()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.4 Detalhar QR Code PIX
- **Endpoint**: `POST /v1/pix/qrcodes/decode`
- **Método**: Decodificar QR Code sem iniciar pagamento
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/qrcode_decode.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `DecodeQRCode()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.5 Consultar por ID
- **Endpoint**: `GET /v1/pix/payments/{payment_id}`
- **Método**: Consultar status de pagamento por payment_id
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `GetPayment()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.6 Consultar por End-to-End ID
- **Endpoint**: `GET /v1/pix/payments/e2e/{e2e_id}`
- **Método**: Consultar status de pagamento por E2E ID (BACEN)
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `GetPaymentByE2E()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

### 2.7 Listar com Paginação (EXTRA)
- **Endpoint**: `GET /v1/pix/payments`
- **Método**: Listar pagamentos com filtros e paginação
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Criar handler REST em `seamless/api/v1/pix/payment.go`
  2. Criar método no app layer: `seamless/app/pix/pix.go` → `ListPayments()`
  3. Chamar Payment service (HTTP ou gRPC - **decisão necessária**)
  4. Implementar handler no Payment service (se necessário)

---

## 3. FUNCIONALIDADES ADICIONAIS

### 3.1 Validação de Mesma Titularidade
- **Objetivo**: Validar se a chave PIX pertence ao mesmo titular do pagador (Portaria SPA/ME nº 615/2024)
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Tabela `same_ownership_registry` (✅ **migration já criada**)
  2. Implementar função de validação no Payment service
  3. Consultar registry interno antes de enviar para SPI
  4. Se não encontrado no registry, consultar DICT

### 3.2 Webhooks com Persistência
- **Objetivo**: Notificar cliente sobre mudanças de status com retry logic
- **Status Atual**: ❌ Não existe
- **O que precisa ser feito**:
  1. Tabela `webhook_deliveries` (✅ **migration já criada**)
  2. Implementar worker de retry (exponential backoff)
  3. Persistir todas as tentativas de envio
  4. Dead letter queue para webhooks que falharam 5x

---

## 4. DECISÕES TÉCNICAS NECESSÁRIAS

### 4.1 Comunicação Seamless → Payment

**Opção A: Manter HTTP REST** (padrão atual)
- ✅ Já existe infraestrutura (`request.Http.Request`)
- ✅ Padrão já conhecido pela equipe
- ❌ Menos performático que gRPC
- ❌ Mais verboso (JSON serialization)

**Opção B: Migrar para gRPC**
- ✅ Mais performático (binary protocol)
- ✅ Typed contracts (proto files)
- ✅ Proto file já criado (`pix_payment_b2b.proto`)
- ❌ Precisa criar cliente gRPC no Seamless
- ❌ Mudança de padrão existente

**⚠️ DECISÃO NECESSÁRIA**: Qual padrão usar?

### 4.2 Payment Service: HTTP REST ou gRPC?

**Cenário Atual**: Payment service expõe tanto HTTP REST quanto gRPC (?)

**Opções**:
1. Implementar apenas gRPC handlers (proto já existe)
2. Implementar apenas HTTP REST endpoints
3. Implementar ambos (REST para Seamless, gRPC para outros serviços)

**⚠️ DECISÃO NECESSÁRIA**: Qual padrão usar no Payment service?

### 4.3 Persistência de Dados

**Tabelas criadas** (✅ migrations prontas):
- `pix_payments_b2b` - pagamentos PIX OUT
- `same_ownership_registry` - registry de validação
- `webhook_deliveries` - tracking de webhooks

**Questão**: Onde rodam as migrations?
- Payment service database?
- Seamless database?
- Database dedicado para PIX?

**⚠️ DECISÃO NECESSÁRIA**: Onde criar as tabelas?

---

## 5. ESTRUTURA DE ARQUIVOS A CRIAR/MODIFICAR

### 5.1 Seamless PIX (REST API)

#### Novos arquivos:
```
apps/seamless/
├── api/v1/pix/
│   ├── payment.go               # ✨ CRIAR - Handlers para PIX OUT
│   ├── payment_test.go          # ✨ CRIAR - Testes
│   └── qrcode_decode.go         # ✨ CRIAR - Handler para decode QR
│
├── app/pix/
│   └── pix.go                   # ✏️ MODIFICAR - Adicionar métodos:
│                                #   - SendPixByKey()
│                                #   - SendPixByAccount()
│                                #   - SendPixByQRCode()
│                                #   - DecodeQRCode()
│                                #   - GetPayment()
│                                #   - GetPaymentByE2E()
│                                #   - ListPayments()
│
└── model/pix/
    ├── request/
    │   └── pix_out.go           # ✨ CRIAR - Request models
    └── response/
        └── pix_out.go           # ✨ CRIAR - Response models
```

#### Modificações em arquivos existentes:
```
apps/seamless/api/v1/pix/qrcode.go
- Linha 31: Adicionar novos endpoints no grupo /pix/
  g.Post("payments", api.sendPixByKey)
  g.Post("payments/account", api.sendPixByAccount)
  g.Post("payments/qrcode", api.sendPixByQRCode)
  g.Post("qrcodes/decode", api.decodeQRCode)
  g.Get("payments/:payment_id", api.getPayment)
  g.Get("payments/e2e/:e2e_id", api.getPaymentByE2E)
  g.Get("payments", api.listPayments)
```

### 5.2 Payment Service (gRPC Server)

**Opção A: Se usar gRPC**

```
apps/payment/
├── proto/
│   ├── pix_payment_b2b.proto        # ✅ JÁ EXISTE
│   ├── pix_payment_b2b.pb.go        # ✅ JÁ EXISTE (auto-generated)
│   └── pix_payment_b2b_grpc.pb.go   # ✅ JÁ EXISTE (auto-generated)
│
├── app/pix/
│   ├── send_by_key.go               # ✨ CRIAR - Lógica de negócio
│   ├── send_by_account.go           # ✨ CRIAR - Lógica de negócio
│   ├── send_by_qrcode.go            # ✨ CRIAR - Lógica de negócio
│   ├── decode_qrcode.go             # ✨ CRIAR - Lógica de negócio
│   ├── get_payment.go               # ✨ CRIAR - Lógica de negócio
│   └── list_payments.go             # ✨ CRIAR - Lógica de negócio
│
├── handler/grpc/
│   └── pix_payment_b2b.go           # ✨ CRIAR - gRPC handler
│
└── server/
    └── server.go                    # ✏️ MODIFICAR - Registrar gRPC service
```

**Opção B: Se usar HTTP REST**

```
apps/payment/
├── api/v1/pix/
│   └── payment.go                   # ✨ CRIAR - REST handlers
│
├── app/pix/
│   └── (mesmos arquivos da Opção A)
│
└── server/
    └── server.go                    # ✏️ MODIFICAR - Registrar rotas REST
```

### 5.3 Migrations (Database)

```
apps/payment/migrations/
├── 001_create_pix_payments_b2b.sql          # ✅ JÁ EXISTE
├── 002_create_same_ownership_registry.sql   # ✅ JÁ EXISTE
└── 003_create_webhook_deliveries.sql        # ✅ JÁ EXISTE
```

**Ação necessária**: Rodar migrations no ambiente de desenvolvimento/staging

---

## 6. ANÁLISE DE INFRAESTRUTURA EXISTENTE

### 6.1 O que JÁ existe e funciona

| Componente | Status | Localização | Observação |
|------------|--------|-------------|------------|
| Database PostgreSQL | ✅ Funcionando | - | Conexão já configurada |
| DICT Integration | ✅ Funcionando | `apps/payment/business_partners/pix` | Consulta de chaves PIX |
| SPI Integration | ✅ Funcionando | `apps/payment/business_partners/pix` | Envio de pagamentos |
| Ledger v2 gRPC | ✅ Funcionando | - | Debitar/creditar contas |
| HTTP Request Client | ✅ Existe | `apps/seamless/request` | Para chamadas HTTP |
| Fiber Router | ✅ Existe | `apps/seamless/api` | Framework REST |
| Validation | ✅ Existe | `github.com/go-playground/validator` | Validação de requests |
| Auth Middleware | ✅ Existe | `apps/seamless/api/api.go` | Kong authentication |
| Permission Check | ✅ Existe | `apps/seamless/api/api.go` | Service permissions |

### 6.2 O que precisa ser criado

| Componente | Status | Esforço | Prioridade |
|------------|--------|---------|------------|
| Seamless PIX OUT handlers | ❌ Não existe | Médio | 🔴 Alta |
| Payment PIX OUT handlers | ❌ Não existe | Alto | 🔴 Alta |
| Same Ownership Validation | ❌ Não existe | Médio | 🟡 Média |
| Webhook Worker | ❌ Não existe | Alto | 🟡 Média |
| Request/Response Models | ❌ Não existe | Baixo | 🔴 Alta |
| Unit Tests | ❌ Não existe | Alto | 🟡 Média |
| Integration Tests | ❌ Não existe | Alto | 🟢 Baixa |

---

## 7. PADRÃO DE REFERÊNCIA: COB/COBV

O padrão de implementação deve seguir o que já existe para COB (Cobrança Imediata):

### 7.1 Seamless PIX - Pattern atual

```go
// apps/seamless/api/v1/pix/qrcode.go
func (p *apiImpl) qrCode(c *fiber.Ctx) error {
    // 1. Parse request
    request := qrCodeRequest.QrCode{}
    if err := c.BodyParser(&request); err != nil {
        return c.Status(http.StatusBadRequest).JSON(error)
    }

    // 2. Validate
    err := p.validator.Struct(request)
    if err != nil {
        return c.Status(http.StatusBadRequest).JSON(error)
    }

    // 3. Call app layer
    data, err := p.apps.Pix.GenerateQrCode(c.Context(), request, accountNumber)
    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(error)
    }

    // 4. Return response
    return c.Status(http.StatusOK).JSON(response)
}
```

### 7.2 App Layer - Pattern atual

```go
// apps/seamless/app/pix/pix.go
func (a *appImpl) GenerateQrCode(ctx context.Context, request pixRequest.QrCode, accountNumber string) (*paymentModel.PaymentResponse, error) {
    // Call Payment service via HTTP REST
    body, err := a.request.Http.Request(ctx,
        http.MethodPost,
        a.paymentUrl+"/v1/qrcode",  // REST endpoint
        headers,
        statusCodes,
        false,
        payload, nil)

    // Parse response
    var response paymentModel.PaymentResponse
    err = json.Unmarshal(body, &response)

    return &response, nil
}
```

**Este padrão deve ser replicado para os 6 endpoints do PIX OUT.**

---

## 8. FLUXO DE DADOS PROPOSTO

### 8.1 Fluxo: Enviar PIX por Chave PIX

```
1. Cliente B2B
   ↓
   POST /v1/pix/payments
   {
     "amount": "1500.50",
     "pix_key": "12345678901",
     "pix_key_type": "cpf",
     "description": "Pagamento fornecedor",
     "validate_same_ownership": true,
     "callback_url": "https://client.com/webhooks/pix"
   }
   ↓
2. Seamless PIX (REST API)
   - api/v1/pix/payment.go → sendPixByKey()
   - Valida request (validator)
   - Extrai account_id do header (Kong)
   ↓
3. Seamless App Layer
   - app/pix/pix.go → SendPixByKey()
   - Prepara payload para Payment service
   ↓
4. Payment Service (HTTP REST ou gRPC)
   - Recebe request
   - Valida PIX key no DICT
   - Se validate_same_ownership=true:
     → Consulta same_ownership_registry
     → Se não encontrado, consulta DICT
   - Debita valor no Ledger v2
   - Envia pagamento para SPI
   - Persiste em pix_payments_b2b
   - Cria webhook_delivery (status=pending)
   - Retorna payment_id
   ↓
5. Response para Cliente
   {
     "payment_id": "pix_abc123xyz",
     "status": "processing",
     "created_at": "2025-12-09T10:00:00Z",
     "estimated_completion": "2025-12-09T10:00:10Z"
   }
   ↓
6. Webhook Worker (background)
   - Busca webhooks pendentes (next_retry_at <= NOW())
   - Envia POST para callback_url do cliente
   - Registra resultado em webhook_deliveries
   - Se falhou: agenda retry (exponential backoff)
   - Se atingiu 5 tentativas: move para dead_letter
```

### 8.2 Fluxo: Consultar por ID

```
1. Cliente B2B
   ↓
   GET /v1/pix/payments/pix_abc123xyz
   ↓
2. Seamless PIX (REST API)
   - api/v1/pix/payment.go → getPayment()
   ↓
3. Seamless App Layer
   - app/pix/pix.go → GetPayment()
   ↓
4. Payment Service
   - Consulta pix_payments_b2b WHERE payment_id = 'pix_abc123xyz'
   - Retorna dados completos
   ↓
5. Response para Cliente
   {
     "payment_id": "pix_abc123xyz",
     "status": "completed",
     "amount": "1500.50",
     "pix_key": "12345678901",
     "payee_name": "João Silva",
     "e2e_id": "E1234567820251209100012345678",
     "created_at": "2025-12-09T10:00:00Z",
     "completed_at": "2025-12-09T10:00:08Z"
   }
```

---

## 9. TRATAMENTO DE ERROS

### 9.1 Erros de Validação (400 Bad Request)
- PIX key inválida (formato incorreto)
- Amount inválido (negativo ou zero)
- Campos obrigatórios faltando
- Idempotency key duplicada (pagamento já existe)

### 9.2 Erros de Negócio (422 Unprocessable Entity)
- PIX key não encontrada no DICT
- Same ownership inválido (chave não pertence ao pagador)
- Saldo insuficiente no Ledger
- Conta bloqueada ou inativa

### 9.3 Erros de Integração (503 Service Unavailable)
- DICT indisponível
- SPI indisponível
- Ledger v2 indisponível
- Timeout nas integrações

### 9.4 Estrutura de Erro Padrão

```json
{
  "error": {
    "code": "invalid_pix_key",
    "message": "PIX key not found in DICT",
    "type": "validation_error",
    "timestamp": "2025-12-09T10:00:00Z"
  }
}
```

---

## 10. REQUISITOS NÃO FUNCIONAIS

### 10.1 Performance
- Tempo de resposta < 2 segundos (95 percentil)
- Throughput: 100 requests/segundo
- Database connection pooling configurado

### 10.2 Disponibilidade
- SLA: 99.9% (excluindo janelas de manutenção)
- Graceful degradation se DICT/SPI indisponíveis
- Circuit breaker para integrações externas

### 10.3 Segurança
- Autenticação via Kong (X-Consumer-Username)
- Validação de permissões (checkServicePermission)
- Logs de todas as transações
- Dados sensíveis não logados (PII masking)

### 10.4 Observabilidade
- Logs estruturados (JSON format com logrus)
- Métricas de latência e throughput
- Tracing distribuído (opcional)

---

## 11. PRÓXIMOS PASSOS

### 11.1 Decisões Pendentes (BLOQUEADORES)

1. **Protocolo de Comunicação Seamless → Payment**
   - [ ] HTTP REST (manter padrão atual)
   - [ ] gRPC (migrar para novo padrão)

2. **Payment Service Interface**
   - [ ] Apenas gRPC
   - [ ] Apenas HTTP REST
   - [ ] Ambos (dual interface)

3. **Database para Migrations**
   - [ ] Payment service database
   - [ ] Seamless database
   - [ ] Database dedicado

### 11.2 Implementação (após decisões)

**Sprint 1: Foundation**
- [ ] Rodar migrations (criar tabelas)
- [ ] Criar models (request/response)
- [ ] Setup gRPC client (se usar gRPC)

**Sprint 2: Core Endpoints**
- [ ] Implementar SendPixByKey (endpoint 2.1)
- [ ] Implementar SendPixByAccount (endpoint 2.2)
- [ ] Implementar SendPixByQRCode (endpoint 2.3)

**Sprint 3: Query Endpoints**
- [ ] Implementar GetPayment (endpoint 2.5)
- [ ] Implementar GetPaymentByE2E (endpoint 2.6)
- [ ] Implementar ListPayments (endpoint 2.7)

**Sprint 4: Utilities**
- [ ] Implementar DecodeQRCode (endpoint 2.4)
- [ ] Implementar Same Ownership Validation

**Sprint 5: Webhooks**
- [ ] Implementar webhook worker
- [ ] Implementar retry logic
- [ ] Implementar dead letter queue

**Sprint 6: Testing & Documentation**
- [ ] Unit tests
- [ ] Integration tests
- [ ] API documentation (Swagger)

---

## 12. RISCOS E MITIGAÇÕES

| Risco | Impacto | Probabilidade | Mitigação |
|-------|---------|---------------|-----------|
| DICT/SPI instabilidade | Alto | Média | Circuit breaker, retry logic |
| Mudança de protocolo (HTTP→gRPC) | Médio | Baixa | POC antes de decisão final |
| Performance de database | Médio | Baixa | Indexes já criados nas migrations |
| Webhook delivery failures | Médio | Alta | Retry com exponential backoff |
| Same ownership validation lenta | Baixo | Média | Cache interno do registry |

---

## 13. VALIDAÇÃO DO ESCOPO

### 13.1 Checklist de Completude

**Requisitos Funcionais**:
- ✅ Enviar PIX por Chave PIX
- ✅ Enviar PIX por Dados Bancários
- ✅ Enviar PIX por QR Code
- ✅ Detalhar QR Code PIX
- ✅ Consultar por ID
- ✅ Consultar por End-to-End ID
- ✅ Listar com Paginação
- ✅ Validação de Mesma Titularidade
- ✅ Webhooks com Persistência

**Infraestrutura**:
- ✅ Database migrations criadas
- ✅ Proto file definido
- ✅ Arquitetura existente mapeada
- ✅ Padrões de código identificados

**Pendente**:
- ⚠️ Decisão: Protocolo de comunicação
- ⚠️ Decisão: Interface do Payment service
- ⚠️ Decisão: Database para migrations
- ⚠️ Implementação: Todos os endpoints
- ⚠️ Implementação: Webhook worker
- ⚠️ Implementação: Same ownership validation
- ⚠️ Testes: Unit + Integration

---

## 14. CONCLUSÃO

Este documento valida que o escopo de implementação está **bem definido** e **tecnicamente viável**. A implementação consiste em:

1. **Estender** a REST API Seamless PIX (7 novos endpoints)
2. **Implementar** handlers no Payment service (HTTP REST ou gRPC)
3. **Reutilizar** infraestrutura existente (DICT, SPI, Ledger v2, Database)
4. **Seguir** o padrão arquitetural existente (COB/COBV como referência)

**Estimativa de esforço**: 4-6 semanas (2 desenvolvedores)

**Bloqueadores**: 3 decisões técnicas pendentes (seção 11.1)

**Próxima ação**: Validar decisões técnicas com a equipe e iniciar Sprint 1.
