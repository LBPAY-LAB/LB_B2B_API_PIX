# Dúvidas de Implementação - PIX OUT B2B API

> **Versão:** 1.0
> **Data:** 2025-12-09
> **Escopo:** Extensão da Seamless PIX REST API + Handlers gRPC no Payment Service
> **Complexidade:** Simples - Seguir padrão existente (COB/COBV)

---

Respostas são iniciadas são iniciados pelo LB: ou <LB> </LB>

## 📋 Contexto da Implementação

A implementação do PIX OUT B2B consiste em:

1. **Seamless PIX (REST API)**
   - Criar novos endpoints REST (seguindo padrão COB existente) na Seamless PIX
         <LB>
             sim, teremos que estender a restAPi existinte para podermos executar os 6 topicos que foram solcitiado.
               
             Enviar PIX por Chave PIX - POST /v1/pix/payments  

             Enviar PIX por Dados Bancários - POST /v1/pix/payments/account (ISPB + Conta)

             Enviar PIX por QR Code - POST /v1/pix/payments/qrcode (PIX Copia e Cola)
             
             Detalhar QR Code PIX - POST /v1/pix/qrcodes/decode (retorna JSON) 
             
             Consultar por ID da Transação - GET /v1/pix/payments/{payment_id}
               
             Consultar por End-to-End ID - GET /v1/pix/payments/e2e/{e2e_id}
               
             Listar com Paginação e Filtros - GET /v1/pix/payments (8 filtros)
               
             Cancelar Pagamento - DELETE /v1/pix/payments/{payment_id}
               
             Validação de Mesma Titularidade - Tabela interna same_ownership_registry + validação automática
               
             Considere como referência o Webhooh da Cobrança imediata.

   - Criar handlers HTTP
         Sim
   - Criar use cases
            Sim
   - Criar cliente gRPC para chamar Payment service
            Não. Já existente no monorepo orchestration-go: PIX e PIX-OUT
2. **Payment Service (gRPC)**
   - Implementar handlers gRPC
            estender o que necessário para atender os novos endpoints na na rest API
   - Implementar lógica de negócio
            em principio o grpc tem a logica de negócio que precisamos
   - Integrar com DICT/SPI/Ledger v2
            isso já está funcionando.
   - Persistir em banco de dados
            isso já está funcionando. apenas precisamos de avaliar o que o PIX Out precisa de persisitir na sua execução.

**Padrão de Referência:** Implementação de COB/COBV já existente no Seamless PIX

---

## 1. Dúvidas sobre Seamless PIX (REST API)

### 1.1 Estrutura de Pastas ✅ DECIDIDO

**Questão:** Seguir o mesmo padrão de `internal/charge` ou criar novo `internal/pixout`?

**Padrão Atual (Charge):**
```
internal/charge/
├── application/
│   ├── usecases/
│   └── validation/
├── domain/
│   ├── dtos/
│   ├── entities/
│   └── usecases/
└── infrastructure/
    └── web/
        ├── grpc/
        │   ├── adapters/
        │   ├── clients/
        │   └── mappers/
        └── http/
            └── handlers/
```

**Opções:**
- [ ] **Opção A:** Criar `internal/pixout` com mesma estrutura
- [ ] **Opção B:** Criar `internal/pix/payment` (dentro do módulo pix existente)
- [ ] **Opção C:** Aproveitar `internal/pix` existente e adicionar sub-módulos

**Decisão Recomendada:** Opção A - `internal/pixout` (isolamento e clareza)

---

### 1.2 Endpoints REST

**Questão:** Confirmar endpoints finais

**Proposta:**
```
POST   /v1/pix/payments              # Criar PIX (chave)
POST   /v1/pix/payments/account      # Criar PIX (conta bancária)
POST   /v1/pix/payments/qrcode       # Criar PIX (QR Code)
POST   /v1/pix/qrcodes/decode        # Decodificar QR Code
GET    /v1/pix/payments/:id          # Consultar por payment_id
GET    /v1/pix/payments/e2e/:e2e_id  # Consultar por E2E ID
GET    /v1/pix/payments              # Listar com filtros
DELETE /v1/pix/payments/:id          # Cancelar
GET    /v1/pix/payments/:id/webhooks # Listar webhooks
POST   /v1/pix/payments/:id/webhooks/retry # Retry webhook
```

**Dúvidas:**
- Aprovar essa estrutura ou ajustar?
- Prefixo `/v1/pix/payments` ou `/v1/pix/out`?

**Decisão Necessária:** [ ] Aprovar endpoints

---

### 1.3 Cliente gRPC Payment

**Questão:** Como criar e injetar cliente gRPC do Payment service?

**Padrão Atual (Charge → Orchestration):**
```go
// internal/charge/infrastructure/web/grpc/clients/orchestration_charge_service_client.go
type OrchestrationChargeServiceClient interface {
    CreateImmediateCharge(ctx, accountNumber, txid, req) (*CobGerada, error)
    // ...
}
```

**Para PIX OUT (Seamless PIX → Payment):**
```go
// internal/pixout/infrastructure/web/grpc/clients/payment_pix_service_client.go
type PaymentPixServiceClient interface {
    InitiatePixPaymentByKey(ctx, req *pixb2bpb.InitiatePixPaymentByKeyRequest) (*pixb2bpb.InitiatePixPaymentResponse, error)
    InitiatePixPaymentByAccount(ctx, req) (*Response, error)
    GetPixPayment(ctx, req) (*PixPaymentResponse, error)
    // ... outros 8 métodos
}
```

**Dúvidas:**
- Connection string do Payment gRPC: `localhost:50051`? Variável de ambiente?
- Usar connection pooling? Lazy connection?
- Timeout padrão: 30s?

**Decisão Necessária:** [ ] Endpoint Payment gRPC e configuração

---

### 1.4 Dependency Injection

**Questão:** Como injetar no `DependencyContainer`?

**Padrão Atual:**
```go
// internal/bootstrap/dependency_container.go
type DependencyContainer struct {
    ChargeServiceAdapter *adapters.ChargeServiceAdapter
    CreateChargeHandler  *handlers.CreateChargeHandler
    // ...
}
```

**Adicionar:**
```go
type DependencyContainer struct {
    // ... existentes

    // PIX OUT B2B
    PixOutServiceClient   *pixoutClients.PaymentPixServiceClient
    PixOutServiceAdapter  *pixoutAdapters.PixOutServiceAdapter
    CreatePixOutHandler   *pixoutHandlers.CreatePixPaymentHandler
    GetPixOutHandler      *pixoutHandlers.GetPixPaymentHandler
    ListPixOutHandler     *pixoutHandlers.ListPixPaymentHandler
    CancelPixOutHandler   *pixoutHandlers.CancelPixPaymentHandler
    DecodeQRCodeHandler   *pixoutHandlers.DecodeQRCodeHandler
}
```

**Decisão Necessária:** [ ] Aprovar estrutura de DI

---

## 2. Dúvidas sobre Payment Service (gRPC)

### 2.1 Estrutura de Implementação

**Questão:** Onde implementar os handlers gRPC?

**Proposta:**
```
apps/payment/
├── proto/
│   ├── pix_payment_b2b.proto          ✅ Já existe
│   ├── pix_payment_b2b.pb.go          ✅ Gerado
│   └── pix_payment_b2b_grpc.pb.go     ✅ Gerado
├── internal/
│   └── grpc/
│       └── handlers/
│           └── pix_payment_b2b_handler.go  ⏳ CRIAR
├── pkg/
│   ├── repository/
│   │   ├── pix_payment_repository.go       ⏳ CRIAR
│   │   ├── same_ownership_repository.go    ⏳ CRIAR
│   │   └── webhook_delivery_repository.go  ⏳ CRIAR
│   ├── service/
│   │   ├── pix_payment_service.go          ⏳ CRIAR
│   │   ├── dict_client.go                  ⏳ CRIAR
│   │   └── spi_client.go                   ⏳ CRIAR
│   └── worker/
│       ├── payment_processor.go            ⏳ CRIAR
│       └── webhook_delivery.go             ⏳ CRIAR
└── migrations/
    ├── 001_create_pix_payments_b2b.sql         ✅ Já existe
    ├── 002_create_same_ownership_registry.sql  ✅ Já existe
    └── 003_create_webhook_deliveries.sql       ✅ Já existe
```

**Decisão Necessária:** [ ] Aprovar estrutura de pastas

---

### 2.2 Registro do Serviço gRPC

**Questão:** Como registrar `PixPaymentB2BService` no servidor gRPC?

**Onde está o servidor gRPC do Payment?**
- Arquivo: `apps/payment/???` (preciso encontrar o main.go ou server setup)

**Código esperado:**
```go
// apps/payment/cmd/server/main.go (ou equivalente)
import (
    pixb2bpb "github.com/london-bridge/money-moving/apps/payment/proto"
    "github.com/london-bridge/money-moving/apps/payment/internal/grpc/handlers"
)

func main() {
    // ... setup gRPC server

    // Registrar serviço PIX OUT B2B
    pixB2BHandler := handlers.NewPixPaymentB2BHandler(
        pixPaymentRepo,
        dictClient,
        spiClient,
        ledgerClient,
    )
    pixb2bpb.RegisterPixPaymentB2BServiceServer(grpcServer, pixB2BHandler)

    // ... start server
}
```

**Dúvidas:**
- Onde está o setup do servidor gRPC do Payment atualmente?
- Já existe registro de outros serviços gRPC?

**Decisão Necessária:** [ ] Identificar onde registrar serviço

---

### 2.3 Database Connection

**Questão:** Como obter conexão com o banco de dados?

**Opções:**
- [ ] Payment service já tem connection pool configurado?
- [ ] Usar mesma conexão que outras features do Payment?
- [ ] Criar nova conexão específica?

**Connection String:**
```
postgresql://user:pass@localhost:5432/lbpay_payment?sslmode=disable
```

**Dúvidas:**
- Variável de ambiente: `DATABASE_URL`? Outro nome?
- Aplicar migrations: manual ou automático no startup?

**Decisão Necessária:** [ ] Configuração de database

---

## 3. Integrações Externas

### 3.1 DICT (Validação de Chaves PIX) 🔴 CRÍTICO

**Questão:** Como validar chaves PIX via DICT?

**Opções:**
1. **DICT Real (BACEN)**
   - Endpoint: `https://dict.bcb.gov.br/api/v1/...` ?
   - Credenciais: API Key? Certificado?
   - Documentação disponível?

2. **Mock/Simulator** (desenvolvimento)
   - Criar mock interno?
   - Usar Wiremock?

3. **Tabela Interna** (mesma titularidade)
   - Usar apenas `same_ownership_registry` (já criada)?
   - Popular via CSV inicial?

**Decisão Necessária:**
- [ ] Ambiente DEV: Mock ou tabela interna?
- [ ] Ambiente PROD: Credenciais DICT

**Impacto:** 🔴 **Bloqueante** para validação de chaves

---

### 3.2 SPI (Envio de PIX) 🔴 CRÍTICO

**Questão:** Como enviar PIX ao BACEN via SPI?

**Opções:**
1. **SPI Real**
   - Endpoint: `https://spi.bcb.gov.br/api/...` ?
   - Autenticação: mTLS? OAuth?

2. **Mock** (desenvolvimento)
   - Simular sucesso/falha?
   - Gerar E2E ID fake?

**Formato do E2E ID:**
```
E{ISPB}{YYYYMMDD}{HHMMss}{uniqueId}
E1234567820251209174530ABC123XYZ
```

**Dúvidas:**
- ISPB do LBPay: qual valor usar?
- Gerar `uniqueId`: UUID? Sequencial?

**Decisão Necessária:**
- [ ] Ambiente DEV: Mock?
- [ ] Ambiente PROD: Endpoint e credenciais SPI

**Impacto:** 🔴 **Bloqueante** para efetivar pagamentos

---

### 3.3 Ledger v2 (Reserva de Saldo)

**Questão:** Como integrar com Ledger v2?

**Fluxo esperado:**
```
1. Reservar saldo: ledger.ReserveBalance(amount, account_id)
2. Se SPI aprovar: ledger.ConfirmTransaction(tx_id)
3. Se SPI rejeitar: ledger.RejectTransaction(tx_id)
```

**Dúvidas:**
- Ledger v2 gRPC endpoint: `localhost:50052`? Variável de ambiente?
- Métodos disponíveis:
  - `ReserveBalance(...)` ?
  - `ConfirmTransaction(...)` ?
  - `RejectTransaction(...)` ?
- Proto file do Ledger v2: onde está?

**Decisão Necessária:** [ ] Endpoint e proto do Ledger v2

**Impacto:** 🟡 **Alta** - Necessário para controle de saldo

---

### 3.4 QR Code Parser

**Questão:** Como parsear BR Code (QR Code PIX)?

**Opções:**
1. **Biblioteca Go**
   - Existe lib recomendada? `github.com/???/brcode` ?
   - Implementar parser interno?

2. **Campos EMV a extrair:**
   ```
   Tag 26: Merchant Account Information (PIX key)
   Tag 52: Merchant Category Code
   Tag 53: Transaction Currency (986 = BRL)
   Tag 54: Transaction Amount
   Tag 58: Country Code (BR)
   Tag 59: Merchant Name
   Tag 60: Merchant City
   Tag 62: Additional Data (txid)
   Tag 63: CRC16
   ```

**Decisão Necessária:** [ ] Biblioteca ou implementação interna?

**Impacto:** 🟢 **Média** - Feature específica

---

## 4. Processamento Assíncrono

### 4.1 Message Queue

**Questão:** PIX OUT precisa de processamento assíncrono?

**Fluxo Síncrono (Simples):**
```
REST Request → gRPC Call → Process → Response
```

**Fluxo Assíncrono (Complexo):**
```
REST Request → gRPC Call → Publish to Queue → Worker Process → Webhook
                         ↓
                    Response 202 Accepted
```

**Pergunta:**
- Implementar versão **síncrona** primeiro (mais simples)?
- Ou já implementar com **workers assíncronos**?

**Decisão Necessária:** [ ] Síncrono ou assíncrono?

**Impacto:** 🟡 **Afeta arquitetura**

---

### 4.2 Webhooks

**Questão:** Implementar delivery de webhooks quando?

**Opções:**
1. **Sprint 1:** Apenas persistir na tabela `webhook_deliveries` (sem enviar)
2. **Sprint 2:** Implementar worker de delivery
3. **Sprint 3:** Implementar retry e dead letter queue

**Decisão Necessária:** [ ] Priorizar webhooks ou deixar para depois?

**Impacto:** 🟢 **Baixo** - Feature pode ser incremental

---

## 5. Validações e Regras de Negócio

### 5.1 Mesma Titularidade

**Questão:** Validação de mesma titularidade é **obrigatória** ou **opcional**?

**Cenários:**
1. **Sempre obrigatória:** Todos os PIX OUT B2B devem validar
2. **Opcional:** Cliente decide via flag `validate_same_ownership: true/false`
3. **Por plano:** Apenas clientes Premium validam

**Fluxo de Validação:**
```
1. Consultar tabela `same_ownership_registry`
2. Se encontrado: OK, prosseguir
3. Se não encontrado:
   a) Consultar DICT (se disponível)
   b) Se DICT confirmar: adicionar na tabela + prosseguir
   c) Se DICT negar: rejeitar pagamento
```

**Decisão Necessária:** [ ] Obrigatória ou opcional?

**Impacto:** 🔴 **Compliance** (Portaria SPA/ME nº 615/2024)

---

### 5.2 Limites de Valores

**Questão:** Definir limites por transação

**Proposta:**
```
Valor mínimo: R$ 0,01
Valor máximo: R$ 500.000,00 (por transação)
Limite diário: R$ 5.000.000,00 (por conta)
```

**Dúvidas:**
- Limites são iguais para todos os clientes B2B?
- Ou configurável por plano/conta?
- Validar no Seamless PIX ou no Payment service?

**Decisão Necessária:** [ ] Matriz de limites

**Impacto:** 🟡 **Média** - Risco e compliance

---

## 6. Testes

### 6.1 Dados de Teste

**Questão:** Como testar sem integração real com BACEN?

**Precisamos:**
1. **Chaves PIX de teste**
   ```
   CPF: 12345678901 (válido, pertence a João Silva)
   Email: teste@lbpay.com.br
   Phone: +5511999887766
   EVP: 123e4567-e89b-12d3-a456-426614174000
   ```

2. **Contas de teste**
   ```
   account_id: acc_test_001
   Saldo disponível: R$ 100.000,00
   ```

3. **QR Codes de teste**
   ```
   QR Code estático: 00020126...63041D3D
   QR Code dinâmico: 00020126...
   ```

**Decisão Necessária:** [ ] Criar seed de dados de teste?

**Impacto:** 🟢 **Baixo** - Facilita desenvolvimento

---

### 6.2 Testes Unitários

**Questão:** Seguir padrão de testes existente?

**Padrão Atual (Charge):**
```
internal/charge/application/usecases/create_charge_usecase_test.go
internal/charge/infrastructure/web/http/handlers/create_charge_handler_test.go
```

**Aplicar para PIX OUT:**
```
internal/pixout/application/usecases/create_payment_usecase_test.go
internal/pixout/infrastructure/web/http/handlers/create_payment_handler_test.go
```

**Decisão Necessária:** [ ] Seguir mesmo padrão de testes

**Impacto:** 🟢 **Baixo** - Qualidade

---

## 7. Deployment e Ambiente

### 7.1 Variáveis de Ambiente

**Questão:** Quais variáveis de ambiente precisam ser configuradas?

**Proposta:**
```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/lbpay_payment

# gRPC Services
PAYMENT_GRPC_ADDR=localhost:50051
LEDGER_V2_GRPC_ADDR=localhost:50052

# External APIs
DICT_API_URL=https://dict.bcb.gov.br/api/v1
DICT_API_KEY=xxx
SPI_API_URL=https://spi.bcb.gov.br/api/v1
SPI_MTLS_CERT=/path/to/cert.pem
SPI_MTLS_KEY=/path/to/key.pem

# Business
LBPAY_ISPB=12345678
PIX_MIN_VALUE=0.01
PIX_MAX_VALUE=500000.00

# Features
ENABLE_SAME_OWNERSHIP_VALIDATION=true
ENABLE_WEBHOOKS=true
```

**Decisão Necessária:** [ ] Aprovar lista de variáveis

**Impacto:** 🟡 **Média** - DevOps

---

### 7.2 Migrations

**Questão:** Como aplicar migrations do Payment service?

**Opções:**
1. **Manual:** DBA aplica via psql
2. **Automático:** Service aplica no startup (golang-migrate)
3. **CI/CD:** Pipeline aplica antes do deploy

**Migrations criadas:**
```
001_create_pix_payments_b2b.sql
002_create_same_ownership_registry.sql
003_create_webhook_deliveries.sql
```

**Decisão Necessária:** [ ] Estratégia de migrations

**Impacto:** 🟡 **Média** - DevOps

---

## 8. Cronograma e Prioridades

### 8.1 MVP (Mínimo Viável)

**O que implementar primeiro?**

**Sprint 1 (MVP):**
- [ ] Seamless PIX: Endpoint `POST /v1/pix/payments` (PIX por chave)
- [ ] Seamless PIX: Endpoint `GET /v1/pix/payments/:id`
- [ ] Payment gRPC: Handler `InitiatePixPaymentByKey`
- [ ] Payment gRPC: Handler `GetPixPayment`
- [ ] Database: Aplicar migration 001
- [ ] Validações: CPF/CNPJ básicas (sem DICT)
- [ ] Mock: DICT e SPI (respostas fake)

**Sprint 2 (Expansão):**
- [ ] Endpoint: Payment by Account
- [ ] Endpoint: Payment by QR Code
- [ ] Endpoint: List Payments
- [ ] Validação: DICT real (se credenciais disponíveis)
- [ ] Database: Migrations 002 e 003

**Sprint 3 (Avançado):**
- [ ] Webhooks: Delivery worker
- [ ] Same Ownership: Tabela interna funcional
- [ ] QR Code: Parser e decode
- [ ] Testes: Cobertura > 80%

**Decisão Necessária:** [ ] Aprovar roadmap de sprints

**Impacto:** 🟡 **Alta** - Planejamento

---

## 9. Resumo: Decisões Críticas (P1)

| # | Questão | Responsável | Prazo | Bloqueante? |
|---|---------|-------------|-------|-------------|
| 1 | Endpoint Payment gRPC | Infra/Config | 12/12 | ✅ Sim |
| 2 | Endpoint Ledger v2 gRPC | Infra/Config | 12/12 | ✅ Sim |
| 3 | Database connection string | DevOps | 12/12 | ✅ Sim |
| 4 | DICT: Mock ou Real? | Produto | 12/12 | ✅ Sim |
| 5 | SPI: Mock ou Real? | Produto | 12/12 | ✅ Sim |
| 6 | Mesma titularidade: Obrigatória? | Compliance | 12/13 | ⚠️ Importante |
| 7 | Limites de valores | Produto | 12/13 | ⚠️ Importante |
| 8 | Processamento: Síncrono ou Assíncrono? | Arquitetura | 12/15 | ⏳ Pode decidir depois |
| 9 | Webhooks: Sprint 1 ou 2? | Produto | 12/15 | ⏳ Pode decidir depois |

---

## 10. Próximos Passos

### Ações Imediatas (Esta Semana)

1. **Configuração:**
   - [ ] Obter endpoint do Payment gRPC service
   - [ ] Obter endpoint do Ledger v2 gRPC service
   - [ ] Obter database connection string

2. **Decisões de Produto:**
   - [ ] Definir se DICT/SPI serão mock (dev) ou real
   - [ ] Definir se mesma titularidade é obrigatória
   - [ ] Aprovar endpoints REST

3. **Setup de Desenvolvimento:**
   - [ ] Clonar padrão de `internal/charge` para `internal/pixout`
   - [ ] Criar handler básico no Payment gRPC
   - [ ] Testar conexão gRPC: Seamless PIX → Payment

4. **Documentação:**
   - [ ] Revisar este documento
   - [ ] Adicionar respostas conforme decisões tomadas

---

## 11. Contatos

| Questão | Responsável | Como Contactar |
|---------|-------------|----------------|
| Endpoints gRPC | DevOps/Infra | Slack: #devops |
| DICT/SPI Mock vs Real | Produto | Email: produto@lbpay.com.br |
| Compliance (Mesma Titularidade) | Compliance | Email: compliance@lbpay.com.br |
| Database | DBA/DevOps | Slack: #database |
| Arquitetura (Sync/Async) | Arquiteto | Email: arquitetura@lbpay.com.br |

---

## 12. Apêndice: Exemplo de Implementação (Referência)

### Como Charge (COB) funciona atualmente

**Fluxo Atual:**
```
1. Cliente → POST /v1/cob (Seamless PIX REST)
2. Seamless PIX → CreateChargeHandler
3. Handler → CreateChargeUseCase
4. UseCase → ChargeServiceAdapter
5. Adapter → OrchestrationChargeServiceClient (gRPC)
6. gRPC Client → Orchestration Service (:50051)
7. Orchestration → Processa e persiste
8. Response ← volta pela cadeia
```

**Aplicar para PIX OUT:**
```
1. Cliente → POST /v1/pix/payments (Seamless PIX REST)
2. Seamless PIX → CreatePixPaymentHandler
3. Handler → CreatePixPaymentUseCase
4. UseCase → PixOutServiceAdapter
5. Adapter → PaymentPixServiceClient (gRPC)
6. gRPC Client → Payment Service (:50051)
7. Payment → Valida, processa, persiste
8. Response ← volta pela cadeia
```

**Conclusão:** Seguir exatamente o mesmo padrão arquitetural!

---

**Fim do Documento**

**Próxima Atualização:** Após reunião de alinhamento (12/12/2025)
