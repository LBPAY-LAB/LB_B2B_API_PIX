# Planejamento: Squad PIX B2B - Pagamentos Unitários

> **Feature:** PIX OUT Unitário para Clientes PJ B2B
> **Duração:** 5 Sprints (10 semanas / 2.5 meses)
> **Data Início:** 2025-01-13
> **Data Fim:** 2025-03-21

---

## 1. Estrutura da Squad

### 1.1 Composição do Time

| Papel | Nome | Responsabilidade | Alocação |
|-------|------|------------------|----------|
| **Product Owner** | [TBD] | Priorização, requisitos de negócio, aceite | 50% |
| **Tech Lead** | [TBD] | Arquitetura, code reviews, decisões técnicas | 100% |
| **Backend Engineer 1** | [TBD] | gRPC service, proto, Payment | 100% |
| **Backend Engineer 2** | [TBD] | REST API, Seamless PIX integration | 100% |
| **Backend Engineer 3** | [TBD] | Worker, DICT, SPI, Ledger integration | 100% |
| **QA Engineer** | [TBD] | Testes automatizados, E2E, performance | 100% |
| **DevOps Engineer** | [TBD] | Infrastructure, CI/CD, monitoring | 50% |
| **Compliance Analyst** | [TBD] | Validação regulatória, auditoria | 25% |

**Total:** 4.75 FTEs

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
- **Slack**: Comunicação do time (#squad-pix-b2b)
- **Confluence**: Documentação técnica
- **Postman**: API testing e collections
- **Grafana**: Monitoring e dashboards

---

## 3. Épicos e User Stories

### Épico 1: Fundação e Infraestrutura
**Objetivo:** Criar a base técnica para pagamentos PIX unitários

#### User Stories:

**E1-US1: Definir Proto para PIX Payment B2B Service**
- **Como** desenvolvedor backend
- **Quero** ter as definições proto completas
- **Para que** eu possa gerar os stubs gRPC

**Critérios de Aceite:**
- [ ] Proto `pix_payment_b2b.proto` criado
- [ ] 4 RPCs definidos (Initiate, Get, List, Cancel)
- [ ] Enums para status e pix_key_type
- [ ] Mensagens com validação via protoc
- [ ] Documentação inline nos protos

**Story Points:** 3
**Sprint:** Sprint 1

---

**E1-US2: Criar Schema do Banco de Dados**
- **Como** desenvolvedor backend
- **Quero** ter a tabela criada
- **Para que** eu possa persistir pagamentos

**Critérios de Aceite:**
- [ ] Tabela `pix_payments_b2b` criada
- [ ] Índices de performance criados
- [ ] Constraints e validações
- [ ] Migration scripts (up/down)
- [ ] Testes de migration

**Story Points:** 2
**Sprint:** Sprint 1

---

**E1-US3: Criar Skeleton do gRPC Service**
- **Como** desenvolvedor backend
- **Quero** ter a estrutura do serviço gRPC
- **Para que** eu possa implementar as operações

**Critérios de Aceite:**
- [ ] Service registrado no Payment app
- [ ] 4 métodos vazios implementados
- [ ] Testes unitários básicos
- [ ] Logging configurado
- [ ] Health check incluído

**Story Points:** 3
**Sprint:** Sprint 1

---

**E1-US4: Criar Endpoints REST no Seamless PIX**
- **Como** desenvolvedor backend
- **Quero** ter os 4 endpoints REST expostos
- **Para que** clientes possam consumir a API

**Critérios de Aceite:**
- [ ] POST /v1/pix/payments
- [ ] GET /v1/pix/payments/{payment_id}
- [ ] GET /v1/pix/payments (listar)
- [ ] DELETE /v1/pix/payments/{payment_id}
- [ ] Handlers criados (vazios ok)
- [ ] OpenAPI spec atualizado
- [ ] Postman collection

**Story Points:** 5
**Sprint:** Sprint 1

---

**E1-US5: Configurar Message Queue**
- **Como** DevOps
- **Quero** ter o tópico Pulsar configurado
- **Para que** pagamentos sejam processados assincronamente

**Critérios de Aceite:**
- [ ] Tópico `pix.payments.b2b` criado
- [ ] Retention policy (7 dias)
- [ ] Dead letter queue
- [ ] Producer e Consumer testados
- [ ] Monitoring (Grafana)

**Story Points:** 2
**Sprint:** Sprint 1

---

### Épico 2: Core Business Logic
**Objetivo:** Implementar criação e validação de pagamentos

#### User Stories:

**E2-US1: Implementar Criação de Pagamento (gRPC)**
- **Como** desenvolvedor backend
- **Quero** implementar `InitiatePixPayment`
- **Para que** pagamentos sejam criados e validados

**Critérios de Aceite:**
- [ ] Validação de schema
- [ ] Validação de saldo disponível
- [ ] Persistência em PostgreSQL
- [ ] Publicação de evento `pix.payment.initiated`
- [ ] Idempotência via `idempotency_key`
- [ ] Testes unitários (90% coverage)
- [ ] Testes de integração

**Story Points:** 8
**Sprint:** Sprint 2

---

**E2-US2: Implementar Endpoint REST de Criação**
- **Como** desenvolvedor backend
- **Quero** conectar REST API ao gRPC
- **Para que** clientes possam criar pagamentos via HTTP

**Critérios de Aceite:**
- [ ] Handler `POST /v1/pix/payments` completo
- [ ] Mappers (Domain DTO → gRPC → Proto)
- [ ] Validação de headers
- [ ] Response 202 Accepted
- [ ] Location header
- [ ] Error handling (400, 401, 403, 422, 429)
- [ ] Testes E2E

**Story Points:** 5
**Sprint:** Sprint 2

---

**E2-US3: Implementar Validações de PIX Key**
- **Como** desenvolvedor backend
- **Quero** validar formato de chaves PIX
- **Para que** apenas chaves válidas sejam aceitas

**Critérios de Aceite:**
- [ ] Validação de CPF (11 dígitos + check digit)
- [ ] Validação de CNPJ (14 dígitos + check digit)
- [ ] Validação de Email (RFC 5322)
- [ ] Validação de Telefone (+55DDNNNNNNNNN)
- [ ] Validação de EVP (UUID v4)
- [ ] Testes unitários
- [ ] Error messages claros

**Story Points:** 3
**Sprint:** Sprint 2

---

**E2-US4: Implementar Consulta de Pagamento**
- **Como** desenvolvedor backend
- **Quero** implementar consulta
- **Para que** clientes vejam status

**Critérios de Aceite:**
- [ ] gRPC `GetPixPayment` implementado
- [ ] REST `GET /v1/pix/payments/{payment_id}`
- [ ] Retorna todos os campos (status, e2e_id, etc)
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 5
**Sprint:** Sprint 2

---

### Épico 3: Processamento de Pagamentos
**Objetivo:** Processar pagamentos de forma assíncrona

#### User Stories:

**E3-US1: Criar Worker para Consumo de Eventos**
- **Como** desenvolvedor backend
- **Quero** ter um worker que processe pagamentos
- **Para que** pagamentos sejam executados

**Critérios de Aceite:**
- [ ] Worker consome `pix.payments.b2b` topic
- [ ] Processa evento `pix.payment.initiated`
- [ ] Atualiza status para `processing`
- [ ] Publica evento `pix.payment.processing`
- [ ] Health check endpoint
- [ ] Graceful shutdown
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 3

---

**E3-US2: Implementar Validação de Chave PIX via DICT**
- **Como** desenvolvedor backend
- **Quero** validar chaves no DICT
- **Para que** pagamentos para chaves inválidas sejam rejeitados

**Critérios de Aceite:**
- [ ] Cliente gRPC para DICT service
- [ ] Consulta de chave PIX
- [ ] Cache de resultados (5 min TTL)
- [ ] Tratamento de erro (chave não encontrada)
- [ ] Timeout configurável
- [ ] Retry com backoff exponencial
- [ ] Testes com mock do DICT

**Story Points:** 8
**Sprint:** Sprint 3

---

**E3-US3: Implementar Validação de Mesma Titularidade**
- **Como** desenvolvedor backend
- **Quero** validar CPF/CNPJ
- **Para que** conformidade seja garantida

**Critérios de Aceite:**
- [ ] Extração de CPF/CNPJ do DICT response
- [ ] Comparação com `payer_info.document`
- [ ] Marca como `failed` se obrigatório e não bater
- [ ] Log de auditoria
- [ ] Testes unitários
- [ ] Testes de integração

**Story Points:** 3
**Sprint:** Sprint 3

---

**E3-US4: Implementar Reserva de Saldo**
- **Como** desenvolvedor backend
- **Quero** reservar saldo no Ledger v2
- **Para que** pagamento tenha saldo garantido

**Critérios de Aceite:**
- [ ] Chamada gRPC `PIX_OUT_PENDING`
- [ ] Armazenamento de reservation_id
- [ ] Rollback se falhar
- [ ] Confirmação após sucesso (`PIX_OUT_CONFIRM`)
- [ ] Tratamento de saldo insuficiente
- [ ] Testes com mock do Ledger

**Story Points:** 8
**Sprint:** Sprint 3

---

**E3-US5: Implementar Envio para SPI/BACEN**
- **Como** desenvolvedor backend
- **Quero** enviar pagamento para SPI
- **Para que** PIX seja executado

**Critérios de Aceite:**
- [ ] Cliente para SPI service
- [ ] Envio de mensagem PACS.008
- [ ] Recepção de PACS.002 (ACK)
- [ ] Geração de E2E ID conforme spec BACEN
- [ ] Armazenamento de e2e_id e txid
- [ ] Timeout e retry
- [ ] Testes com mock do SPI

**Story Points:** 13
**Sprint:** Sprint 3-4

---

**E3-US6: Implementar Finalização de Pagamento**
- **Como** desenvolvedor backend
- **Quero** finalizar pagamento após sucesso/falha
- **Para que** status seja atualizado

**Critérios de Aceite:**
- [ ] Atualização de status para `completed` ou `failed`
- [ ] Atualização de `completed_at`
- [ ] Confirmação no Ledger (`PIX_OUT_CONFIRM`)
- [ ] Publicação de evento `pix.payment.completed`
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 4

---

### Épico 4: Webhooks e Listagem
**Objetivo:** Notificar clientes e permitir consultas

#### User Stories:

**E4-US1: Implementar Publicação de Eventos**
- **Como** desenvolvedor backend
- **Quero** publicar eventos no Pulsar
- **Para que** webhook service possa consumir

**Critérios de Aceite:**
- [ ] Publisher para tópico `pix.payment.events`
- [ ] 6 tipos de eventos
- [ ] Schema Avro ou Protobuf
- [ ] Retry em caso de falha
- [ ] Testes unitários

**Story Points:** 3
**Sprint:** Sprint 4

---

**E4-US2: Implementar Webhook Delivery**
- **Como** desenvolvedor backend
- **Quero** enviar webhooks para `callback_url`
- **Para que** cliente seja notificado

**Critérios de Aceite:**
- [ ] Worker consome `pix.payment.events` topic
- [ ] HTTP POST para `callback_url`
- [ ] HMAC-SHA256 signature
- [ ] Timeout de 10 segundos
- [ ] Retry com backoff exponencial (5 tentativas)
- [ ] Dead letter queue
- [ ] Testes com mock HTTP server

**Story Points:** 8
**Sprint:** Sprint 4

---

**E4-US3: Implementar Listagem de Pagamentos**
- **Como** desenvolvedor backend
- **Quero** listar pagamentos
- **Para que** clientes vejam histórico

**Critérios de Aceite:**
- [ ] gRPC `ListPixPayments` implementado
- [ ] REST `GET /v1/pix/payments`
- [ ] Filtros: account_id, status, start_date, end_date
- [ ] Paginação cursor-based
- [ ] Ordenação por created_at DESC
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 5
**Sprint:** Sprint 4

---

### Épico 5: Finalização e Produção
**Objetivo:** Completar feature e preparar para produção

#### User Stories:

**E5-US1: Implementar Cancelamento**
- **Como** cliente PJ
- **Quero** cancelar um pagamento pendente
- **Para que** valor não seja debitado

**Critérios de Aceite:**
- [ ] gRPC `CancelPixPayment` implementado
- [ ] REST `DELETE /v1/pix/payments/{payment_id}`
- [ ] Cancela apenas status `pending` ou `validating`
- [ ] Libera reserva de saldo
- [ ] Publicação de evento `pix.payment.cancelled`
- [ ] Testes unitários
- [ ] Testes E2E

**Story Points:** 5
**Sprint:** Sprint 5

---

**E5-US2: Implementar Circuit Breaker**
- **Como** desenvolvedor backend
- **Quero** ter circuit breaker para SPI
- **Para que** falhas em cascata sejam evitadas

**Critérios de Aceite:**
- [ ] Circuit breaker (closed, open, half-open)
- [ ] Threshold: 50% erro em 1 minuto
- [ ] Timeout: 30 segundos
- [ ] Retry após 1 minuto
- [ ] Métricas expostas
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 5

---

**E5-US3: Implementar Rate Limiting**
- **Como** Tech Lead
- **Quero** limitar pagamentos por cliente
- **Para que** recursos sejam balanceados

**Critérios de Aceite:**
- [ ] Máximo 100 pagamentos simultâneos por account_id
- [ ] Máximo 1,000 pagamentos pendentes por account_id
- [ ] Response 429 se limite excedido
- [ ] Header `X-RateLimit-*`
- [ ] Testes unitários

**Story Points:** 3
**Sprint:** Sprint 5

---

**E5-US4: Implementar Caching DICT**
- **Como** desenvolvedor backend
- **Quero** cachear resultados do DICT
- **Para que** performance seja melhorada

**Critérios de Aceite:**
- [ ] Redis cache com TTL de 5 minutos
- [ ] Cache key: `dict:{pix_key}`
- [ ] Cache miss → consulta DICT
- [ ] Cache hit → retorna direto
- [ ] Invalidação manual (admin endpoint)
- [ ] Métricas de hit rate
- [ ] Testes unitários

**Story Points:** 5
**Sprint:** Sprint 5

---

**E5-US5: Load Testing**
- **Como** QA Engineer
- **Quero** executar testes de carga
- **Para que** capacidade seja validada

**Critérios de Aceite:**
- [ ] Script k6 ou Gatling
- [ ] Cenário: 10,000 pagamentos em 1 hora
- [ ] Target: 14 TPS sustentado
- [ ] p95 latency < 2s
- [ ] Report com gráficos

**Story Points:** 8
**Sprint:** Sprint 5

---

**E5-US6: Criar Dashboard e Alertas**
- **Como** DevOps
- **Quero** ter dashboard e alertas
- **Para que** operações sejam monitoradas

**Critérios de Aceite:**
- [ ] Dashboard "PIX Payments B2B"
- [ ] Métricas: throughput, latency, error rate
- [ ] Alertas: taxa de erro, latência, fila
- [ ] Notificação via Slack + PagerDuty
- [ ] Runbook linkado

**Story Points:** 5
**Sprint:** Sprint 5

---

**E5-US7: Documentação Completa**
- **Como** Tech Writer
- **Quero** documentação para clientes
- **Para que** integração seja facilitada

**Critérios de Aceite:**
- [ ] README.md atualizado
- [ ] API Reference (OpenAPI)
- [ ] Integration Guide
- [ ] Troubleshooting Guide
- [ ] Postman Collection
- [ ] Exemplos de código (cURL, Python)

**Story Points:** 5
**Sprint:** Sprint 5

---

## 4. Backlog Priorizado

### Sprint 1 (Jan 13 - Jan 24)
**Objetivo:** Fundação técnica

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E1-US1 | Definir Proto | 3 | Backend 1 |
| E1-US2 | Criar Schema DB | 2 | Backend 1 |
| E1-US3 | Skeleton gRPC Service | 3 | Backend 1 |
| E1-US4 | Endpoints REST | 5 | Backend 2 |
| E1-US5 | Configurar Message Queue | 2 | DevOps |

**Total:** 15 pontos
**Velocity Esperada:** 15-20 pontos

---

### Sprint 2 (Jan 27 - Feb 07)
**Objetivo:** Core business logic - criação

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E2-US1 | Criar Pagamento gRPC | 8 | Backend 1 |
| E2-US2 | Endpoint REST Criação | 5 | Backend 2 |
| E2-US3 | Validações PIX Key | 3 | Backend 3 |
| E2-US4 | Consulta de Pagamento | 5 | Backend 2 |

**Total:** 21 pontos

---

### Sprint 3 (Feb 10 - Feb 21)
**Objetivo:** Processamento assíncrono

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E3-US1 | Worker Consumo | 5 | Backend 3 |
| E3-US2 | Validação DICT | 8 | Backend 3 |
| E3-US3 | Validação Titularidade | 3 | Backend 3 |
| E3-US4 | Reserva Saldo | 8 | Backend 1 |

**Total:** 24 pontos

---

### Sprint 4 (Feb 24 - Mar 07)
**Objetivo:** Completar processamento e webhooks

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E3-US5 | Envio SPI (parte 1) | 8 | Backend 3 |
| E3-US6 | Finalização Pagamento | 5 | Backend 3 |
| E4-US1 | Publicação Eventos | 3 | Backend 1 |
| E4-US2 | Webhook Delivery | 8 | Backend 2 |

**Total:** 24 pontos

---

### Sprint 5 (Mar 10 - Mar 21)
**Objetivo:** Finalização e produção

| ID | Story | Pontos | Responsável |
|----|-------|--------|-------------|
| E4-US3 | Listagem Pagamentos | 5 | Backend 2 |
| E5-US1 | Cancelamento | 5 | Backend 2 |
| E5-US2 | Circuit Breaker | 5 | Backend 1 |
| E5-US3 | Rate Limiting | 3 | Backend 2 |
| E5-US4 | Caching DICT | 5 | Backend 3 |
| E5-US5 | Load Testing | 8 | QA |
| E5-US6 | Dashboard e Alertas | 5 | DevOps |
| E5-US7 | Documentação | 5 | Backend 1 + 2 |

**Total:** 41 pontos (sprint pesada, considerar redução ou extensão)

---

## 5. Kanban Board Structure

### 5.1 Colunas

```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│  Backlog     │   To Do      │  In Progress │  In Review   │   Testing    │     Done     │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Stories      │ Stories      │ Stories      │ PRs          │ Test Cases   │ Deployed     │
│ priorizadas  │ sprint atual │ WIP limit: 4 │ Code Review  │ E2E Testing  │ Validated    │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

### 5.2 WIP Limits

| Coluna | WIP Limit | Motivo |
|--------|-----------|--------|
| To Do | ∞ | Priorização livre |
| In Progress | 4 | Foco da squad (3 backend + 1 QA) |
| In Review | 8 | Tempo de code review |
| Testing | 6 | Tempo de execução de testes |
| Done | ∞ | Histórico |

---

## 6. Definition of Ready (DoR)

Uma story está pronta para ser iniciada quando:

- [ ] Critérios de aceite claramente definidos
- [ ] Dependências identificadas e resolvidas
- [ ] Design técnico aprovado (se aplicável)
- [ ] Estimativa de story points
- [ ] Prioridade definida pelo PO
- [ ] Testável

---

## 7. Definition of Done (DoD)

Uma story está completa quando:

- [ ] Código implementado conforme critérios
- [ ] Code review aprovado (2 aprovações)
- [ ] Testes unitários (>80% coverage)
- [ ] Testes de integração (se aplicável)
- [ ] Testes E2E passando
- [ ] Documentação atualizada
- [ ] Deployed em staging
- [ ] PO aceitou a story
- [ ] Sem bugs críticos

---

## 8. Riscos e Mitigações

### 8.1 Riscos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **DICT rate limiting** | Média | Alto | Cache agressivo (5 min TTL) |
| **SPI downtime** | Média | Alto | Circuit breaker + retry |
| **Performance insuficiente** | Baixa | Médio | Load testing cedo |
| **Validação titularidade complexa** | Baixa | Médio | Spec clara + testes |

### 8.2 Riscos de Prazo

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **Sprint 5 sobrecarregada** | Alta | Médio | Reduzir scope ou adicionar sprint 6 |
| **Code reviews lentas** | Média | Médio | SLA de 24h |
| **Bugs em integração** | Média | Alto | Testes rigorosos + staging |

---

## 9. Métricas de Sucesso

### 9.1 Métricas de Engenharia

| Métrica | Target | Medição |
|---------|--------|---------|
| **Velocity** | 18-24 pontos/sprint | Jira |
| **Code Coverage** | >80% | SonarQube |
| **Code Review Time** | <24h | GitHub |
| **Deployment Frequency** | 2x/semana | CI/CD |
| **Lead Time** | <5 dias | Jira |

### 9.2 Métricas de Produto

| Métrica | Target (2 meses após launch) | Medição |
|---------|------------------------------|---------|
| **Pagamentos/dia** | 5,000 | Analytics |
| **Taxa de sucesso** | >95% | Analytics |
| **Tempo médio processamento** | <2s | Logs |
| **Adoção clientes PJ** | 20% | CRM |

---

## 10. Rollout Plan

### Fase 1: Internal Alpha (Sprint 4)
- Ambiente: Staging
- Usuários: Time interno
- Duração: 1 semana

### Fase 2: Beta Privada (Sprint 5)
- Ambiente: Produção (feature flag)
- Usuários: 3 clientes PJ selecionados
- Duração: 1 semana

### Fase 3: Canary Release (Pós-Sprint 5, Semana 1)
- Ambiente: Produção
- Usuários: 10% dos clientes PJ
- Duração: 1 semana

### Fase 4: General Availability (Semana 2)
- Ambiente: Produção
- Usuários: 100% dos clientes PJ
- Comunicação: Email + docs

---

## 11. Budget e Recursos

### 11.1 Estimativa de Custo

| Item | Custo Mensal | Duração | Total |
|------|--------------|---------|-------|
| **Squad (4.75 FTEs)** | $40,000 | 2.5 meses | $100,000 |
| **Infrastructure** | $1,500 | 2.5 meses | $3,750 |
| **Tools** | $500 | 2.5 meses | $1,250 |
| **External Services** | $1,000 | 2.5 meses | $2,500 |
| **Contingência (10%)** | - | - | $10,750 |
| **TOTAL** | - | - | **$118,250** |

### 11.2 ROI Esperado

**Premissas:**
- 5,000 pagamentos/dia após 2 meses
- Fee de $0.15 por transação
- Receita mensal: 5,000 × 30 × $0.15 = **$22,500/mês**
- **Payback period: 5.3 meses**

---

## 12. Comparação: Lote vs Unitário

| Aspecto | Lote (Original) | Unitário (Atual) |
|---------|-----------------|------------------|
| **Duração** | 8 sprints (4 meses) | 5 sprints (2.5 meses) | ✅ **-37% tempo** |
| **Complexidade** | Alta | Baixa | ✅ **Menor risco** |
| **Squad Size** | 5.5 FTEs | 4.75 FTEs | ✅ **-14% custo** |
| **Budget** | $235,400 | $118,250 | ✅ **-50% custo** |
| **Tabelas DB** | 2 | 1 | ✅ **Mais simples** |
| **Endpoints** | 5 | 4 | ✅ **Mais simples** |
| **Worker** | Paralelo complexo | Simples 1:1 | ✅ **Mais simples** |
| **Time to Market** | 4 meses | 2.5 meses | ✅ **Mais rápido** |

---

## 13. Próximos Passos Imediatos

### Semana 1 (Antes do Sprint 1)

- [ ] **PO**: Aprovar especificação
- [ ] **Tech Lead**: Criar épicos no Jira
- [ ] **Tech Lead**: Criar stories Sprint 1
- [ ] **DevOps**: Provisionar staging
- [ ] **DevOps**: Configurar CI/CD
- [ ] **Squad**: Kick-off meeting (2h)

---

## 14. Contatos

| Papel | Nome | Email | Slack |
|-------|------|-------|-------|
| **Product Owner** | [TBD] | | @po |
| **Tech Lead** | [TBD] | | @techlead |
| **Backend Engineer 1** | [TBD] | | @backend1 |
| **Backend Engineer 2** | [TBD] | | @backend2 |
| **Backend Engineer 3** | [TBD] | | @backend3 |
| **QA Engineer** | [TBD] | | @qa |
| **DevOps Engineer** | [TBD] | | @devops |

---

**Documento criado em:** 2025-12-09
**Última atualização:** 2025-12-09
**Status:** Draft - Aguardando Aprovação
