# PIX OUT B2B API - Implementation Progress

> **Last Updated:** 2025-12-09
> **Status:** Foundation Complete - Ready for Implementation

---

## Executive Summary

The foundation for the PIX OUT B2B API has been successfully completed. This includes:

1. Complete gRPC service definition (proto files)
2. Database schema with 3 core tables
3. Comprehensive specifications and mappings
4. Clear implementation roadmap

**Next Steps:** Begin implementation of gRPC service handlers and REST API layer.

---

## Completed Work

### 1. Proto Files Generated

**Location:** `/Users/jose.silva.lb/LondonBridge/money-moving/apps/payment/proto/`

**Files Created:**
- [pix_payment_b2b.proto](../../../LondonBridge/money-moving/apps/payment/proto/pix_payment_b2b.proto) - Complete service definition
- `pix_payment_b2b.pb.go` - Auto-generated Go structs (98KB)
- `pix_payment_b2b_grpc.pb.go` - Auto-generated gRPC service code (24KB)

**Service Definition:**
```protobuf
service PixPaymentB2BService {
  // Payment initiation (3 methods)
  rpc InitiatePixPaymentByKey(...)
  rpc InitiatePixPaymentByAccount(...)
  rpc InitiatePixPaymentByQRCode(...)

  // QR Code operations (1 method)
  rpc DecodeQRCode(...)

  // Query operations (3 methods)
  rpc GetPixPayment(...)
  rpc GetPixPaymentByE2EId(...)
  rpc ListPixPayments(...)

  // Management (1 method)
  rpc CancelPixPayment(...)

  // Webhooks (2 methods)
  rpc ListWebhookDeliveries(...)
  rpc RetryWebhookDelivery(...)
}
```

**Total:** 11 gRPC methods covering all 10 required features

### 2. Database Migrations Created

**Location:** `/Users/jose.silva.lb/LondonBridge/money-moving/apps/payment/migrations/`

#### Migration 001: Main Payments Table
**File:** [001_create_pix_payments_b2b.sql](../../../LondonBridge/money-moving/apps/payment/migrations/001_create_pix_payments_b2b.sql)

**Features:**
- Supports 3 payment methods: `pix_key`, `account`, `qrcode`
- Validates payment status flow
- Tracks same ownership validation
- Stores custom metadata (JSONB)
- Idempotency support
- 12 performance indexes
- Auto-updating `updated_at` trigger

**Key Columns:**
```sql
payment_id         VARCHAR(50)    -- Unique payment identifier
payment_method     VARCHAR(20)    -- pix_key | account | qrcode
pix_key           VARCHAR(77)    -- PIX key (if method = pix_key)
payee_account     (multiple)     -- Bank account (if method = account)
qrcode            TEXT           -- BR Code (if method = qrcode)
e2e_id            VARCHAR(32)    -- BACEN End-to-End ID
validate_same_ownership BOOLEAN  -- Portaria SPA/ME compliance
```

#### Migration 002: Same Ownership Registry
**File:** [002_create_same_ownership_registry.sql](../../../LondonBridge/money-moving/apps/payment/migrations/002_create_same_ownership_registry.sql)

**Features:**
- Internal PIX key-to-document mapping
- Compliance with Portaria SPA/ME nº 615/2024
- Soft delete support (is_active flag)
- Multiple validation sources (DICT, manual, import)
- Helper functions for validation and bulk import

**Helper Functions:**
```sql
validate_same_ownership(pix_key, document)         -- Check if key belongs to document
deactivate_same_ownership_entry(pix_key, ...)      -- Soft delete entry
import_same_ownership_entries(entries_jsonb)       -- Bulk import from JSON
```

#### Migration 003: Webhook Deliveries
**File:** [003_create_webhook_deliveries.sql](../../../LondonBridge/money-moving/apps/payment/migrations/003_create_webhook_deliveries.sql)

**Features:**
- Webhook delivery tracking and persistence
- Retry logic with exponential backoff (0s, 60s, 5m, 15m, 1h)
- Dead letter queue for failed webhooks
- Request/response audit trail
- Row-level locking for worker processing

**Helper Functions:**
```sql
create_webhook_delivery(payment_id, event, ...)    -- Create new webhook
get_webhooks_for_retry(limit)                      -- Get pending webhooks (with lock)
mark_webhook_delivered(webhook_id, ...)            -- Mark as delivered
mark_webhook_for_retry(webhook_id, ...)            -- Schedule retry
calculate_retry_delay(attempt_number)              -- Exponential backoff + jitter
get_webhook_stats(payment_id)                      -- Delivery statistics
```

### 3. Specification Documents

All specifications are available in [specs_docs/](../):

1. **[API_LB.md](API_LB.md)** (4,590 lines)
   - Complete REST API specification v1.0
   - OAuth 2.0 + mTLS security
   - 60+ endpoints
   - BACEN compliance requirements

2. **[PIX_OUT_B2B_COMPLETE_SPEC.md](PIX_OUT_B2B_COMPLETE_SPEC.md)** (1,130 lines)
   - Complete feature specification
   - All 10 required features
   - REST API endpoints
   - Database schemas
   - Webhook specifications

3. **[PIX_OUT_REST_TO_GRPC_MAPPING.md](PIX_OUT_REST_TO_GRPC_MAPPING.md)** (900 lines)
   - Detailed mapping for all 7 PIX payment types
   - JSON request/response examples
   - Validation rules with regex patterns
   - Error response structures
   - Field-by-field proto definitions

4. **[PIX_OUT_LOTE_PLANNING.md](PIX_OUT_LOTE_PLANNING.md)**
   - Original batch payment planning (superseded but kept for reference)

---

## Architecture Overview

### Request Flow

```
Client B2B (PJ)
    ↓
Kong API Gateway
    ↓
Seamless PIX REST API (:8081)
    ├─ POST /v1/pix/payments (by key)
    ├─ POST /v1/pix/payments/account (by account)
    ├─ POST /v1/pix/payments/qrcode (by QR Code)
    ├─ POST /v1/pix/qrcodes/decode
    ├─ GET /v1/pix/payments/{id}
    ├─ GET /v1/pix/payments/e2e/{e2e_id}
    ├─ GET /v1/pix/payments
    └─ DELETE /v1/pix/payments/{id}
    ↓
Payment gRPC Service (:50051)
    └─ PixPaymentB2BService
    ↓
    ├─→ DICT Service (PIX key validation)
    ├─→ Same Ownership Validator (internal table)
    ├─→ QR Code Parser (BR Code decoder)
    ├─→ Ledger v2 (balance reservation)
    ├─→ SPI/BACEN (payment submission)
    └─→ Webhook Delivery Service
```

### Database Schema

```
payments.pix_payments_b2b
├─ id (UUID)
├─ payment_id (VARCHAR 50, unique)
├─ payment_method (pix_key | account | qrcode)
├─ status (pending → validating → processing → completed/failed)
├─ pix_key, pix_key_type (if payment_method = pix_key)
├─ payee_account details (if payment_method = account)
├─ qrcode, qrcode_parsed (if payment_method = qrcode)
├─ validate_same_ownership (boolean)
├─ e2e_id (BACEN End-to-End ID)
└─ metadata (JSONB)

payments.same_ownership_registry
├─ id (UUID)
├─ pix_key (VARCHAR 77, unique)
├─ document (VARCHAR 14)
├─ bank_ispb, bank_name
├─ is_active (soft delete)
└─ validation_source (dict | manual | import)

payments.webhook_deliveries
├─ id (UUID)
├─ webhook_id (VARCHAR 50, unique)
├─ payment_id (FK → pix_payments_b2b)
├─ event (e.g., pix.payment.completed)
├─ delivery_status (pending | delivered | failed | dead_letter)
├─ attempt_number (1-5)
├─ request_payload, response_body (JSONB)
└─ next_retry_at (for retry queue)
```

---

## Implementation Roadmap

### Phase 1: Core Infrastructure (Sprint 1-2)

**gRPC Service Implementation**
- [ ] Create gRPC server bootstrap in Payment service
- [ ] Implement service registration
- [ ] Configure gRPC server port (50051)
- [ ] Add health check endpoint

**Database Setup**
- [ ] Run migration 001 (pix_payments_b2b)
- [ ] Run migration 002 (same_ownership_registry)
- [ ] Run migration 003 (webhook_deliveries)
- [ ] Test all helper functions

**Repository Layer**
- [ ] Create `PixPaymentB2BRepository` interface
- [ ] Implement PostgreSQL repository
- [ ] Create `SameOwnershipRepository` interface
- [ ] Create `WebhookDeliveryRepository` interface

### Phase 2: Payment Initiation (Sprint 3-4)

**Payment by PIX Key**
- [ ] Implement `InitiatePixPaymentByKey` handler
- [ ] PIX key validation (CPF, CNPJ, Email, Phone, EVP)
- [ ] DICT integration for key lookup
- [ ] Same ownership validation (if enabled)
- [ ] Idempotency check

**Payment by Bank Account**
- [ ] Implement `InitiatePixPaymentByAccount` handler
- [ ] Account validation
- [ ] Bank ISPB lookup

**Payment by QR Code**
- [ ] Implement `InitiatePixPaymentByQRCode` handler
- [ ] BR Code parser (EMV format)
- [ ] QR Code validation (CRC, expiration)
- [ ] Implement `DecodeQRCode` handler

### Phase 3: Processing Pipeline (Sprint 5-6)

**Payment Processing Workers**
- [ ] Create validation worker
  - PIX key validation
  - Balance check (Ledger v2)
  - Same ownership check
- [ ] Create processing worker
  - Generate End-to-End ID (E2E)
  - Submit to SPI/BACEN
  - Update payment status
- [ ] Create completion worker
  - Handle SPI callbacks
  - Update final status
  - Trigger webhooks

**Ledger v2 Integration**
- [ ] Implement balance reservation
- [ ] Implement debit operation (PIX_OUT_PENDING)
- [ ] Implement confirmation (PIX_OUT_CONFIRM)
- [ ] Implement rejection (PIX_OUT_REJECT)

### Phase 4: Query Operations (Sprint 7)

**Query Handlers**
- [ ] Implement `GetPixPayment` handler
- [ ] Implement `GetPixPaymentByE2EId` handler
- [ ] Implement `ListPixPayments` handler
  - Pagination (cursor-based)
  - Filters (account, status, date range, PIX key, E2E ID)
  - Sorting

**Management**
- [ ] Implement `CancelPixPayment` handler
- [ ] Cancel validation (only pending/validating)

### Phase 5: Webhooks (Sprint 8)

**Webhook Delivery System**
- [ ] Create webhook worker
- [ ] Implement retry logic (exponential backoff)
- [ ] Implement webhook signature (HMAC SHA-256)
- [ ] Dead letter queue handling

**Webhook Handlers**
- [ ] Implement `ListWebhookDeliveries` handler
- [ ] Implement `RetryWebhookDelivery` handler

### Phase 6: REST API Layer (Sprint 9-10)

**Seamless PIX REST Endpoints**
- [ ] Create REST handlers in Seamless PIX
- [ ] Implement gRPC client calls
- [ ] Request/response mappers (JSON ↔ Proto)
- [ ] Error handling and mapping

**Authentication & Authorization**
- [ ] OAuth 2.0 integration
- [ ] mTLS certificate validation
- [ ] Account-level permissions

### Phase 7: Testing & Quality (Sprint 11-12)

**Unit Tests**
- [ ] gRPC handler tests
- [ ] Repository tests
- [ ] Validation tests
- [ ] Worker tests

**Integration Tests**
- [ ] End-to-end payment flow tests
- [ ] Webhook delivery tests
- [ ] Same ownership validation tests
- [ ] QR Code parsing tests

**Performance Tests**
- [ ] Load testing (100 TPS target)
- [ ] Database query optimization
- [ ] Index validation

### Phase 8: Documentation & Deployment (Sprint 13)

**Documentation**
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Integration guide for clients
- [ ] Operational runbook
- [ ] Troubleshooting guide

**Deployment**
- [ ] Staging environment deployment
- [ ] Production environment preparation
- [ ] Monitoring and alerting setup
- [ ] Rollback procedures

---

## File Structure

```
money-moving/
└─ apps/
   └─ payment/
      ├─ proto/
      │  ├─ pix_payment_b2b.proto          ✅ Complete
      │  ├─ pix_payment_b2b.pb.go          ✅ Generated
      │  └─ pix_payment_b2b_grpc.pb.go     ✅ Generated
      │
      ├─ migrations/
      │  ├─ 001_create_pix_payments_b2b.sql           ✅ Complete
      │  ├─ 002_create_same_ownership_registry.sql    ✅ Complete
      │  └─ 003_create_webhook_deliveries.sql         ✅ Complete
      │
      ├─ internal/
      │  ├─ grpc/
      │  │  ├─ server.go                   ⏳ TODO
      │  │  └─ handlers/
      │  │     └─ pix_payment_b2b.go       ⏳ TODO
      │  │
      │  ├─ repository/
      │  │  ├─ pix_payment_b2b.go          ⏳ TODO
      │  │  ├─ same_ownership.go           ⏳ TODO
      │  │  └─ webhook_delivery.go         ⏳ TODO
      │  │
      │  ├─ usecase/
      │  │  ├─ initiate_payment.go         ⏳ TODO
      │  │  ├─ validate_payment.go         ⏳ TODO
      │  │  ├─ process_payment.go          ⏳ TODO
      │  │  └─ query_payment.go            ⏳ TODO
      │  │
      │  ├─ worker/
      │  │  ├─ payment_validator.go        ⏳ TODO
      │  │  ├─ payment_processor.go        ⏳ TODO
      │  │  └─ webhook_delivery.go         ⏳ TODO
      │  │
      │  └─ adapter/
      │     ├─ dict_client.go              ⏳ TODO
      │     ├─ spi_client.go               ⏳ TODO
      │     ├─ ledger_v2_client.go         ⏳ TODO
      │     └─ qrcode_parser.go            ⏳ TODO
      │
      └─ ...

LB_B2B_API_PIX/
└─ specs_docs/
   ├─ API_LB.md                                    ✅ Complete
   ├─ PIX_OUT_B2B_COMPLETE_SPEC.md                 ✅ Complete
   ├─ PIX_OUT_REST_TO_GRPC_MAPPING.md              ✅ Complete
   ├─ PIX_OUT_LOTE_PLANNING.md                     ✅ Complete (reference)
   └─ IMPLEMENTATION_PROGRESS.md                   ✅ This file
```

---

## Key Technical Decisions

### 1. Payment Method Polymorphism
**Decision:** Use single table with `payment_method` discriminator instead of separate tables.

**Rationale:**
- Simpler queries for listing all payments
- Easier status tracking across all payment types
- Reduced join complexity
- Check constraints ensure data integrity

### 2. Same Ownership Internal Registry
**Decision:** Maintain internal table instead of always querying DICT.

**Rationale:**
- Compliance with Portaria SPA/ME nº 615/2024
- Performance (avoid external API calls)
- Cost reduction (DICT queries may be billed)
- Audit trail (track validation sources)

### 3. Webhook Persistence
**Decision:** Store all webhook attempts with full request/response details.

**Rationale:**
- Debugging support
- Audit compliance
- Retry capability
- Client transparency

### 4. Cursor-Based Pagination
**Decision:** Use cursor-based pagination instead of offset-based.

**Rationale:**
- Better performance on large datasets
- Consistent results (no duplicates or skips during pagination)
- Standard approach for APIs

### 5. gRPC for Internal Communication
**Decision:** Use gRPC between Seamless PIX and Payment service.

**Rationale:**
- Type safety (proto definitions)
- Better performance than REST
- Streaming support (future)
- Consistent with existing architecture (Ledger v2)

---

## Dependencies

### External Services
- **DICT (BACEN)**: PIX key validation and lookup
- **SPI (BACEN)**: Payment submission and processing
- **Ledger v2**: Balance management and transaction recording
- **Kong**: API Gateway for authentication and routing

### Libraries/Frameworks
- **gRPC**: Service communication
- **PostgreSQL**: Data persistence
- **Fiber**: REST API framework (Seamless PIX)
- **Protocol Buffers**: Service definitions

---

## Security & Compliance

### Authentication
- OAuth 2.0 (client credentials flow)
- mTLS (mutual TLS certificate validation)

### Compliance
- **LGPD**: Data privacy and retention
- **Portaria SPA/ME nº 615/2024**: Same ownership validation
- **BACEN Regulations**: E2E ID format, log retention (5 years)
- **PCI-DSS**: Secure payment processing

### Data Protection
- Sensitive data encryption at rest
- TLS 1.3 for data in transit
- Audit logging for all operations
- Role-based access control (RBAC)

---

## Monitoring & Observability

### Metrics to Track
- Payment volume (by type, status)
- Success rate (%)
- Average processing time
- Webhook delivery success rate
- Same ownership validation hit rate
- Error rates by error code

### Alerts
- Payment failure rate > 5%
- Webhook dead letter queue growth
- Processing time > 30s
- DICT/SPI API errors

### Logs
- Structured logging (JSON format)
- Correlation IDs for request tracing
- Payment status transitions
- Webhook delivery attempts

---

## Next Steps

### Immediate Actions (This Week)
1. Review and approve proto definitions
2. Run database migrations in development environment
3. Set up development environment for Payment service
4. Create skeleton project structure

### Next Sprint (Sprint 1)
1. Implement gRPC server bootstrap
2. Create repository layer
3. Implement first handler: `InitiatePixPaymentByKey`
4. Set up unit testing framework

### Questions to Resolve
- [ ] DICT API credentials and environment
- [ ] SPI integration endpoint and authentication
- [ ] Ledger v2 gRPC client configuration
- [ ] Message queue selection (Kafka vs Pulsar)
- [ ] Monitoring tools (Prometheus, Grafana, Datadog?)

---

## References

- [Complete Specification](PIX_OUT_B2B_COMPLETE_SPEC.md)
- [REST to gRPC Mapping](PIX_OUT_REST_TO_GRPC_MAPPING.md)
- [Proto Definition](../../../LondonBridge/money-moving/apps/payment/proto/pix_payment_b2b.proto)
- [Database Migrations](../../../LondonBridge/money-moving/apps/payment/migrations/)

---

**Document Owner:** LBPay Engineering Team
**Last Updated:** 2025-12-09
**Status:** Ready for Implementation
