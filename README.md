# LB B2B API PIX - Especificações Técnicas

Repositório de especificações e documentação técnica para implementação do **PIX OUT B2B API** na plataforma LBPay.

## 📋 Sobre este Repositório

Este repositório contém **apenas documentação e especificações técnicas**. O código de implementação está no monorepo `money-moving` (repositório separado).

## 📚 Documentação Principal

### Especificação de Escopo
- **[PIX_OUT_ESCOPO_IMPLEMENTACAO.md](specs_docs/PIX_OUT_ESCOPO_IMPLEMENTACAO.md)** ⭐
  - Especificação completa e validada dos 6 pontos de escopo
  - Análise da arquitetura existente (Seamless PIX + Payment service)
  - Decisões técnicas pendentes
  - Roadmap de implementação em 6 sprints

### Especificações Detalhadas
- **[PIX_OUT_B2B_COMPLETE_SPEC.md](specs_docs/PIX_OUT_B2B_COMPLETE_SPEC.md)**
  - Especificação original completa com 10 features
  - Endpoints REST API detalhados
  - Schemas de banco de dados
  - Especificações de webhooks

- **[PIX_OUT_REST_TO_GRPC_MAPPING.md](specs_docs/PIX_OUT_REST_TO_GRPC_MAPPING.md)**
  - Mapeamento completo REST → gRPC
  - 7 tipos de pagamento PIX com exemplos JSON
  - Definições proto
  - Regras de validação

### Documentos de Apoio
- **[IMPLEMENTATION_PROGRESS.md](specs_docs/IMPLEMENTATION_PROGRESS.md)**
  - Tracking de progresso de implementação
  - Estrutura de arquivos e diretórios
  - Decisões técnicas documentadas

- **[DUVIDAS_IMPLEMENTACAO_SIMPLES.md](specs_docs/DUVIDAS_IMPLEMENTACAO_SIMPLES.md)**
  - Questões técnicas esclarecidas
  - Infraestrutura existente documentada

## 🎯 Escopo do Projeto

### Endpoints Principais (6 pontos)

1. **Enviar PIX por Chave PIX**
   - `POST /v1/pix/payments`
   - Tipos de chave: CPF, CNPJ, Email, Phone, EVP

2. **Enviar PIX por Dados Bancários**
   - `POST /v1/pix/payments/account`
   - ISPB + Branch + Account Number

3. **Enviar PIX por QR Code**
   - `POST /v1/pix/payments/qrcode`
   - BR Code / PIX Copia e Cola

4. **Detalhar QR Code PIX**
   - `POST /v1/pix/qrcodes/decode`
   - Decodificar sem iniciar pagamento

5. **Consultar por ID**
   - `GET /v1/pix/payments/{payment_id}`

6. **Consultar por End-to-End ID**
   - `GET /v1/pix/payments/e2e/{e2e_id}`

7. **Listar com Paginação**
   - `GET /v1/pix/payments`

### Funcionalidades Adicionais

- ✅ **Validação de Mesma Titularidade** (Portaria SPA/ME nº 615/2024)
- ✅ **Webhooks com Persistência** e retry logic
- ✅ **Integração com DICT/SPI** (já funcionando)
- ✅ **Integração com Ledger v2** (já funcionando)

## 🏗️ Arquitetura

```
┌─────────────────┐      HTTP REST       ┌──────────────────┐
│                 │  ───────────────────> │                  │
│  Seamless PIX   │                       │  Payment Service │
│  (REST API)     │  <─────────────────── │  (gRPC/REST)     │
│                 │                        │                  │
└─────────────────┘                       └──────────────────┘
        │                                          │
        │                                          │
        v                                          v
   Fiber/Go                               DICT/SPI/Ledger v2
   Middleware                             Integrações BACEN
```

## ⚠️ Decisões Técnicas Pendentes

Antes de iniciar a implementação, precisam ser tomadas **3 decisões bloqueadoras**:

1. **Protocolo Seamless → Payment**
   - [ ] Manter HTTP REST (padrão atual)
   - [ ] Migrar para gRPC (novo padrão)

2. **Interface do Payment Service**
   - [ ] Apenas gRPC
   - [ ] Apenas HTTP REST
   - [ ] Ambos (dual interface)

3. **Database para Migrations**
   - [ ] Payment service database
   - [ ] Seamless database
   - [ ] Database dedicado PIX

## 📦 Database Migrations (Criadas)

Migrations prontas em `/apps/payment/migrations/`:

1. **001_create_pix_payments_b2b.sql**
   - Tabela principal de pagamentos PIX OUT
   - Suporta 3 métodos: pix_key, account, qrcode
   - 12 indexes para performance

2. **002_create_same_ownership_registry.sql**
   - Registry de validação de mesma titularidade
   - Funções helper para validação
   - Suporte para bulk import

3. **003_create_webhook_deliveries.sql**
   - Tracking de webhooks
   - Retry logic com exponential backoff
   - Dead letter queue

## 🚀 Roadmap de Implementação

### Sprint 1: Foundation (1 semana)
- [ ] Rodar migrations
- [ ] Criar models (request/response)
- [ ] Setup gRPC client (se aplicável)

### Sprint 2-3: Core Endpoints (2 semanas)
- [ ] SendPixByKey
- [ ] SendPixByAccount
- [ ] SendPixByQRCode

### Sprint 4: Query Endpoints (1 semana)
- [ ] GetPayment
- [ ] GetPaymentByE2E
- [ ] ListPayments

### Sprint 5: Utilities (1 semana)
- [ ] DecodeQRCode
- [ ] Same Ownership Validation

### Sprint 6: Webhooks (1 semana)
- [ ] Webhook worker
- [ ] Retry logic
- [ ] Dead letter queue

**Estimativa Total**: 4-6 semanas (2 desenvolvedores)

## 🔗 Repositórios Relacionados

- **money-moving**: Monorepo com código de implementação (Seamless PIX + Payment service)
- Este repositório contém apenas especificações e documentação

## 📝 Notas

- Este é um projeto de **extensão** de infraestrutura existente
- **NÃO** estamos construindo do zero
- Padrão de referência: COB/COBV (Cobrança Imediata)
- Todas as especificações foram validadas com a equipe

## 👥 Equipe

- **Arquitetura**: Especificações completas de extensão do Seamless PIX e Payment service
- **Desenvolvimento**: A ser implementado no monorepo `money-moving`

---

**Status**: 📋 Especificações completas | ⏳ Aguardando decisões técnicas | 🚧 Implementação pendente
