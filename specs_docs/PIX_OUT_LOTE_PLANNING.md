# Planejamento: Squad PIX Batch Payments

> **Feature:** PIX OUT em Lote para Clientes PJ
> **Duração:** 8 Sprints (16 semanas / 4 meses)
> **Data Início:** 2025-01-13
> **Data Fim:** 2025-05-09

---

## 1. Estrutura da Squad

### 1.1 Composição do Time

| Papel | Nome | Responsabilidade | Alocação |
|-------|------|------------------|----------|
| **Product Owner** | [TBD] | Priorização, requisitos de negócio, aceite | 50% |
| **Tech Lead** | [TBD] | Arquitetura, code reviews, decisões técnicas | 100% |
| **Backend Engineer 1** | [TBD] | gRPC service, proto, validações | 100% |
| **Backend Engineer 2** | [TBD] | REST API, Seamless PIX integration | 100% |
| **Backend Engineer 3** | [TBD] | Batch processor worker, event handling | 100% |
| **QA Engineer** | [TBD] | Testes automatizados, E2E, performance | 100% |
| **DevOps Engineer** | [TBD] | Infrastructure, CI/CD, monitoring | 50% |
| **UX/UI Designer** | [TBD] | Dashboard updates (se aplicável) | 25% |
| **Compliance Analyst** | [TBD] | Validação regulatória, auditoria | 25% |

**Total:** 5.5 FTEs

---

## 2. Metodologia

### 2.1 Framework

- **Scrum** com sprints de 2 semanas
- **Daily standups** (15 min, 9:30 AM)
- **Sprint Planning** (Segunda-feira, 2h)
- **Sprint Review** (Sexta-feira, 1h)
- **Sprint Retrospective** (Sexta-feira, 1h)
- **Backlog Refinement** (Quarta-feira, 1h)

### 2.2 Ferramentas

- **Jira**: Gestão de épicos, stories, tasks, bugs
- **GitHub**: Code repository, PRs, code reviews
- **Slack**: Comunicação do time (#squad-pix-batch)
- **Confluence**: Documentação técnica
- **Figma**: Designs e protótipos (se aplicável)
- **Postman**: API testing e collections
- **Grafana**: Monitoring e dashboards

---

## 3. Épicos e User Stories

### Épico 1: Fundação e Infraestrutura
**Objetivo:** Criar a base técnica para a feature

#### User Stories:

**E1-US1: Definir Proto para Batch PIX Payment Service**
- **Como** desenvolvedor backend
- **Quero** ter as definições proto completas
- **Para que** eu possa gerar os stubs gRPC

**Critérios de Aceite:**
- [ ] Proto `batch_pix_payment.proto` criado
- [ ] 5 RPCs definidos (Create, Get, List, Cancel, Report)
- [ ] Enums para status (Batch e Item)
- [ ] Mensagens com validação via protoc
- [ ] Documentação inline nos protos

**Story Points:** 5
**Sprint:** Sprint 1

---

**E1-US2: Criar Schema do Banco de Dados**
- **Como** desenvolvedor backend
- **Quero** ter as tabelas e índices criados
- **Para que** eu possa persistir lotes e items

**Critérios de Aceite:**
- [ ] Tabela `batch_pix_payments` criada
- [ ] Tabela `batch_pix_payment_items` criada
- [ ] Índices de performance criados
- [ ] Constraints e foreign keys definidos
- [ ] Migration scripts (up/down)
- [ ] Testes de migration

**Story Points:** 3
**Sprint:** Sprint 1

---

**E1-US3: Criar Skeleton do gRPC Service**
- **Como** desenvolvedor backend
- **Quero** ter a estrutura do serviço gRPC
- **Para que** eu possa implementar as operações

**Critérios de Aceite:**
- [ ] Service registrado no Payment app
- [ ] Métodos vazios implementados (retornam Unimplemented)
- [ ] Testes unitários básicos
- [ ] Logging configurado
- [ ] Health check incluído

**Story Points:** 5
**Sprint:** Sprint 1

---

**E1-US4: Criar Endpoints REST no Seamless PIX**
- **Como** desenvolvedor backend
- **Quero** ter os 5 endpoints REST expostos
- **Para que** clientes possam consumir a API

**Critérios de Aceite:**
- [ ] POST /v1/pix/batch/payments
- [ ] GET /v1/pix/batch/payments/{batch_id}
- [ ] GET /v1/pix/batch/payments/{batch_id}/items
- [ ] DELETE /v1/pix/batch/payments/{batch_id}
- [ ] GET /v1/pix/batch/payments/{batch_id}/report
- [ ] Handlers criados (vazios ok)
- [ ] OpenAPI spec atualizado
- [ ] Postman collection

**Story Points:** 8
**Sprint:** Sprint 1-2

---

**E1-US5: Configurar Message Queue para Batch Processing**
- **Como** DevOps
- **Quero** ter o tópico Pulsar configurado
- **Para que** eventos de lote sejam publicados

**Critérios de Aceite:**
- [ ] Tópico `pix.batch.payments` criado
- [ ] Retention policy configurada (7 dias)
- [ ] Dead letter queue configurada
- [ ] Producer e Consumer clients testados
- [ ] Monitoring do tópico (Grafana)

**Story Points:** 3
**Sprint:** Sprint 2

---

### Épico 2: Core Business Logic
**Objetivo:** Implementar a lógica de criação e validação de lotes

#### User Stories:

**E2-US1: Implementar Criação de Lote (gRPC)**
- **Como** desenvolvedor backend
- **Quero** implementar `CreateBatchPixPayment`
- **Para que** lotes sejam persistidos e validados

**Critérios de Aceite:**
- [ ] Validação de schema (total_items, total_amount)
- [ ] Validação de saldo disponível
- [ ] Persistência em PostgreSQL
- [ ] Publicação de evento `batch.created`
- [ ] Idempotência via `idempotency_key`
- [ ] Testes unitários (90% coverage)
- [ ] Testes de integração

**Story Points:** 13
**Sprint:** Sprint 2-3

---

**E2-US2: Implementar Endpoint REST de Criação**
- **Como** desenvolvedor backend
- **Quero** conectar REST API ao gRPC
- **Para que** clientes possam criar lotes via HTTP

**Critérios de Aceite:**
- [ ] Handler `POST /v1/pix/batch/payments` completo
- [ ] Mappers (Domain DTO → gRPC → Proto)
- [ ] Validação de headers (Authorization, Idempotency-Key)
- [ ] Response 202 Accepted
- [ ] Location header com batch_id
- [ ] Error handling (400, 401, 403, 422, 429)
- [ ] Testes E2E

**Story Points:** 8
**Sprint:** Sprint 3

---

**E2-US3: Implementar Validações de PIX Key**
- **Como** desenvolvedor backend
- **Quero** validar formato de chaves PIX
- **Para que** apenas chaves válidas sejam aceitas

**Critérios de Aceite:**
- [ ] Validação de CPF (11 dígitos + check digit)
- [ ] Validação de CNPJ (14 dígitos + check digit)
- [ ] Validação de Email (RFC 5322, max 77 chars)
- [ ] Validação de Telefone (+55DDNNNNNNNNN)
- [ ] Validação de EVP (UUID v4)
- [ ] Testes unitários para cada tipo
- [ ] Error messages claros

**Story Points:** 5
**Sprint:** Sprint 3

---

**E2-US4: Implementar Consulta de Status (gRPC + REST)**
- **Como** desenvolvedor backend
- **Quero** implementar consulta de lote
- **Para que** clientes vejam progresso

**Critérios de Aceite:**
- [ ] gRPC `GetBatchPixPayment` implementado
- [ ] REST `GET /v1/pix/batch/payments/{batch_id}`
- [ ] Cálculo de `progress_percentage`
- [ ] Summary com totais (processed, successful, failed)
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 8
**Sprint:** Sprint 3

---

**E2-US5: Implementar Listagem de Items (gRPC + REST)**
- **Como** desenvolvedor backend
- **Quero** listar items de um lote
- **Para que** clientes vejam detalhes de cada pagamento

**Critérios de Aceite:**
- [ ] gRPC `ListBatchPixPaymentItems` implementado
- [ ] REST `GET /v1/pix/batch/payments/{batch_id}/items`
- [ ] Paginação cursor-based
- [ ] Filtro por status
- [ ] Ordenação por created_at DESC
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 8
**Sprint:** Sprint 4

---

### Épico 3: Batch Processing Worker
**Objetivo:** Processar lotes de forma assíncrona e eficiente

#### User Stories:

**E3-US1: Criar Worker para Consumo de Eventos**
- **Como** desenvolvedor backend
- **Quero** ter um worker que processe lotes
- **Para que** pagamentos sejam executados

**Critérios de Aceite:**
- [ ] Worker consome `pix.batch.payments` topic
- [ ] Processa evento `batch.created`
- [ ] Atualiza status para `processing`
- [ ] Publica evento `batch.processing`
- [ ] Health check endpoint
- [ ] Graceful shutdown
- [ ] Testes unitários

**Story Points:** 8
**Sprint:** Sprint 4

---

**E3-US2: Implementar Validação de Chaves PIX via DICT**
- **Como** desenvolvedor backend
- **Quero** validar chaves no DICT
- **Para que** apenas pagamentos para chaves válidas sejam processados

**Critérios de Aceite:**
- [ ] Cliente gRPC para DICT service
- [ ] Batch query (múltiplas chaves em uma chamada)
- [ ] Cache de resultados (5 min TTL)
- [ ] Tratamento de erro (chave não encontrada)
- [ ] Timeout configurável
- [ ] Retry com backoff exponencial
- [ ] Testes com mock do DICT

**Story Points:** 13
**Sprint:** Sprint 4-5

---

**E3-US3: Implementar Validação de Mesma Titularidade**
- **Como** desenvolvedor backend
- **Quero** validar CPF/CNPJ do pagador vs beneficiário
- **Para que** conformidade com Portaria 615/2024 seja garantida

**Critérios de Aceite:**
- [ ] Extração de CPF/CNPJ do DICT response
- [ ] Comparação com `payer_info.document`
- [ ] Marca item como `failed` se não bater (quando obrigatório)
- [ ] Log de auditoria com resultado da validação
- [ ] Testes unitários
- [ ] Testes de integração

**Story Points:** 5
**Sprint:** Sprint 5

---

**E3-US4: Implementar Reserva de Saldo Consolidada**
- **Como** desenvolvedor backend
- **Quero** reservar saldo uma única vez no Ledger v2
- **Para que** performance seja otimizada

**Critérios de Aceite:**
- [ ] Chamada gRPC `PIX_OUT_PENDING` com total do lote
- [ ] Armazenamento de reservation_id
- [ ] Liberação incremental conforme processamento
- [ ] Rollback se lote cancelado
- [ ] Tratamento de saldo insuficiente
- [ ] Testes com mock do Ledger

**Story Points:** 13
**Sprint:** Sprint 5

---

**E3-US5: Implementar Processamento de Items**
- **Como** desenvolvedor backend
- **Quero** processar cada item do lote
- **Para que** pagamentos PIX sejam enviados

**Critérios de Aceite:**
- [ ] Loop sobre items com status `pending`
- [ ] Chamada gRPC `PixTransactionInit` (Payment service)
- [ ] Atualização de status do item para `processing`
- [ ] Armazenamento de `e2e_id` e `txid`
- [ ] Tratamento de erro por item (não para o lote)
- [ ] Publicação de evento `batch.item.completed`
- [ ] Testes unitários

**Story Points:** 13
**Sprint:** Sprint 5-6

---

**E3-US6: Implementar Paralelização de Processamento**
- **Como** desenvolvedor backend
- **Quero** processar múltiplos items em paralelo
- **Para que** throughput seja maximizado

**Critérios de Aceite:**
- [ ] Worker pool com 10 goroutines/threads
- [ ] Semáforo para limitar concorrência
- [ ] Channel/Queue para distribuição de items
- [ ] Atualização thread-safe do status
- [ ] Graceful shutdown (aguarda items em processamento)
- [ ] Testes de concorrência

**Story Points:** 8
**Sprint:** Sprint 6

---

**E3-US7: Implementar Finalização de Lote**
- **Como** desenvolvedor backend
- **Quero** finalizar lote após todos items
- **Para que** cliente seja notificado

**Critérios de Aceite:**
- [ ] Atualização de status para `completed` ou `partial_success`
- [ ] Cálculo de summary (totais)
- [ ] Atualização de `completed_at`
- [ ] Publicação de evento `batch.completed`
- [ ] Liberação de reserva de saldo (se houver)
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 6

---

### Épico 4: Webhooks e Eventos
**Objetivo:** Notificar clientes sobre progresso do lote

#### User Stories:

**E4-US1: Implementar Publicação de Eventos**
- **Como** desenvolvedor backend
- **Quero** publicar eventos no Pulsar
- **Para que** webhook service possa consumir

**Critérios de Aceite:**
- [ ] Publisher para tópico `pix.batch.events`
- [ ] 9 tipos de eventos (batch.created, batch.processing, etc)
- [ ] Schema Avro ou Protobuf para eventos
- [ ] Retry em caso de falha de publicação
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 5

---

**E4-US2: Implementar Webhook Delivery**
- **Como** desenvolvedor backend
- **Quero** enviar webhooks para `callback_url`
- **Para que** cliente seja notificado

**Critérios de Aceite:**
- [ ] Worker consome `pix.batch.events` topic
- [ ] HTTP POST para `callback_url`
- [ ] HMAC-SHA256 signature no header
- [ ] Timeout de 10 segundos
- [ ] Retry com backoff exponencial (5 tentativas)
- [ ] Dead letter queue após 5 falhas
- [ ] Testes com mock HTTP server

**Story Points:** 13
**Sprint:** Sprint 5-6

---

**E4-US3: Implementar Configuração de Webhook no Lote**
- **Como** cliente PJ
- **Quero** configurar `callback_url` ao criar lote
- **Para que** eu seja notificado automaticamente

**Critérios de Aceite:**
- [ ] Campo `callback_url` no request
- [ ] Validação de URL (formato HTTPS)
- [ ] Persistência no banco
- [ ] Teste de conectividade (opcional)
- [ ] Testes E2E

**Story Points:** 3
**Sprint:** Sprint 5

---

### Épico 5: Cancelamento e Relatórios
**Objetivo:** Permitir cancelamento e geração de relatórios

#### User Stories:

**E5-US1: Implementar Cancelamento de Lote (gRPC + REST)**
- **Como** cliente PJ
- **Quero** cancelar um lote
- **Para que** pagamentos pendentes não sejam processados

**Critérios de Aceite:**
- [ ] gRPC `CancelBatchPixPayment` implementado
- [ ] REST `DELETE /v1/pix/batch/payments/{batch_id}`
- [ ] Cancela apenas items `pending`
- [ ] Atualiza status para `cancelled`
- [ ] Libera reserva de saldo
- [ ] Publicação de evento `batch.cancelled`
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 8
**Sprint:** Sprint 6

---

**E5-US2: Implementar Geração de Relatório JSON**
- **Como** cliente PJ
- **Quero** baixar relatório do lote em JSON
- **Para que** eu possa processar programaticamente

**Critérios de Aceite:**
- [ ] gRPC `GetBatchPixPaymentReport` (format=JSON)
- [ ] REST `GET /v1/pix/batch/payments/{batch_id}/report?format=json`
- [ ] Inclui summary e todos items
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 5
**Sprint:** Sprint 6

---

**E5-US3: Implementar Geração de Relatório CSV**
- **Como** cliente PJ
- **Quero** baixar relatório em CSV
- **Para que** eu possa abrir no Excel

**Critérios de Aceite:**
- [ ] gRPC `GetBatchPixPaymentReport` (format=CSV)
- [ ] REST `GET /v1/pix/batch/payments/{batch_id}/report?format=csv`
- [ ] Content-Type: text/csv
- [ ] Content-Disposition header
- [ ] Encoding UTF-8 com BOM
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 7

---

**E5-US4: Implementar Geração de Relatório PDF (Opcional)**
- **Como** cliente PJ
- **Quero** baixar relatório em PDF
- **Para que** eu tenha formato imprimível

**Critérios de Aceite:**
- [ ] gRPC `GetBatchPixPaymentReport` (format=PDF)
- [ ] REST `GET /v1/pix/batch/payments/{batch_id}/report?format=pdf`
- [ ] Content-Type: application/pdf
- [ ] Logo LB Pay no cabeçalho
- [ ] Tabela com items
- [ ] Summary no rodapé
- [ ] Testes unitários

**Story Points:** 8
**Sprint:** Sprint 7 (Opcional - Nice to Have)

---

### Épico 6: Performance e Resiliência
**Objetivo:** Garantir performance e confiabilidade

#### User Stories:

**E6-US1: Implementar Circuit Breaker para SPI**
- **Como** desenvolvedor backend
- **Quero** ter circuit breaker para SPI/BACEN
- **Para que** falhas em cascata sejam evitadas

**Critérios de Aceite:**
- [ ] Circuit breaker com 3 estados (closed, open, half-open)
- [ ] Threshold: 50% erro em 1 minuto
- [ ] Timeout: 30 segundos
- [ ] Retry após 1 minuto
- [ ] Métricas expostas
- [ ] Testes unitários

**Story Points:** 8
**Sprint:** Sprint 7

---

**E6-US2: Implementar Rate Limiting por Cliente**
- **Como** Tech Lead
- **Quero** limitar lotes por cliente
- **Para que** recursos sejam balanceados

**Critérios de Aceite:**
- [ ] Máximo 50 lotes simultâneos por `account_id`
- [ ] Máximo 5,000 items pendentes por `account_id`
- [ ] Response 429 se limite excedido
- [ ] Header `X-RateLimit-*` nos responses
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 7

---

**E6-US3: Implementar Caching de Validações DICT**
- **Como** desenvolvedor backend
- **Quero** cachear resultados do DICT
- **Para que** performance seja melhorada

**Critérios de Aceite:**
- [ ] Redis cache com TTL de 5 minutos
- [ ] Cache key: `dict:{pix_key}`
- [ ] Cache miss → consulta DICT → armazena
- [ ] Cache hit → retorna direto
- [ ] Invalidação manual via endpoint (admin)
- [ ] Métricas de hit rate
- [ ] Testes unitários

**Story Points:** 8
**Sprint:** Sprint 7

---

**E6-US4: Load Testing**
- **Como** QA Engineer
- **Quero** executar testes de carga
- **Para que** capacidade seja validada

**Critérios de Aceite:**
- [ ] Script k6 ou Gatling
- [ ] Cenário: 1,000 lotes com 100 items cada
- [ ] Cenário: 100 lotes com 1,000 items cada
- [ ] Duração: 1 hora
- [ ] Target: 500 items/seg
- [ ] p95 latency < 500ms por item
- [ ] Report com gráficos

**Story Points:** 13
**Sprint:** Sprint 8

---

### Épico 7: Monitoramento e Documentação
**Objetivo:** Observabilidade e documentação completa

#### User Stories:

**E7-US1: Criar Dashboard Grafana**
- **Como** DevOps
- **Quero** ter dashboard de monitoramento
- **Para que** operações sejam observáveis

**Critérios de Aceite:**
- [ ] Dashboard "Batch PIX Overview"
- [ ] Dashboard "Item Processing"
- [ ] Métricas: throughput, latency, error rate
- [ ] Gráficos de série temporal
- [ ] Painéis de top erros
- [ ] Variáveis para filtrar por account_id

**Story Points:** 8
**Sprint:** Sprint 7

---

**E7-US2: Configurar Alertas**
- **Como** DevOps
- **Quero** ter alertas configurados
- **Para que** incidentes sejam detectados

**Critérios de Aceite:**
- [ ] Alerta: Taxa de erro > 10%
- [ ] Alerta: p95 latency > 15min
- [ ] Alerta: Fila > 20,000 items
- [ ] Alerta: Worker down
- [ ] Notificação via Slack + PagerDuty
- [ ] Runbook linkado no alerta

**Story Points:** 5
**Sprint:** Sprint 7

---

**E7-US3: Escrever Documentação Técnica**
- **Como** Tech Writer
- **Quero** documentação completa
- **Para que** desenvolvedores possam integrar

**Critérios de Aceite:**
- [ ] README.md atualizado
- [ ] API Reference (OpenAPI)
- [ ] Integration Guide
- [ ] Troubleshooting Guide
- [ ] Runbook de Operações
- [ ] Postman Collection atualizada
- [ ] Exemplos de código (cURL, Python, Java)

**Story Points:** 8
**Sprint:** Sprint 8

---

**E7-US4: Criar Guia de Integração para Clientes**
- **Como** Product Owner
- **Quero** guia para clientes
- **Para que** integração seja facilitada

**Critérios de Aceite:**
- [ ] Passo a passo: obter credenciais
- [ ] Passo a passo: criar primeiro lote
- [ ] Exemplos de payloads
- [ ] Códigos de erro e soluções
- [ ] FAQ
- [ ] Vídeo tutorial (opcional)

**Story Points:** 5
**Sprint:** Sprint 8

---

## 4. Backlog Priorizado

### Sprint 1 (Jan 13 - Jan 24)
**Objetivo:** Fundação técnica

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E1-US1 | Definir Proto | 5 | Backend 1 |
| E1-US2 | Criar Schema DB | 3 | Backend 1 |
| E1-US3 | Skeleton gRPC Service | 5 | Backend 1 |
| E1-US4 | Endpoints REST (parte 1) | 4 | Backend 2 |

**Total:** 17 pontos
**Velocity Esperada:** 15-20 pontos

---

### Sprint 2 (Jan 27 - Feb 07)
**Objetivo:** Completar infraestrutura e iniciar lógica

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E1-US4 | Endpoints REST (parte 2) | 4 | Backend 2 |
| E1-US5 | Configurar Message Queue | 3 | DevOps |
| E2-US1 | Criar Lote gRPC (parte 1) | 8 | Backend 1 |

**Total:** 15 pontos

---

### Sprint 3 (Feb 10 - Feb 21)
**Objetivo:** Core business logic - criação e validação

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E2-US1 | Criar Lote gRPC (parte 2) | 5 | Backend 1 |
| E2-US2 | Endpoint REST Criação | 8 | Backend 2 |
| E2-US3 | Validações PIX Key | 5 | Backend 3 |

**Total:** 18 pontos

---

### Sprint 4 (Feb 24 - Mar 07)
**Objetivo:** Consultas e início do worker

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E2-US4 | Consulta de Status | 8 | Backend 2 |
| E2-US5 | Listagem de Items | 8 | Backend 2 |
| E3-US1 | Worker Consumo Eventos | 8 | Backend 3 |

**Total:** 24 pontos (sprint mais pesada, considerar redução)

---

### Sprint 5 (Mar 10 - Mar 21)
**Objetivo:** Processamento de lotes

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E3-US2 | Validação DICT | 13 | Backend 3 |
| E3-US3 | Validação Titularidade | 5 | Backend 3 |
| E4-US1 | Publicação Eventos | 5 | Backend 1 |
| E4-US3 | Config Webhook | 3 | Backend 2 |

**Total:** 26 pontos (sprint pesada, considerar redução)

---

### Sprint 6 (Mar 24 - Apr 04)
**Objetivo:** Completar processamento e iniciar cancelamento

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E3-US4 | Reserva Saldo | 13 | Backend 1 |
| E3-US5 | Processar Items (parte 1) | 8 | Backend 3 |
| E4-US2 | Webhook Delivery (parte 1) | 8 | Backend 2 |

**Total:** 29 pontos (sprint muito pesada, reduzir)

---

### Sprint 7 (Apr 07 - Apr 18)
**Objetivo:** Performance, resiliência e monitoramento

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E3-US6 | Paralelização | 8 | Backend 3 |
| E3-US7 | Finalização Lote | 5 | Backend 3 |
| E5-US1 | Cancelamento | 8 | Backend 2 |
| E5-US2 | Relatório JSON | 5 | Backend 1 |
| E6-US1 | Circuit Breaker | 8 | Backend 1 |

**Total:** 34 pontos (sprint muito pesada, reduzir)

---

### Sprint 8 (Apr 21 - May 02)
**Objetivo:** Finalização, testes e documentação

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E5-US3 | Relatório CSV | 5 | Backend 1 |
| E6-US2 | Rate Limiting | 5 | Backend 2 |
| E6-US3 | Caching DICT | 8 | Backend 3 |
| E6-US4 | Load Testing | 13 | QA |
| E7-US1 | Dashboard Grafana | 8 | DevOps |
| E7-US2 | Alertas | 5 | DevOps |

**Total:** 44 pontos (sprint muito pesada, reduzir)

---

### Sprint 9 (Contingência - May 05 - May 16)
**Objetivo:** Buffer para atrasos e polish

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E7-US3 | Documentação Técnica | 8 | Tech Writer |
| E7-US4 | Guia Integração | 5 | Product |
| E5-US4 | Relatório PDF (opcional) | 8 | Backend 1 |
| - | Bug fixes | - | Toda squad |

**Total:** 21 pontos

---

## 5. Kanban Board Structure

### 5.1 Colunas

```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│  Backlog     │   To Do      │  In Progress │  In Review   │   Testing    │     Done     │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Stories      │ Stories      │ Stories      │ PRs          │ Test Cases   │ Deployed     │
│ priorizadas  │ sprint atual │ WIP limit: 5 │ Code Review  │ E2E Testing  │ Validated    │
│ refinadas    │              │              │              │              │              │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

### 5.2 WIP Limits

| Coluna | WIP Limit | Motivo |
|--------|-----------|--------|
| To Do | ∞ | Priorização livre |
| In Progress | 5 | Foco da squad (5 devs) |
| In Review | 10 | Tempo de code review |
| Testing | 8 | Tempo de execução de testes |
| Done | ∞ | Histórico |

### 5.3 Swimlanes

- **Epics**: Agrupar por épico
- **Priority**: P0 (Blocker), P1 (High), P2 (Medium), P3 (Low)
- **Type**: Feature, Bug, Tech Debt, Spike

---

## 6. Definition of Ready (DoR)

Uma story está pronta para ser iniciada quando:

- [ ] Critérios de aceite claramente definidos
- [ ] Dependências identificadas e resolvidas
- [ ] Design técnico aprovado (se aplicável)
- [ ] Estimativa de story points feita pela squad
- [ ] Prioridade definida pelo PO
- [ ] Testável (critérios de teste claros)

---

## 7. Definition of Done (DoD)

Uma story está completa quando:

- [ ] Código implementado conforme critérios de aceite
- [ ] Code review aprovado (2 aprovações mínimas)
- [ ] Testes unitários escritos (>80% coverage)
- [ ] Testes de integração escritos (se aplicável)
- [ ] Testes E2E passando (se aplicável)
- [ ] Documentação atualizada (README, OpenAPI, etc)
- [ ] Deployed em ambiente de staging
- [ ] PO aceitou a story (demo)
- [ ] Sem bugs críticos em aberto

---

## 8. Riscos e Mitigações

### 8.1 Riscos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **DICT rate limiting excessivo** | Alta | Alto | Cache agressivo + batch queries |
| **SPI downtime prolongado** | Média | Alto | Circuit breaker + retry com backoff |
| **Performance do worker insuficiente** | Média | Médio | Paralelização + auto-scaling |
| **Memory leaks em processamento longo** | Baixa | Alto | Health checks + auto-restart |
| **Deadlocks em reserva de saldo** | Baixa | Médio | Timeouts + lock ordering |

### 8.2 Riscos de Prazo

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **Sprints sobrecarregadas** | Alta | Alto | Reduzir pontos, priorizar MVP |
| **Dependências bloqueadoras** | Média | Médio | Identificar cedo, paralelizar |
| **Code reviews lentas** | Média | Médio | SLA de 24h para reviews |
| **Bugs críticos em produção** | Baixa | Alto | Testes rigorosos + canary deploy |

### 8.3 Riscos de Negócio

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **Mudança regulatória (BACEN)** | Baixa | Alto | Monitorar regulação, arquitetura flexível |
| **Baixa adoção por clientes** | Média | Médio | User research + beta testing |
| **Concorrentes lançam antes** | Média | Baixo | Time-to-market agressivo |

---

## 9. Métricas de Sucesso

### 9.1 Métricas de Engenharia

| Métrica | Target | Medição |
|---------|--------|---------|
| **Velocity** | 18-22 pontos/sprint | Jira |
| **Code Coverage** | >80% | SonarQube |
| **Code Review Time** | <24h | GitHub |
| **Deployment Frequency** | 2x/semana | CI/CD |
| **Lead Time** | <5 dias | Jira |
| **Change Failure Rate** | <5% | Incident tracking |
| **MTTR** | <2h | Incident tracking |

### 9.2 Métricas de Produto

| Métrica | Target (3 meses após launch) | Medição |
|---------|------------------------------|---------|
| **Lotes criados/dia** | 1,000 | Analytics |
| **Items processados/dia** | 50,000 | Analytics |
| **Taxa de sucesso** | >95% | Analytics |
| **Tempo médio processamento (lote 100 items)** | <3 min | Logs |
| **NPS (Net Promoter Score)** | >50 | Survey |
| **Adoção por clientes PJ** | 30% | CRM |

---

## 10. Cerimônias e Comunicação

### 10.1 Calendário de Cerimônias

**Weekly:**
- Segunda 09:00 - Sprint Planning (2h)
- Diariamente 09:30 - Daily Standup (15 min)
- Quarta 14:00 - Backlog Refinement (1h)
- Sexta 14:00 - Sprint Review (1h)
- Sexta 15:30 - Sprint Retrospective (1h)

**Bi-Weekly:**
- Tech Sync (30 min) - Discussões técnicas deep dive

**Monthly:**
- Stakeholder Update (1h) - Apresentação para liderança

### 10.2 Comunicação Assíncrona

**Slack Channels:**
- `#squad-pix-batch` - Comunicação geral
- `#squad-pix-batch-tech` - Discussões técnicas
- `#squad-pix-batch-incidents` - Alertas e incidents

**GitHub:**
- PRs devem ter descrição detalhada
- Template de PR com checklist
- Reviews obrigatórias (2 aprovações)

**Confluence:**
- ADRs (Architecture Decision Records)
- Postmortems de incidents
- Weekly reports

---

## 11. Rollout Plan

### 11.1 Fases de Rollout

**Fase 1: Internal Alpha (Sprint 7)**
- Ambiente: Staging
- Usuários: Time interno (10 pessoas)
- Objetivo: Smoke testing, validação básica
- Duração: 1 semana

**Fase 2: Beta Privada (Sprint 8)**
- Ambiente: Produção (feature flag)
- Usuários: 5 clientes PJ selecionados
- Objetivo: Validação com dados reais
- Duração: 2 semanas

**Fase 3: Canary Release (Semana 1 pós-Sprint 8)**
- Ambiente: Produção
- Usuários: 10% dos clientes PJ
- Objetivo: Monitorar performance e estabilidade
- Duração: 1 semana

**Fase 4: General Availability (Semana 2 pós-Sprint 8)**
- Ambiente: Produção
- Usuários: 100% dos clientes PJ
- Objetivo: Rollout completo
- Comunicação: Email + blog post + docs

### 11.2 Rollback Plan

**Trigger de Rollback:**
- Taxa de erro > 20%
- p95 latency > 30 min
- Incidents críticos (P0)

**Procedimento:**
1. Desabilitar feature flag
2. Notificar stakeholders
3. Investigar causa raiz
4. Aplicar hotfix
5. Re-deploy após validação

---

## 12. Budget e Recursos

### 12.1 Estimativa de Custo

| Item | Custo Mensal | Duração | Total |
|------|--------------|---------|-------|
| **Squad (5.5 FTEs)** | $50,000 | 4 meses | $200,000 |
| **Infrastructure (staging + prod)** | $2,000 | 4 meses | $8,000 |
| **Tools (Jira, GitHub, Postman, etc)** | $500 | 4 meses | $2,000 |
| **External Services (DICT, SPI)** | $1,000 | 4 meses | $4,000 |
| **Contingência (10%)** | - | - | $21,400 |
| **TOTAL** | - | - | **$235,400** |

### 12.2 ROI Esperado

**Premissas:**
- 1,000 lotes/dia após 3 meses
- Média de 50 items/lote = 50,000 transações/dia
- Fee de $0.10 por transação PIX
- Receita mensal: 50,000 × 30 × $0.10 = **$150,000/mês**
- **Payback period: 1.6 meses**

---

## 13. Próximos Passos Imediatos

### Semana 1 (Antes do Sprint 1)

- [ ] **PO**: Aprovar esta especificação
- [ ] **Tech Lead**: Criar épicos no Jira
- [ ] **Tech Lead**: Criar stories no Jira (Sprint 1)
- [ ] **DevOps**: Provisionar ambientes (staging)
- [ ] **DevOps**: Configurar CI/CD pipelines
- [ ] **Toda Squad**: Kick-off meeting (2h)
  - Apresentação da feature
  - Tour pela especificação
  - Q&A
  - Definition of Ready/Done
  - Sprint 1 planning

### Checklist Pré-Sprint 1

- [ ] Ambientes provisionados
- [ ] Repositório configurado
- [ ] CI/CD funcionando
- [ ] Jira configurado (épicos + stories)
- [ ] Slack channels criados
- [ ] Squad completa alocada
- [ ] Documentação base criada (README)

---

## 14. Contatos e Responsáveis

| Papel | Nome | Email | Slack |
|-------|------|-------|-------|
| **Product Owner** | [TBD] | | @po |
| **Tech Lead** | [TBD] | | @techlead |
| **Backend Engineer 1** | [TBD] | | @backend1 |
| **Backend Engineer 2** | [TBD] | | @backend2 |
| **Backend Engineer 3** | [TBD] | | @backend3 |
| **QA Engineer** | [TBD] | | @qa |
| **DevOps Engineer** | [TBD] | | @devops |
| **Compliance Analyst** | [TBD] | | @compliance |

---

**Documento criado em:** 2025-12-09
**Última atualização:** 2025-12-09
**Status:** Draft - Aguardando Aprovação
