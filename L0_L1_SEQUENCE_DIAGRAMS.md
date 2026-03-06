# FinTech Enterprise Platform — L0/L1 Overall Sequence Diagrams

> **Type:** Single Source of Truth — Customer Journey End-to-End Flow Sequences
> **Scope:** L0/L1 customer journeys · Experience API orchestration · Choreography and Orchestration Sagas · Strangler Fig traffic migration
> **Platform:** Digital Banking and Wealth Platform
> **Stack:** React 18 · Webpack Module Federation · Java 21 · Spring Boot 3.3 · Apache Kafka · PostgreSQL 16
> **Regulatory:** PCI-DSS Level 1 · SOC 2 Type II · PSD2/Open Banking · MiFID II
> **Perspective:** FinTech Principal Architects · Software and Quality Engineers

---

## Quick Navigation — All Architecture References

| Layer | Document | Scope |
|---|---|---|
| **L0/L1 Architecture** | [L0_L1_ARCHITECTURE.md](./L0_L1_ARCHITECTURE.md) | L0/L1 diagrams · Experience API · Saga patterns · Strangler Fig blueprint |
| **Front-End Architecture** | [ARCHITECTURE.md](./ARCHITECTURE.md) | MFE topology · Module Federation · Design System · Auth · Feature Flags |
| **Front-End Sequences** | [SEQUENCE_DIAGRAMS.md](./SEQUENCE_DIAGRAMS.md) | PKCE auth · MFE lazy load · PCI-DSS boundary · audit trail · token refresh |
| **Back-End Architecture** | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | API Gateway · 6 domain microservices · Kafka · Data Layer · Security · ORM · ADRs |
| **Back-End Sequences** | [BACKEND_SEQUENCE_DIAGRAMS.md](./BACKEND_SEQUENCE_DIAGRAMS.md) | Payment Saga · KYC/AML flows · trading · notification · auth token lifecycle |

---

## Diagram Index

| # | Flow | Saga Type | Domains Covered |
|---|---|---|---|
| 1 | [Account Overview — Parallel Aggregation](#flow-1-account-overview--parallel-aggregation) | Synchronous parallel | account-service + trading-service |
| 2 | [Payment Initiation — Choreography Saga](#flow-2-payment-initiation--choreography-saga) | Kafka choreography | payment + compliance + account + notification |
| 3 | [Trade Order Placement — MiFID II](#flow-3-trade-order-placement--mifid-ii) | Sync + async Kafka | account + trading + compliance + notification |
| 4 | [KYC Onboarding — Orchestration Saga](#flow-4-kyc-onboarding--orchestration-saga) | Saga orchestrator | compliance + account + notification |
| 5 | [Strangler Fig — Traffic Cutover](#flow-5-strangler-fig--traffic-cutover) | Infrastructure routing | API Gateway + Legacy Monolith + new microservices |
| 6 | [Saga Compensation — Payment Rollback](#flow-6-saga-compensation--payment-rollback) | Compensating transaction | payment + account + notification + audit |

---

## Flow 1: Account Overview — Parallel Aggregation

> **Journey:** Customer navigates to the Dashboard. The Dashboard MFE calls one Experience API endpoint. The Account Overview Journey API calls two domain services in parallel and returns a single composite response optimised for the Dashboard screen.
>
> **Pattern:** Synchronous parallel aggregation — Promise.all equivalent at the Experience API layer.
>
> → Detailed front-end context: [ARCHITECTURE.md §5.1](./ARCHITECTURE.md) · [BACKEND_ARCHITECTURE.md §3.1](./BACKEND_ARCHITECTURE.md)

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer
    participant Shell as Shell MFE
    participant DashMFE as Dashboard MFE
    participant GW as API Gateway
    participant ExpAcct as Account Overview Journey API
    participant AcctSvc as account-service
    participant TradeSvc as trading-service

    Customer->>Shell: Navigate to /dashboard
    Shell->>DashMFE: React.lazy() — load Dashboard MFE
    activate DashMFE

    DashMFE->>GW: GET /experience/v1/accounts/overview\nAuthorization: Bearer JWT
    activate GW
    GW->>GW: Validate RS256 JWT signature and claims
    GW->>ExpAcct: Route to Account Overview Journey API
    activate ExpAcct

    Note over ExpAcct: Journey API calls both services in parallel — single composite response

    par Call account-service
        ExpAcct->>AcctSvc: GET /api/accounts?customerId={id}
        activate AcctSvc
        AcctSvc-->>ExpAcct: accounts[], balance, recent transactions
        deactivate AcctSvc
    and Call trading-service
        ExpAcct->>TradeSvc: GET /api/portfolio/summary?customerId={id}
        activate TradeSvc
        TradeSvc-->>ExpAcct: portfolio holdings, total value, daily PnL
        deactivate TradeSvc
    end

    ExpAcct->>ExpAcct: Aggregate results into journey-optimised DTO
    ExpAcct-->>GW: 200 OK — AccountOverviewDTO
    deactivate ExpAcct
    GW-->>DashMFE: 200 OK — AccountOverviewDTO
    deactivate GW

    DashMFE->>DashMFE: Render AccountSummaryCard and PortfolioWidget components
    DashMFE-->>Customer: Dashboard rendered with live balances and portfolio

    deactivate DashMFE
```

---

## Flow 2: Payment Initiation — Choreography Saga

> **Journey:** Customer initiates a payment in the Payments MFE. The Payment Journey API coordinates a balance check synchronously, then creates the payment and triggers an asynchronous Kafka choreography saga across compliance, account, and notification services.
>
> **Pattern:** Kafka Choreography Saga — each service independently subscribes to events and publishes the next event in the chain. The Experience API returns 202 Accepted immediately. The MFE polls for final status.
>
> → Detailed back-end context: [BACKEND_ARCHITECTURE.md §3.2](./BACKEND_ARCHITECTURE.md) · [BACKEND_SEQUENCE_DIAGRAMS.md](./BACKEND_SEQUENCE_DIAGRAMS.md)

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer
    participant PayMFE as Payments MFE
    participant GW as API Gateway
    participant ExpPay as Payment Journey API
    participant AcctSvc as account-service
    participant PaySvc as payment-service
    participant Kafka as Apache Kafka
    participant CompSvc as compliance-service
    participant NotifSvc as notification-service

    Customer->>PayMFE: Fill payment form and click Send
    PayMFE->>GW: POST /experience/v1/payments/initiate\nIdempotency-Key: IDEM-001\nBearer JWT
    activate GW
    GW->>GW: Validate JWT — check scope payments:write
    GW->>ExpPay: Route to Payment Journey API
    activate ExpPay

    ExpPay->>AcctSvc: GET /api/accounts/{id}/balance — sync funding check
    activate AcctSvc
    AcctSvc-->>ExpPay: balance GBP 2500 — sufficient for GBP 500 payment
    deactivate AcctSvc

    ExpPay->>PaySvc: POST /api/payments — idempotencyKey=IDEM-001
    activate PaySvc
    PaySvc->>PaySvc: Idempotency check — new request
    PaySvc->>PaySvc: Tokenise card via Vault Transit Encryption
    PaySvc->>PaySvc: Save Payment — status CREATED — @Transactional
    PaySvc->>PaySvc: Save OutboxEvent row in same transaction

    PaySvc-->>ExpPay: 202 Accepted — paymentId=PAY-001 status=FRAUD_CHECK
    deactivate PaySvc

    ExpPay-->>GW: 202 Accepted — paymentId=PAY-001
    deactivate ExpPay
    GW-->>PayMFE: 202 Accepted — paymentId=PAY-001
    deactivate GW

    PayMFE-->>Customer: Show processing spinner with paymentId

    Note over PaySvc,Kafka: OutboxRelay polls outbox — publishes event asynchronously

    PaySvc->>Kafka: Publish payment.initiated — paymentId=PAY-001

    Note over Kafka,CompSvc: Choreography Saga Step 1 — compliance AML check

    Kafka->>CompSvc: Consume payment.initiated
    activate CompSvc
    CompSvc->>CompSvc: Run AML rule engine — check transaction patterns
    CompSvc->>CompSvc: Check PEP and sanctions screening

    alt AML check CLEAR
        CompSvc->>Kafka: Publish kyc.passed — paymentId=PAY-001
        deactivate CompSvc

        Note over Kafka,PaySvc: Choreography Saga Step 2 — payment authorisation

        Kafka->>PaySvc: Consume kyc.passed
        activate PaySvc
        PaySvc->>PaySvc: Transition Payment to AUTHORISED
        PaySvc->>Kafka: Publish payment.completed — paymentId=PAY-001
        deactivate PaySvc

        Note over Kafka,AcctSvc: Choreography Saga Step 3 — parallel debit and notification

        par Debit account balance
            Kafka->>AcctSvc: Consume payment.completed
            activate AcctSvc
            AcctSvc->>AcctSvc: Debit account — GBP 500
            AcctSvc->>AcctSvc: Create Transaction record
            deactivate AcctSvc
        and Send confirmation notification
            Kafka->>NotifSvc: Consume payment.completed
            activate NotifSvc
            NotifSvc->>NotifSvc: Render payment confirmation template
            NotifSvc->>NotifSvc: Send SMS and email to customer
            deactivate NotifSvc
        end

    else AML check FLAGGED
        CompSvc->>Kafka: Publish kyc.flagged — paymentId=PAY-001
        deactivate CompSvc
        Kafka->>PaySvc: Consume kyc.flagged
        activate PaySvc
        PaySvc->>PaySvc: Transition Payment to REJECTED
        PaySvc->>Kafka: Publish payment.failed — paymentId=PAY-001
        deactivate PaySvc
        Kafka->>NotifSvc: Consume payment.failed — send rejection alert
    end

    PayMFE->>GW: GET /experience/v1/payments/PAY-001/status — status polling
    GW-->>PayMFE: 200 OK — status=AUTHORISED or status=REJECTED
    PayMFE-->>Customer: Show payment result screen
```

---

## Flow 3: Trade Order Placement — MiFID II

> **Journey:** Customer places a trade order in the Trading MFE. The Trading Journey API synchronously checks available funding, then places the order with the trading-service. The execution and MiFID II regulatory reporting happen asynchronously via Kafka.
>
> **Pattern:** Synchronous orchestration for the critical path (funding check + order placement) combined with async Kafka events for reporting and notification.
>
> → Detailed context: [BACKEND_ARCHITECTURE.md §3.3](./BACKEND_ARCHITECTURE.md) · [BACKEND_SEQUENCE_DIAGRAMS.md](./BACKEND_SEQUENCE_DIAGRAMS.md)

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer
    participant TradeMFE as Trading MFE
    participant GW as API Gateway
    participant ExpTrade as Trading Journey API
    participant AcctSvc as account-service
    participant TradeSvc as trading-service
    participant Kafka as Apache Kafka
    participant CompSvc as compliance-service
    participant NotifSvc as notification-service

    Customer->>TradeMFE: Place order BUY 100 shares AAPL at market price
    TradeMFE->>GW: POST /experience/v1/trading/orders\nBearer JWT — scope trading:write
    activate GW
    GW->>GW: Validate JWT and MiFID II scope
    GW->>ExpTrade: Route to Trading Journey API
    activate ExpTrade

    ExpTrade->>AcctSvc: GET /api/accounts/{id}/balance — funding check
    activate AcctSvc
    AcctSvc-->>ExpTrade: available balance GBP 15000 — sufficient for estimated order
    deactivate AcctSvc

    ExpTrade->>TradeSvc: POST /api/orders — instrument=AAPL side=BUY quantity=100
    activate TradeSvc
    TradeSvc->>TradeSvc: Validate order parameters and instrument eligibility
    TradeSvc->>TradeSvc: Save Order — status NEW — @Version optimistic lock
    TradeSvc->>Kafka: Publish trade.order.placed — orderId=ORD-001

    TradeSvc-->>ExpTrade: 201 Created — orderId=ORD-001 status=NEW
    deactivate TradeSvc

    ExpTrade-->>GW: 201 Created — orderId=ORD-001
    deactivate ExpTrade
    GW-->>TradeMFE: 201 Created — orderId=ORD-001
    deactivate GW

    TradeMFE-->>Customer: Order acknowledged — show pending status

    Note over Kafka,CompSvc: Async Saga Step 1 — MiFID II regulatory reporting

    Kafka->>CompSvc: Consume trade.order.placed
    activate CompSvc
    CompSvc->>CompSvc: Generate RTS 22 MiFID II transaction report
    CompSvc->>CompSvc: Submit report to FCA regulatory body
    CompSvc->>CompSvc: Save MifidReport record — status SUBMITTED
    deactivate CompSvc

    Note over TradeSvc,Kafka: Order matched and executed at venue

    TradeSvc->>Kafka: Publish trade.executed — orderId=ORD-001 price=178.50 qty=100

    par Update portfolio
        Kafka->>TradeSvc: Consume trade.executed
        activate TradeSvc
        TradeSvc->>TradeSvc: Save Execution record — immutable
        TradeSvc->>TradeSvc: Update Portfolio — VWAP average cost
        deactivate TradeSvc
    and Send trade confirmation
        Kafka->>NotifSvc: Consume trade.executed
        activate NotifSvc
        NotifSvc->>NotifSvc: Render trade confirmation template
        NotifSvc->>NotifSvc: Send email with execution details and venue
        deactivate NotifSvc
    end

    TradeMFE->>GW: GET /experience/v1/trading/orders/ORD-001 — status poll
    GW-->>TradeMFE: 200 OK — status=FILLED executedPrice=178.50
    TradeMFE-->>Customer: Show filled order confirmation with execution details
```

---

## Flow 4: KYC Onboarding — Orchestration Saga

> **Journey:** New customer starts KYC onboarding in the Compliance MFE. The KYC Journey API delegates to a central `KycOnboardingSagaOrchestrator` that manages the state machine and calls each step sequentially. Each step has an explicit compensation action if it fails.
>
> **Pattern:** Orchestration Saga — centralised state machine with sequential steps and controlled compensation.
>
> → Detailed context: [BACKEND_ARCHITECTURE.md §3.4](./BACKEND_ARCHITECTURE.md)

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer
    participant CompMFE as Compliance MFE
    participant GW as API Gateway
    participant ExpKyc as KYC Journey API
    participant Orch as KycSagaOrchestrator
    participant CompSvc as compliance-service
    participant AcctSvc as account-service
    participant NotifSvc as notification-service

    Customer->>CompMFE: Start KYC — upload identity documents
    CompMFE->>GW: POST /experience/v1/kyc/onboard\nBearer JWT — customerId, documentRefs
    activate GW
    GW->>GW: Validate JWT
    GW->>ExpKyc: Route to KYC Journey API
    activate ExpKyc

    ExpKyc->>Orch: Start new KYC saga — customerId, documentRefs
    activate Orch
    Orch->>Orch: Create saga record — state STARTED

    Note over Orch,CompSvc: Step 1 — Document Verification

    Orch->>CompSvc: POST /api/kyc/verify-documents — OCR scan and identity match
    activate CompSvc
    CompSvc->>CompSvc: OCR extract document fields
    CompSvc->>CompSvc: Match name and date-of-birth against submitted data
    CompSvc-->>Orch: DOCS_VERIFIED or DOCS_REJECTED
    deactivate CompSvc

    Orch->>Orch: Update saga state to DOCS_VERIFIED

    Note over Orch,CompSvc: Step 2 — Biometric Check

    Orch->>CompSvc: POST /api/kyc/biometric — face match and liveness check
    activate CompSvc
    CompSvc->>CompSvc: Run face recognition against document photo
    CompSvc->>CompSvc: Liveness detection — anti-spoofing check
    CompSvc-->>Orch: BIOMETRIC_PASSED or BIOMETRIC_FAILED
    deactivate CompSvc

    Orch->>Orch: Update saga state to BIOMETRIC_PASSED

    Note over Orch,CompSvc: Step 3 — PEP and Sanctions Screening

    Orch->>CompSvc: POST /api/kyc/risk-score — PEP sanctions and adverse media
    activate CompSvc
    CompSvc->>CompSvc: Screen against PEP database
    CompSvc->>CompSvc: Check OFAC and HMT sanctions lists
    CompSvc->>CompSvc: Score risk level LOW, MEDIUM, or HIGH
    CompSvc-->>Orch: riskLevel=LOW, MEDIUM, or HIGH
    deactivate CompSvc

    alt Risk level LOW or MEDIUM — APPROVED

        Orch->>Orch: Update saga state to APPROVED

        Note over Orch,AcctSvc: Step 4 — Account Provisioning

        Orch->>AcctSvc: POST /api/accounts/open — provision current account for customer
        activate AcctSvc
        AcctSvc->>AcctSvc: Generate IBAN
        AcctSvc->>AcctSvc: Create Account record — status ACTIVE
        AcctSvc-->>Orch: accountId, IBAN created
        deactivate AcctSvc

        Note over Orch,NotifSvc: Step 5 — Welcome Notification

        Orch->>NotifSvc: POST /api/notifications — send welcome pack
        activate NotifSvc
        NotifSvc->>NotifSvc: Render welcome email and SMS template
        NotifSvc->>NotifSvc: Queue delivery attempts
        NotifSvc-->>Orch: messageLogId — notification queued
        deactivate NotifSvc

        Orch->>Orch: Update saga state to SAGA_COMPLETE
        Orch-->>ExpKyc: Saga complete — accountId, IBAN, kycStatus=APPROVED
        ExpKyc-->>GW: 201 Created — kycApproved account opened
        GW-->>CompMFE: 201 Created
        CompMFE-->>Customer: Congratulations — account opened with IBAN

    else Risk level HIGH or SANCTIONS HIT — REJECTED

        Orch->>Orch: Update saga state to REJECTED

        Note over Orch,CompSvc: Compensation — archive KYC attempt

        Orch->>CompSvc: POST /api/kyc/archive — store attempt with rejection reason
        activate CompSvc
        CompSvc->>CompSvc: Archive KYC record for regulatory audit trail
        deactivate CompSvc

        Orch->>NotifSvc: POST /api/notifications — send rejection notification
        activate NotifSvc
        NotifSvc->>NotifSvc: Render rejection letter template
        NotifSvc-->>Orch: notification queued
        deactivate NotifSvc

        Orch->>Orch: Update saga state to SAGA_REJECTED
        Orch-->>ExpKyc: Saga rejected — kycStatus=REJECTED reason=HIGH_RISK
        ExpKyc-->>GW: 422 Unprocessable Entity — KYC rejected
        GW-->>CompMFE: 422 — KYC not approved
        CompMFE-->>Customer: KYC application unsuccessful — review letter sent

    end

    deactivate Orch
    deactivate ExpKyc
```

---

## Flow 5: Strangler Fig — Traffic Cutover

> **Journey:** Platform migration team extracts the `account-service` domain from the legacy monolith. This flow shows the full cutover sequence — from data migration through dual-write validation to final routing switch and legacy retirement.
>
> **Pattern:** Strangler Fig — incremental traffic migration via API Gateway routing table update, validated by parity checks and canary progression.
>
> → Architecture detail: [L0_L1_ARCHITECTURE.md §5](./L0_L1_ARCHITECTURE.md)

```mermaid
sequenceDiagram
    autonumber
    participant MFE as MFE Clients
    participant GW as API Gateway\nSpring Cloud Gateway
    participant Legacy as Legacy Core Banking Monolith
    participant MigJob as Data Migration Job\nLiquibase ETL
    participant NewSvc as account-service\nNew Microservice
    participant ACL as Anti-Corruption Layer
    participant Dual as Dual-Write Adapter\nTemporary

    Note over GW,Legacy: Phase 1 — Proxy Installed — 100 percent traffic to legacy

    MFE->>GW: All API requests
    GW->>Legacy: Proxy 100 percent of all paths to legacy
    Legacy-->>GW: Responses from legacy
    GW-->>MFE: Responses passed through

    Note over MigJob,NewSvc: Phase 2 — Pre-cutover data migration begins

    MigJob->>Legacy: Read all account records from legacy database
    MigJob->>ACL: Translate legacy schema to new DDD model format
    ACL->>NewSvc: Write migrated account records to new PostgreSQL database
    MigJob->>MigJob: Run parity check — zero discrepancies confirmed

    Note over NewSvc,Legacy: Dual-write adapter activated — new service writes to both stores

    NewSvc->>Dual: All writes replicated
    Dual->>ACL: Translate new model to legacy format
    ACL->>Legacy: Dual-write to legacy for consistency during cutover

    Note over MFE,Legacy: Validation gates — must all pass before routing switch

    MFE->>NewSvc: Pact contract test suite — all provider tests green
    MFE->>NewSvc: k6 load test — P99 within 10 percent of legacy baseline
    MFE->>NewSvc: OWASP ZAP scan — zero High or Critical findings

    Note over GW,NewSvc: Phase 2 routing switch — canary begins

    GW->>GW: Update routing table\nGET POST /api/accounts\nrouted to new account-service at 10 percent

    MFE->>GW: Account API requests
    GW->>NewSvc: Route 10 percent of account traffic to new service
    GW->>Legacy: Route 90 percent of account traffic to legacy

    Note over GW,Legacy: Canary progresses after error-rate validation

    GW->>GW: Canary 50 percent — no errors after 24 hours
    GW->>GW: Canary 100 percent — no errors after 72 hours

    Note over NewSvc,Legacy: Remove dual-write adapter — legacy no longer receives account writes

    NewSvc->>Dual: Dual-write adapter removed
    GW->>Legacy: Zero account traffic to legacy

    Note over GW,Legacy: Phase 3 — Legacy account domain fully strangled

    MFE->>GW: All account API requests
    GW->>NewSvc: 100 percent of account traffic to new microservice
    Legacy-->>Legacy: Account domain idle — no incoming traffic

    Note over Legacy: Phase 4 — Legacy decommissioned after all 6 domains extracted
```

---

## Flow 6: Saga Compensation — Payment Rollback

> **Journey:** A payment that was partially completed encounters a failure after the account was debited (e.g., payment network timeout after authorisation). The Choreography Saga triggers compensating transactions to restore consistency across all affected services.
>
> **Pattern:** Compensating transaction chain — each service subscribes to `payment.failed` or `payment.reversed` and executes its own compensation action in parallel. The audit trail receives an immutable compensation record.
>
> → Architecture detail: [L0_L1_ARCHITECTURE.md §4.1](./L0_L1_ARCHITECTURE.md) · [BACKEND_ARCHITECTURE.md §3.2](./BACKEND_ARCHITECTURE.md)

```mermaid
sequenceDiagram
    autonumber
    participant PaySvc as payment-service
    participant Kafka as Apache Kafka
    participant AcctSvc as account-service
    participant NotifSvc as notification-service
    participant CompSvc as compliance-service\naudit trail

    Note over PaySvc: Payment PAY-001 was AUTHORISED and account was debited\nBut payment network returned timeout — cannot confirm settlement

    PaySvc->>PaySvc: Payment network timeout after 30 seconds
    PaySvc->>PaySvc: Transition Payment status to FAILED
    PaySvc->>PaySvc: Save reversal OutboxEvent in same @Transactional block

    Note over PaySvc,Kafka: OutboxRelay publishes the compensation event

    PaySvc->>Kafka: Publish payment.failed\npaymentId=PAY-001 reason=NETWORK_TIMEOUT

    Note over Kafka: Three services independently consume payment.failed in parallel

    par Compensate account debit
        Kafka->>AcctSvc: Consume payment.failed
        activate AcctSvc
        AcctSvc->>AcctSvc: Check if debit Transaction exists for PAY-001
        AcctSvc->>AcctSvc: Create REVERSAL Transaction record — credit GBP 500 back
        AcctSvc->>AcctSvc: Update account balance restored to original
        AcctSvc->>Kafka: Publish account.debit.reversed\npaymentId=PAY-001 amount=GBP500
        deactivate AcctSvc

    and Notify customer of failure
        Kafka->>NotifSvc: Consume payment.failed
        activate NotifSvc
        NotifSvc->>NotifSvc: Render payment failure notification template
        NotifSvc->>NotifSvc: Send SMS alert — payment did not complete
        NotifSvc->>NotifSvc: Send email with reversal timeline information
        deactivate NotifSvc

    and Append to immutable audit trail
        Kafka->>CompSvc: Consume payment.failed
        activate CompSvc
        CompSvc->>CompSvc: Append PAYMENT_FAILED event to audit.trail topic
        CompSvc->>CompSvc: Record compensation timeline for PCI-DSS audit
        CompSvc->>CompSvc: Flag for risk team review if pattern detected
        deactivate CompSvc
    end

    Note over AcctSvc: Kafka->>AcctSvc: Consume account.debit.reversed\nBalance reconciliation confirmed

    Note over PaySvc: Final state — payment PAY-001 status=FAILED\naccount balance fully restored\ncustomer notified\naudit trail complete
```

---

## Validation Checkpoints — Step-by-Step Verification Guide

Use these checkpoints to validate each flow in your environment:

### Flow 1 — Account Overview
| Step | What to Verify | Expected Result |
|---|---|---|
| MFE → Gateway | JWT is attached to request | Request reaches API Gateway with Authorization header |
| Gateway → Experience API | JWT validated, route matched | HTTP 200 response within 300ms |
| Parallel calls | Both services called concurrently | account-service and trading-service called simultaneously |
| Composite response | Single response for Dashboard | Response contains accounts array and portfolio summary |

### Flow 2 — Payment Initiation
| Step | What to Verify | Expected Result |
|---|---|---|
| Idempotency key | Duplicate key returns same result | Second call returns same paymentId without duplicate payment |
| Funding check | Insufficient balance rejected | HTTP 422 with INSUFFICIENT_FUNDS error before payment created |
| Saga starts | 202 Accepted returned | MFE receives paymentId immediately — saga runs in background |
| Kafka event | payment.initiated published | compliance-service consumer lag moves to 0 within 30s |
| AML check | Both paths tested | kyc.passed triggers AUTHORISED — kyc.flagged triggers REJECTED |
| Account debit | Balance updated after completion | account balance decremented by payment amount |

### Flow 3 — Trade Order
| Step | What to Verify | Expected Result |
|---|---|---|
| Funding check | Order rejected if insufficient | HTTP 422 before order created |
| MiFID II report | Created after execution | mifid_transaction_report row inserted with SUBMITTED status |
| Portfolio update | Reflects execution | Portfolio quantity and average cost updated after trade.executed |

### Flow 4 — KYC Onboarding Saga
| Step | What to Verify | Expected Result |
|---|---|---|
| Saga state | Visible in orchestrator | Saga record moves through each state transition |
| Compensation | Triggered on HIGH RISK | Account provisioning rolled back if risk step fails |
| Account opened | Created on APPROVED | New account with IBAN created and active |

### Flow 5 — Strangler Fig Cutover
| Step | What to Verify | Expected Result |
|---|---|---|
| Data parity | Before routing switch | Zero record discrepancies between legacy and new DB |
| Canary 10% | First traffic to new service | Error rate below 0.1% — Grafana dashboard confirms |
| Full cutover | 100% to new service | Legacy receives zero account domain traffic |
| Dual-write removed | No writes to legacy | Legacy account DB shows zero new writes |

### Flow 6 — Saga Compensation
| Step | What to Verify | Expected Result |
|---|---|---|
| Compensation idempotency | payment.failed consumed twice | account reversal applied only once — idempotency guard |
| Balance restored | After reversal | account balance equals pre-payment value |
| Audit trail | Immutable record appended | audit.trail contains PAYMENT_FAILED event with full context |

---

*Generated 2026 · Digital Banking and Wealth Platform — L0/L1 Sequence Diagrams Reference*
*Stack: React 18 · Webpack Module Federation · Java 21 · Spring Boot 3.3 · Spring Cloud 2023 · Apache Kafka · PostgreSQL 16 · Redis 7 · Kubernetes*
*Regulatory scope: PCI-DSS Level 1 · SOC 2 Type II · PSD2/Open Banking · MiFID II*
*Perspective: FinTech Principal Architects · Software Engineers · Quality Engineers*
