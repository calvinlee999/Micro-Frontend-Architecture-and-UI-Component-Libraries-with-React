# FinTech Enterprise Platform — L0/L1 Overall Architecture

> **Type:** Single Source of Truth — Platform-Wide Architecture Overview
> **Scope:** L0 System Context · L1 Container Architecture · Customer Journey Experience API · Saga Patterns · Strangler Fig Modernization Blueprint
> **Platform:** Digital Banking and Wealth Platform
> **Stack:** React 18 · Webpack Module Federation · Java 21 · Spring Boot 3.3 · Apache Kafka · PostgreSQL 16 · Redis 7 · Kubernetes
> **Regulatory:** PCI-DSS Level 1 · SOC 2 Type II · PSD2/Open Banking · MiFID II
> **Perspective:** FinTech Principal Architects · Software and Quality Engineers

---

## Quick Navigation — All Architecture References

| Layer | Document | Scope |
|---|---|---|
| **This document** | L0_L1_ARCHITECTURE.md | L0/L1 overview · Experience API · Saga · Strangler Fig |
| **L0/L1 Sequences** | [L0_L1_SEQUENCE_DIAGRAMS.md](./L0_L1_SEQUENCE_DIAGRAMS.md) | Customer Journey end-to-end flows · Saga · Strangler Fig migration |
| **Front-End Architecture** | [ARCHITECTURE.md](./ARCHITECTURE.md) | MFE topology · Module Federation · Design System · Auth · Feature Flags |
| **Front-End Sequences** | [SEQUENCE_DIAGRAMS.md](./SEQUENCE_DIAGRAMS.md) | PKCE auth · MFE lazy load · PCI-DSS boundary · audit trail · token refresh |
| **Back-End Architecture** | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | API Gateway · 6 domain microservices · Kafka · Data Layer · Security · ORM · ADRs |
| **Back-End Sequences** | [BACKEND_SEQUENCE_DIAGRAMS.md](./BACKEND_SEQUENCE_DIAGRAMS.md) | Payment Saga · KYC/AML · trading flows · notification · auth token lifecycle |

---

## How to Read This Document

| Level | What It Shows | Primary Audience |
|---|---|---|
| **L0 System Context** | What the platform does and who uses it | Business stakeholders · Product owners |
| **L1 Container Architecture** | All deployable containers and how they connect | All engineers · Architects |
| **Experience API Layer** | Customer Journey BFF connecting MFE to domain services | Full-stack architects · Senior engineers |
| **Saga Pattern** | Cross-domain (Choreography via Kafka) and same-domain (Orchestration) coordination | Back-end engineers · Architects |
| **Strangler Fig Blueprint** | Incremental legacy modernization — phase by phase with validation gates | Principal architects · Transformation leads |

---

## 1. L0 — System Context

The platform serves three categories of external actors and integrates with six external systems. All customer traffic enters through the Front-End MFE layer. Open Banking TPPs and internal operations teams call the Experience API directly.

```mermaid
flowchart TB
    CUST["Customer\nWeb Browser · iOS · Android App"]
    TPP["Open Banking TPP\nPSD2 Third-Party Provider"]
    OPS["Operations and Compliance Team\nInternal Admin Portal"]

    subgraph PLATFORM["Digital Banking and Wealth Platform"]
        FE["Front-End Layer\nMFE Shell and 4 Domain MFEs\nReact 18 · Webpack Module Federation\nDesign System · TypeScript · Tailwind CSS"]
        EXP["Experience API Layer\nCustomer Journey BFF\nOrchestrates cross-domain journeys\nSaga coordination · Composite responses"]
        BE["Back-End Domain Microservices\naccount · payment · trading · compliance · auth · notification\nJava 21 · Spring Boot 3.3 · Apache Kafka · PostgreSQL 16"]
    end

    IDP["Identity Provider\nOAuth2 OIDC · MFA · PKCE"]
    NET["Payment Networks\nSWIFT · SEPA · Visa · Mastercard"]
    MKT["Market Data Feed\nBloomberg · Refinitiv"]
    REG["Regulatory Bodies\nFCA · ESMA · NCA · FIU"]
    NOTIF_P["Notification Providers\nSendGrid · Twilio · Firebase FCM"]
    FRAUD_E["Fraud Risk Engine\nExternal Scoring API"]

    CUST -->|"HTTPS — Web or Mobile"| FE
    TPP -->|"Open Banking REST API PSD2"| EXP
    OPS -->|"Internal Portal REST"| EXP

    FE -->|"REST JSON Bearer JWT"| EXP
    EXP -->|"REST JSON Bearer JWT via API Gateway"| BE

    BE -.->|"OAuth2 PKCE"| IDP
    BE -.->|"ISO 20022 SEPA SWIFT"| NET
    BE -.->|"WebSocket FIX protocol"| MKT
    BE -.->|"RTS 22 SAR MiFID II reports"| REG
    BE -.->|"Email SMS Push"| NOTIF_P
    BE -.->|"Risk score API"| FRAUD_E
```

---

## 2. L1 — Container Architecture

### 2.1 Front-End Containers

> Full detail: [ARCHITECTURE.md](./ARCHITECTURE.md)

```mermaid
flowchart TB
    subgraph BROWSER["Browser — React 18 · TypeScript · Tailwind CSS"]
        SHELL["Shell — Module Federation Host\nOAuth2 PKCE · React Router · Global Nav\nFeature Flags LaunchDarkly · Audit Dispatcher\nport 3000"]

        subgraph REMOTES["Domain MFEs — Module Federation Remotes"]
            MFE_D["Dashboard MFE\nAccounts · Portfolio Summary\nport 3001"]
            MFE_P["Payments MFE\nSend Money · History · Open Banking\nPCI-DSS tokenised card iframe\nport 3002"]
            MFE_T["Trading MFE\nOrder Book · Portfolio · Market Data\nMiFID II reporting\nport 3003"]
            MFE_C["Compliance MFE\nKYC Forms · AML Alerts · Doc Upload\nport 3004"]
        end

        DS["Design System\n@fintechbank/design-system\nStorybook 8 · Atomic Design\nAtoms · Molecules · Organisms · Templates\nChromatic visual regression · axe-core a11y"]
    end

    GW_ENTRY["API Gateway\nSpring Cloud Gateway :8080\nJWT validation · Rate limiting · Circuit breaker"]

    SHELL -->|"React.lazy + Suspense\nfetches remoteEntry.js from CDN"| REMOTES
    REMOTES -.->|"build-time npm dependency"| DS
    SHELL -.->|"build-time npm dependency"| DS
    SHELL -->|"Bearer JWT REST"| GW_ENTRY
    REMOTES -->|"Bearer JWT REST"| GW_ENTRY
```

---

### 2.2 Experience API Layer — Customer Journey BFF

The Experience API layer provides **Journey-Scoped Backend for Frontend (BFF) services**. Each BFF:

- Receives one MFE request for a complete customer journey screen
- Aggregates data from multiple domain services in parallel or sequence
- Returns a response tailored to what the MFE needs — not what the domain model exposes
- Coordinates Sagas for journeys that span multiple domains
- Enforces idempotency, circuit breaking, and timeout contracts

```mermaid
flowchart TB
    subgraph MFE_IN["Front-End Callers"]
        M1["Dashboard MFE"]
        M2["Payments MFE"]
        M3["Trading MFE"]
        M4["Compliance MFE"]
    end

    GW["API Gateway\nSpring Cloud Gateway :8080\nJWT validation · Rate limiting · Circuit breaker\nEureka service discovery"]

    subgraph EXP_API["Experience API Layer — Journey BFF Services"]
        E1["Account Overview Journey API\nGET /experience/v1/accounts/overview\nAggregates: accounts + transactions + portfolio summary\nParallel calls — single composite response for Dashboard MFE"]
        E2["Payment Journey API\nPOST /experience/v1/payments/initiate\nOrchestrates: balance-check + fraud-precheck + payment-create\nSaga coordinator — returns 202 Accepted with payment ID"]
        E3["Trading Journey API\nPOST /experience/v1/trading/orders\nOrchestrates: funding-check + order-place + MiFID-II-report\nSaga coordinator — returns 201 Created with order ID"]
        E4["KYC Onboarding Journey API\nPOST /experience/v1/kyc/onboard\nOrchestrates: doc-verify + biometric + risk-score + account-open\nSaga orchestrator — KycSagaOrchestrator manages state machine"]
    end

    subgraph DOMAIN["Domain Microservices"]
        ACCT["account-service :8081"]
        PMT["payment-service :8082"]
        TRADE["trading-service :8083"]
        COMP["compliance-service :8084"]
        NOTIF["notification-service :8086"]
    end

    M1 --> GW
    M2 --> GW
    M3 --> GW
    M4 --> GW

    GW --> E1
    GW --> E2
    GW --> E3
    GW --> E4

    E1 -->|"GET /accounts"| ACCT
    E1 -->|"GET /portfolio/summary"| TRADE
    E2 -->|"GET /balance — sync check"| ACCT
    E2 -->|"POST /payments"| PMT
    E2 -->|"POST /compliance/pre-check"| COMP
    E3 -->|"GET /balance — funding check"| ACCT
    E3 -->|"POST /orders"| TRADE
    E3 -->|"POST /mifid/report"| COMP
    E4 -->|"POST /kyc/verify-docs"| COMP
    E4 -->|"POST /kyc/biometric"| COMP
    E4 -->|"POST /accounts/open"| ACCT
    E4 -->|"POST /notifications/welcome"| NOTIF
```

---

### 2.3 Back-End Domain Microservices

> Full detail: [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)

```mermaid
flowchart TB
    GW2["API Gateway :8080\nSpring Cloud Gateway\nJWT · Rate limit · Circuit breaker"]

    subgraph SC["Spring Cloud Infrastructure"]
        CFG["Config Server :8888\nGit-backed · @RefreshScope"]
        EUR["Eureka Server :8761\nService registry · K8s DNS fallback"]
    end

    subgraph MS["Domain Microservices"]
        ACCT2["account-service :8081\nBalance · Transactions · Limits\nJPA entities · PostgreSQL 16"]
        PMT2["payment-service :8082\nPSD2 · SEPA · Fraud pre-check\nPCI-DSS scope · Outbox pattern"]
        TRADE2["trading-service :8083\nOrder management · Executions\nMiFID II audit · Portfolio valuation"]
        COMP2["compliance-service :8084\nKYC engine · AML · SAR filing\nRisk scoring · Rule engine"]
        AUTH2["auth-service :8085\nOAuth2 server · JWT RS256\nMFA · PKCE · Redis sessions"]
        NOTIF2["notification-service :8086\nEmail · SMS · Push\nTemplates · Retry · DeliveryLog"]
    end

    subgraph KFKA["Apache Kafka — Event Streaming"]
        KT1["payment.initiated · payment.completed · payment.failed"]
        KT2["trade.order.placed · trade.executed · trade.cancelled"]
        KT3["kyc.triggered · kyc.passed · kyc.flagged"]
        KT4["audit.trail — append-only PCI-DSS and MiFID II"]
    end

    subgraph DB_TIER["Data Layer — Private Subnets"]
        PGACCT[("account-db\nPostgreSQL 16")]
        PGPMT[("payment-db\nPostgreSQL 16\nPCI-DSS")]
        PGTRADE[("trading-db\nPostgreSQL 16\nMiFID II")]
        PGCOMP[("compliance-db\nPostgreSQL 16")]
        PGNOTIF[("notification-db\nPostgreSQL 16")]
        RDS[("Redis 7\nJWT deny-list · Rate limit · Sessions")]
        VLT["HashiCorp Vault\nDB credentials · PCI keys · JWT signing keys"]
    end

    GW2 --> MS
    MS -->|"register and heartbeat"| EUR
    MS -->|"pull config on boot"| CFG

    PMT2 --> KT1
    TRADE2 --> KT2
    COMP2 --> KT3

    KT1 --> COMP2
    KT1 --> NOTIF2
    KT2 --> COMP2
    KT2 --> NOTIF2
    KT3 --> PMT2
    KT3 --> NOTIF2

    ACCT2 --- PGACCT
    PMT2 --- PGPMT
    TRADE2 --- PGTRADE
    COMP2 --- PGCOMP
    NOTIF2 --- PGNOTIF

    AUTH2 --- RDS
    GW2 --- RDS
    VLT -.->|"Vault CSI secrets injection"| PMT2
    VLT -.->|"Vault CSI secrets injection"| AUTH2
```

---

## 3. Customer Journey Experience API Design

### 3.1 Journey-to-Service Mapping Table

| Customer Journey | Trigger MFE | Experience API Endpoint | Domain Services Orchestrated | Pattern |
|---|---|---|---|---|
| Account Overview | Dashboard MFE | `GET /experience/v1/accounts/overview` | account-service + trading-service | Parallel aggregation |
| Payment Initiation | Payments MFE | `POST /experience/v1/payments/initiate` | account + payment + compliance + notification | Choreography Saga via Kafka |
| Trade Order | Trading MFE | `POST /experience/v1/trading/orders` | account + trading + compliance | Orchestration + async Kafka confirm |
| KYC Onboarding | Compliance MFE | `POST /experience/v1/kyc/onboard` | compliance + account + notification | Orchestration Saga |
| Account Statement | Dashboard MFE | `GET /experience/v1/accounts/{id}/statement` | account-service | Direct proxy |
| Open Banking Consent | Payments MFE | `POST /experience/v1/openbanking/consent` | auth-service + account-service | PSD2 consent flow |

### 3.2 Experience API Design Principles

| Principle | Description |
|---|---|
| **Journey-first design** | API contract designed for the MFE customer journey — not for domain entity structure |
| **Composite responses** | One MFE request returns aggregated data from N domain services |
| **Saga coordinator** | Experience API owns saga state and compensation logic for multi-step journeys |
| **Idempotency** | All POST endpoints accept an `Idempotency-Key` header — safe to retry |
| **Circuit breaking** | Resilience4j wraps each domain service call with fallback |
| **Timeout contract** | 3s hard timeout per domain service call — 8s total per Experience API request |
| **Versioning** | `/v1/` URL prefix — backward-compatible contract evolution |
| **JWT forwarding** | Experience API forwards the caller's Bearer JWT downstream to all domain services |

---

## 4. Saga Pattern — Transaction Coordination

### 4.1 Choreography Saga — Cross-Domain via Apache Kafka

Used when services are **loosely coupled** and can independently react to events. No central coordinator. Each service subscribes to events, performs its action, and publishes the next event in the chain.

**Example: Payment Initiation Saga**

```mermaid
flowchart LR
    subgraph CHORE["Payment Choreography Saga — Kafka Event Chain"]
        direction TB
        P1["1 Payments MFE\nPOST /experience/payments/initiate"]
        P2["2 payment-service\nSave Payment — status CREATED\nPublish payment.initiated to Kafka"]
        P3["3 compliance-service\nSubscribes payment.initiated\nRuns AML and fraud check\nResult: CLEAR or FLAGGED"]
        P4A["4a payment-service\nSubscribes kyc.passed\nTransitions status to AUTHORISED\nPublishes payment.completed"]
        P4B["4b payment-service\nSubscribes kyc.flagged\nTransitions status to REJECTED\nPublishes payment.failed"]
        P5A["5a account-service\nSubscribes payment.completed\nDebits account balance"]
        P5B["5b notification-service\nSubscribes payment.completed\nSends confirmation SMS and email"]
        P5C["5c notification-service\nSubscribes payment.failed\nSends rejection alert to customer"]

        P1 --> P2
        P2 -->|"Kafka: payment.initiated"| P3
        P3 -->|"Kafka: kyc.passed"| P4A
        P3 -->|"Kafka: kyc.flagged"| P4B
        P4A -->|"Kafka: payment.completed"| P5A
        P4A -->|"Kafka: payment.completed"| P5B
        P4B -->|"Kafka: payment.failed"| P5C
    end
```

**Compensating Transaction (Rollback):**

```mermaid
flowchart LR
    FAIL2["payment-service\nPublishes payment.failed\nor payment.reversed"]
    CA["account-service\nSubscribes payment.failed\nReverses debit if already applied\nPublishes account.debit.reversed"]
    CN["notification-service\nSubscribes payment.failed\nSends customer failure alert"]
    CAU["compliance-service\nSubscribes payment.failed\nAppends PAYMENT_ROLLBACK\nto immutable PCI-DSS audit trail"]

    FAIL2 --> CA
    FAIL2 --> CN
    FAIL2 --> CAU
```

---

### 4.2 Orchestration Saga — Same-Domain via Saga Orchestrator

Used when steps must be **sequentially ordered** with explicit compensation, and the saga needs centralised state management. A dedicated Saga Orchestrator service owns the state machine and controls each step.

**Example: KYC Onboarding Saga**

```mermaid
flowchart TB
    KYC_IN["Compliance MFE\nPOST /experience/v1/kyc/onboard"]
    ORCH2["KycOnboardingSagaOrchestrator\nState machine\nSTARTED > DOCS_VERIFIED > BIOMETRIC_PASSED\n> RISK_SCORED > APPROVED or REJECTED"]

    subgraph STEPS["Sequential Saga Steps — Orchestrator Calls Each in Order"]
        ST1["Step 1 Document Verification\ncompliance-service\nOCR scan and identity document match"]
        ST2["Step 2 Biometric Check\ncompliance-service\nFace match and liveness detection"]
        ST3["Step 3 PEP and Sanctions Screening\ncompliance-service\nRisk scoring rule engine"]
        ST4["Step 4 Account Provisioning\naccount-service\nCreate account on APPROVED decision"]
        ST5["Step 5 Welcome Notification\nnotification-service\nSend welcome pack and card details"]
    end

    C1["Compensate Step 4\nClose provisioned account\nif later step fails"]
    C2["Compensate and Archive\nStore KYC attempt record\nNotify applicant of rejection reason"]

    KYC_IN --> ORCH2
    ORCH2 --> ST1
    ST1 -->|"DOCS_VERIFIED"| ST2
    ST2 -->|"BIOMETRIC_PASSED"| ST3
    ST3 -->|"LOW or MEDIUM RISK"| ST4
    ST4 -->|"ACCOUNT_CREATED"| ST5
    ST5 -->|"SAGA COMPLETE"| ORCH2

    ST3 -->|"HIGH RISK or SANCTIONS HIT"| C2
    ST4 -->|"PROVISIONING FAILED"| C1
    C1 --> C2
```

### 4.3 Saga Pattern Decision Matrix

| Criterion | Choreography Saga | Orchestration Saga |
|---|---|---|
| **Coupling** | Loose — services react to events independently | Tighter — orchestrator drives each step |
| **State visibility** | Distributed across consumers — trace via OpenTelemetry | Centralised in orchestrator — easy to inspect and audit |
| **Failure handling** | Compensating events published per service | Orchestrator controls exact compensation sequence |
| **Best fit — cross-domain** | Ideal — Kafka decouples domain boundaries cleanly | Possible but orchestrator crosses bounded contexts |
| **Best fit — same-domain** | Can be used | Preferred — explicit sequential step ordering |
| **Observability** | Requires distributed trace with correlation IDs | Orchestrator holds full saga state history |
| **Example in platform** | Payment completion triggering compliance + account + notification | KYC onboarding with 5 sequential steps and compensations |

---

## 5. Strangler Fig Modernization Blueprint

> **Goal:** Replace a legacy Core Banking Monolith incrementally — one domain at a time — without a big-bang rewrite.
> **Method:** Deploy an API Gateway proxy in front of the legacy system. Build new microservices in parallel. Shift gateway routing domain by domain. Decommission legacy only after 100% traffic validation at each phase.
> **Principle:** Never rewrite everything at once. Build new capability alongside old. Shift traffic incrementally. Validate before each shift. Strangle domain by domain.

### 5.0 Migration Phases Overview

```mermaid
flowchart LR
    PH0["Phase 0\nLegacy Monolith Only\nAll business logic in one deployable\nClients call monolith directly\nNo API Gateway"]
    PH1["Phase 1\nProxy Layer Installed\nAPI Gateway deployed in front\n100 percent of traffic proxied to legacy\nZero logic change — low risk"]
    PH2["Phase 2\nFirst Domain Extracted\naccount-service built and live\nGateway routes accounts traffic to new service\nLegacy still handles remaining domains"]
    PH3["Phase 3\nAll Domains Extracted\nAll 6 microservices live\nGateway routes 100 percent to new services\nLegacy running in cold standby"]
    PH4["Phase 4\nLegacy Decommissioned\nMonolith shut down completely\nMicroservices-only platform\nFull cloud-native operation"]

    PH0 -->|"Install API Gateway\nno business change"| PH1
    PH1 -->|"Extract account domain\nbuild and test new service"| PH2
    PH2 -->|"Extract domains 2 to 6\none domain per sprint cycle"| PH3
    PH3 -->|"72h at 100 percent canary\nstakeholder sign-off"| PH4
```

---

### 5.1 Phase 1 — Proxy Layer Installation

Install the API Gateway (Spring Cloud Gateway) in front of the existing legacy monolith. Route 100% of traffic through the proxy. Zero functional change to business logic. Validates the proxy layer is invisible to users.

```mermaid
flowchart LR
    CL["MFE Clients\nBrowser · Mobile"]
    GW3["API Gateway\nSpring Cloud Gateway\nJWT validation\nRate limiting installed"]
    LEG["Legacy Core Banking Monolith\nAll domains: Account · Payment · Trading · KYC\nRuns completely unchanged"]

    CL -->|"HTTPS — all traffic"| GW3
    GW3 -->|"100 percent of all API calls proxied"| LEG
```

**Validation checkpoint:** Smoke test confirms proxy adds less than 2ms latency. Zero 5xx errors. Functional parity confirmed.

---

### 5.2 Phase 2 — Domain Extraction with Anti-Corruption Layer

Extract one domain at a time. The API Gateway routes requests for the extracted domain to the new microservice. All other paths still proxy to the legacy monolith. An Anti-Corruption Layer (ACL) translates the data model between old and new systems.

```mermaid
flowchart TB
    CL2["MFE Clients"]
    GW4["API Gateway\nPer-path routing table\nUpdated for extracted domain"]

    NEW_ACCT["account-service — New Microservice\nSpring Boot · PostgreSQL 16\nDDD model: IBAN-centric Account entity\nOptimistic locking with @Version"]

    ACL_ACCT["Anti-Corruption Layer\nData model translator\nlegacy customer_code maps to UUID\nlegacy balance_pence maps to BigDecimal GBP\nlegacy status_code maps to AccountStatus enum"]

    LEG2["Legacy Monolith\nPayment · Trading · KYC\nStill handles all other domains"]

    SYNC["Data Migration Job\nOne-time ETL run before cutover\nHistorical account records migrated\nValidated with parity check"]

    DUAL["Dual-Write Adapter\nTemporary — active during cutover window\nNew service writes to both new and legacy DB\nRemoved once cutover is confirmed stable"]

    CL2 --> GW4
    GW4 -->|"GET POST /api/accounts/**\nrouted to new microservice"| NEW_ACCT
    GW4 -->|"All other paths still proxied"| ACL_ACCT
    ACL_ACCT --> LEG2
    NEW_ACCT -.->|"dual-write during cutover window"| DUAL
    DUAL -.->|"translated write"| LEG2
    SYNC -.->|"run before routing switch"| NEW_ACCT
```

---

### 5.3 Phase 3 — Traffic Migration Validation Gates

All gates must pass before shifting traffic and before decommissioning each legacy domain:

| Gate | What to Validate | Tool | Pass Criteria |
|---|---|---|---|
| **Data parity** | New service data matches legacy for all records | Liquibase diff + reconciliation job | Zero record discrepancies |
| **Functional parity** | All API contracts satisfied by new service | Pact Consumer-Driven Contract tests | All provider tests green |
| **Performance parity** | P99 latency of new service vs legacy baseline | k6 load test comparison | Within 10% of legacy P99 |
| **Security parity** | No new OWASP Top 10 vulnerabilities introduced | OWASP ZAP baseline scan | Zero High or Critical findings |
| **Regulatory parity** | Audit trail, PCI-DSS, MiFID II reports equivalent | Compliance team review | Written sign-off obtained |
| **Canary validation** | 100% canary traffic without regression | Prometheus error-rate dashboard | Less than 0.1% errors for 72 hours |

---

### 5.4 Phase 4 — Legacy Decommission Sequence

```mermaid
flowchart TB
    CHK["Pre-Decommission Checks\nAll 6 domains extracted and canary-validated\nData archives exported to cold storage\nAudit logs archived for regulatory retention period\nHashiCorp Vault legacy DB credentials revoked\nCTO and Chief Risk Officer written sign-off"]

    EXEC["Decommission Execution\nLegacy Kubernetes Deployment scaled to 0 replicas\nLegacy database connections revoked in Vault\nLegacy codebase tagged as archive-final in Git\nDNS CNAME and routing rules removed\nMonitoring alerts redirected to new services\nRunbook updated — legacy ops runbooks archived"]

    POST["Post-Decommission State\nMicroservices-only cloud-native platform\nAPI Gateway is sole entry point\nFull OpenTelemetry distributed tracing\nPrometheus and Grafana observability\nZero legacy infrastructure running costs\nCI/CD pipeline covers only new microservices"]

    CHK --> EXEC
    EXEC --> POST
```

---

### 5.5 Anti-Corruption Layer (ACL) Design

The ACL isolates each new bounded context from the legacy domain model. It translates between legacy data schemas and the new DDD-aligned entity models without contaminating new microservices with legacy concepts.

```mermaid
flowchart LR
    NEW_SVC["New account-service\nDDD Bounded Context\nAccount entity — IBAN-centric\nBigDecimal balance in GBP\nAccountStatus enum\nLocalDate for dates"]

    ACL_IN["ACL — Inbound Translator\nlegacy customer_code to UUID\nbalance_pence INTEGER to BigDecimal GBP\nstatus_code CHAR to AccountStatus enum\ndate INTEGER YYYYMMDD to LocalDate"]

    LEG_DB["Legacy Core Banking Database\ncustomer_code VARCHAR(20)\nbalance_pence INTEGER\nstatus_code CHAR(1)\nopen_date INTEGER YYYYMMDD format"]

    ACL_OUT2["ACL — Outbound Translator\nNew AccountDTO to legacy wire format\nActive only during dual-write window\nRemoved completely at cutover confirmation"]

    NEW_SVC -->|"write during dual-write"| ACL_OUT2
    ACL_OUT2 -->|"translated write to legacy"| LEG_DB
    LEG_DB -->|"read from legacy during migration"| ACL_IN
    ACL_IN -->|"translated read to new service"| NEW_SVC
```

**ACL responsibilities:**

| Direction | What ACL Does |
|---|---|
| **Inbound (legacy → new)** | Translates legacy data types, codes, and identifiers into the new DDD model vocabulary |
| **Outbound (new → legacy)** | Reverse-translates new model writes back to legacy format during dual-write window only |
| **Event bridging** | Converts legacy database polling events to new DDD domain events for Kafka |
| **Removed when** | After cutover validation passes and dual-write window closes — ACL is temporary |

---

## 6. Cross-Cutting Concerns

| Concern | Implementation | Reference |
|---|---|---|
| **Authentication** | OAuth2 PKCE · RS256 JWT · MFA · 15-min token TTL · httpOnly refresh cookie | [ARCHITECTURE.md §4](./ARCHITECTURE.md) · [BACKEND_ARCHITECTURE.md §6](./BACKEND_ARCHITECTURE.md) |
| **Authorisation** | Spring Security `@PreAuthorize` · RBAC roles per domain scope | [BACKEND_ARCHITECTURE.md §6](./BACKEND_ARCHITECTURE.md) |
| **Observability** | OpenTelemetry · Prometheus · Grafana · ELK Stack · Jaeger distributed tracing | [BACKEND_ARCHITECTURE.md §7](./BACKEND_ARCHITECTURE.md) |
| **PCI-DSS** | Card iframe isolation · Vault TokenVault encryption · payment-db dedicated subnet + NetworkPolicy | [ARCHITECTURE.md §4.1](./ARCHITECTURE.md) · [BACKEND_ARCHITECTURE.md §5.0.3](./BACKEND_ARCHITECTURE.md) |
| **MiFID II** | Trade records 7-year retention · RTS 22 transaction reports · Append-only execution log + Row Security | [BACKEND_ARCHITECTURE.md §5.0.4](./BACKEND_ARCHITECTURE.md) |
| **PSD2** | Open Banking APIs · SCA Strong Customer Authentication · Consent management lifecycle | [BACKEND_ARCHITECTURE.md §3.2](./BACKEND_ARCHITECTURE.md) |
| **Accessibility** | WCAG 2.1 AA · axe-core CI gate · Storybook a11y addon · Chromatic visual regression | [ARCHITECTURE.md §3](./ARCHITECTURE.md) |
| **Testing Pyramid** | 70% unit · 20% integration Testcontainers · 8% contract Pact · 2% E2E k6 and OWASP ZAP | [BACKEND_ARCHITECTURE.md §9](./BACKEND_ARCHITECTURE.md) |
| **CI/CD** | GitHub Actions · Helm upgrade · Canary 10% to 50% to 100% · Metric gate auto-rollback | [BACKEND_ARCHITECTURE.md §8.2](./BACKEND_ARCHITECTURE.md) |
| **Resilience** | Resilience4j circuit breaker · Bulkhead · Retry · `@Version` optimistic locking · K8s HPA | [BACKEND_ARCHITECTURE.md §3](./BACKEND_ARCHITECTURE.md) |
| **Schema versioning** | Liquibase per domain · Changelog contexts per environment · Zero manual DDL | [BACKEND_ARCHITECTURE.md §5.1](./BACKEND_ARCHITECTURE.md) |
| **Secrets management** | HashiCorp Vault · CSI driver injection · Zero secrets in environment variables or config files | [BACKEND_ARCHITECTURE.md §6](./BACKEND_ARCHITECTURE.md) |

---

*Generated 2026 · Digital Banking and Wealth Platform — L0/L1 Architecture Reference*
*Stack: React 18 · Webpack Module Federation · Java 21 · Spring Boot 3.3 · Spring Cloud 2023 · Apache Kafka · PostgreSQL 16 · Redis 7 · Kubernetes*
*Regulatory scope: PCI-DSS Level 1 · SOC 2 Type II · PSD2/Open Banking · MiFID II*
*Perspective: FinTech Principal Architects · Software Engineers · Quality Engineers*
