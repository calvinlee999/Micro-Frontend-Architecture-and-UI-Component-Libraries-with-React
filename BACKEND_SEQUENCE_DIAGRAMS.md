# Digital Banking & Wealth Platform — Back-End Sequence Diagrams

> **Platform:** Digital Banking & Wealth Platform — Back-End Engineering Reference  
> **Stack:** Java 21 · Spring Boot 3.3 · Spring Cloud 2023 · Apache Kafka · PostgreSQL 16 · Redis 7 · Kubernetes  
> **Regulatory scope:** PCI-DSS Level 1 · SOC 2 Type II · PSD2/Open Banking · MiFID II  
> **Perspective:** Principal Back-End Engineer · Solution Architect · Data Engineer · QE

---

## Index

| Flow | Scenario | Layer Focus |
|---|---|---|
| [Flow 1](#flow-1-api-gateway--jwt-validation--rate-limit--routing) | API Gateway: JWT validation, rate-limit, route to Payment service | Gateway · Security |
| [Flow 2](#flow-2-payment-initiation--transactional-processing--kafka-event) | Payment initiation: `@Transactional` processing + Kafka `payment.initiated` event | Service · Kafka |
| [Flow 3](#flow-3-pci-dss-card-tokenisation-via-vault) | PCI-DSS card tokenisation via HashiCorp Vault Transit Secrets Engine | Security · PCI |
| [Flow 4](#flow-4-account-service--redis-cache-aside-hit-and-miss-paths) | Account balance: Redis cache-aside (HIT path + MISS path) | Caching · Performance |
| [Flow 5](#flow-5-kafka-event-driven-compliance--kyc-check) | Kafka consumer: Compliance/KYC check triggered by `payment.initiated` | Event-Driven · Compliance |
| [Flow 6](#flow-6-trading-service--order-placement--optimistic-locking--mifid-ii-audit) | Trading order placement: `@Version` optimistic locking + MiFID II audit trail | Concurrency · Regulatory |
| [Flow 7](#flow-7-oauth2-jwt-resource-server-validation-chain) | OAuth2 JWT validation: JWKS fetch → signature verify → claims → SecurityContext | Auth · Security |
| [Flow 8](#flow-8-testcontainers-integration-test-lifecycle) | Testcontainers: PostgreSQL + Kafka containers, `@ServiceConnection`, test lifecycle | Testing · QE |
| [Flow 9](#flow-9-opentelemetry-distributed-tracing-across-services) | OpenTelemetry: trace propagation across Gateway → Payment → Kafka → Compliance | Observability |
| [Flow 10](#flow-10-microservice-canary-deploy--kubernetes-rolling-update) | Canary deploy: Kubernetes rolling update + Argo Rollouts metric gate | Deployment · SRE |

---

## Flow 1: API Gateway — JWT Validation, Rate-Limit, Routing

> **Scenario:** A mobile banking client sends `POST /api/payments` with a Bearer JWT. The Spring Cloud Gateway validates the token, checks Redis for revocation, enforces rate limiting, and routes to payment-service.  
> **Key patterns:** Global pre-filter chain · Redis token-bucket rate limiter · RFC-7807 ProblemDetail on rejection · W3C Trace Context injection · X-Correlation-Id propagation.

### 1a — Happy Path (valid JWT, under rate limit)

```mermaid
sequenceDiagram
    autonumber
    participant MFE as Mobile Client / MFE Shell
    participant GW as Spring Cloud Gateway :8080
    participant REDIS as Redis 7<br/>(deny-list + rate-limit)
    participant EUR as Eureka :8761
    participant PMT as payment-service :8082

    MFE->>GW: POST /api/payments<br/>Authorization: Bearer eyJhbGci...

    Note over GW: CORS Filter — validates Origin header

    GW->>GW: JwtAuthenticationFilter (Order -100)<br/>Extract Bearer token from header

    GW->>REDIS: GET jwt:denied:{jti}
    REDIS-->>GW: (nil) — token NOT in deny-list

    GW->>GW: JwtService.validateAndExtract(token)<br/>RS256 signature verify<br/>Check exp · iss · aud claims

    Note over GW: Inject enriched headers:<br/>X-Authenticated-UserId: sub claim<br/>X-Authenticated-Roles: roles claim<br/>X-Client-Id: client_id claim

    GW->>REDIS: INCR rl:{clientId}:{60s-window}<br/>EXPIRE rl:{clientId}:{60s-window} 60

    REDIS-->>GW: count = 42 (< 100 limit) ✓

    Note over GW: RequestRateLimiter: token bucket<br/>100 req/s per client-id<br/>Burst: 200

    GW->>GW: Route matching: /api/payments/** → payment-service<br/>CircuitBreaker filter: payment-cb (CLOSED)

    GW->>EUR: GET /eureka/apps/payment-service
    EUR-->>GW: [10.0.1.5:8082, 10.0.1.6:8082, 10.0.1.7:8082]

    Note over GW: Spring Cloud LoadBalancer<br/>Zone-aware round-robin<br/>Select: 10.0.1.6:8082

    GW->>GW: Inject:<br/>X-Correlation-Id: uuid4()<br/>traceparent: 00-{traceId}-{spanId}-01<br/>X-Source-Gateway: fintechbank-gateway

    GW->>PMT: POST /api/payments (with enriched headers)<br/>X-Authenticated-UserId: cust-001

    PMT-->>GW: 201 Created<br/>{"id":"pmt-abc","status":"FRAUD_CHECK"}

    GW-->>MFE: 201 Created<br/>X-Correlation-Id: {uuid}
```

### 1b — Rejected Paths

```mermaid
sequenceDiagram
    autonumber
    participant MFE as Mobile Client
    participant GW as Spring Cloud Gateway :8080
    participant REDIS as Redis 7

    rect rgb(255, 240, 240)
        Note over MFE, REDIS: Path A — Revoked JWT (logged-out token)
        MFE->>GW: POST /api/payments<br/>Authorization: Bearer eyJhbGci... (valid sig but revoked)
        GW->>REDIS: GET jwt:denied:{jti}
        REDIS-->>GW: "1" — token IS in deny-list (logged out)
        GW-->>MFE: 401 Unauthorized<br/>Content-Type: application/problem+json<br/>{"type":"../unauthorized","detail":"Token has been revoked"}
    end

    rect rgb(255, 245, 220)
        Note over MFE, REDIS: Path B — Rate Limit Exceeded
        MFE->>GW: POST /api/payments (101st request in window)
        GW->>REDIS: INCR rl:{clientId}:{window}
        REDIS-->>GW: count = 201 (> burstCapacity 200) ✗
        GW-->>MFE: 429 Too Many Requests<br/>Retry-After: 30<br/>X-Rate-Limit-Remaining: 0
    end

    rect rgb(240, 255, 240)
        Note over MFE, REDIS: Path C — Expired JWT
        MFE->>GW: POST /api/payments<br/>Authorization: Bearer eyJhbGci... (exp in past)
        GW->>GW: JwtService.validateAndExtract(token)<br/>throws JwtException("Token has expired")
        GW-->>MFE: 401 Unauthorized<br/>{"type":"../unauthorized","detail":"Invalid or expired token: Token has expired"}
    end
```

---

## Flow 2: Payment Initiation — Transactional Processing + Kafka Event

> **Scenario:** Authenticated POST reaches payment-service. The service validates, checks idempotency, runs fraud pre-check, persists via `@Transactional`, writes to the outbox table, and emits `payment.initiated` to Kafka using the transactional producer.  
> **Key patterns:** Idempotency guard · `@Transactional` + Kafka transactional producer · Outbox relay · RFC-7807 `ProblemDetail` on fraud rejection · Structured logging with `traceId`.

### 2a — Successful Payment Initiation

```mermaid
sequenceDiagram
    autonumber
    participant GW as Spring Cloud Gateway
    participant PC as PaymentController
    participant PS as PaymentService
    participant IDEM as IdempotencyGuard<br/>(PaymentRepo.findByIdempotencyKey)
    participant FRAUD as FraudPreCheckService
    participant COMP as compliance-service<br/>(Feign / RestClient)
    participant REPO as PaymentRepo
    participant DB as PostgreSQL<br/>payment-db
    participant RELAY as OutboxRelay<br/>(Scheduled)
    participant KAFKA as Apache Kafka<br/>payment.initiated

    GW->>PC: POST /api/payments<br/>@RequestBody @Valid InitiatePaymentRequest<br/>X-Authenticated-UserId: cust-001

    PC->>PC: @PreAuthorize check<br/>req.customerId matches authentication.name ✓

    PC->>PS: initiatePayment(req)

    PS->>IDEM: findByIdempotencyKey("IDEM-001")
    IDEM->>DB: SELECT * FROM payment WHERE idempotency_key = ?
    DB-->>IDEM: empty ResultSet (new request)
    IDEM-->>PS: Optional.empty()

    Note over PS: Begin @Transactional boundary

    PS->>FRAUD: check(customerId, amount, currency)
    FRAUD->>COMP: GET /api/compliance/fraud-pre-check<br/>(Feign client with hystrix timeout 500ms)
    COMP-->>FRAUD: FraudResult.PASS (score: 12/100)
    FRAUD-->>PS: FraudResult.PASS

    PS->>PS: Build Payment entity<br/>status=FRAUD_CHECK · @Version=0

    PS->>REPO: save(payment)
    REPO->>DB: INSERT INTO payment (id, idempotency_key, amount, status, version...)<br/>INSERT INTO outbox_event (id, topic, payload, published=false)
    DB-->>REPO: saved (payment.id=pmt-abc, version=0)
    REPO-->>PS: saved Payment

    Note over PS: End @Transactional — DB commit includes both tables

    PS-->>PC: PaymentDTO(id=pmt-abc, status=FRAUD_CHECK)
    PC-->>GW: 201 Created<br/>Location: /api/payments/pmt-abc

    Note over RELAY: OutboxRelay polls every 500ms
    RELAY->>DB: SELECT * FROM outbox_event<br/>WHERE published = false<br/>ORDER BY created_at<br/>FOR UPDATE SKIP LOCKED (prevents duplicate relay)
    DB-->>RELAY: [outbox_event row: payment.initiated, payload={...}]

    RELAY->>KAFKA: KafkaTemplate.send("payment.initiated", "cust-001", PaymentInitiatedEvent)
    KAFKA-->>RELAY: RecordMetadata (partition=3, offset=1042)

    RELAY->>DB: UPDATE outbox_event SET published=true WHERE id=?
```

### 2b — Duplicate Idempotency Key (Replay Safety)

```mermaid
sequenceDiagram
    autonumber
    participant GW as Spring Cloud Gateway
    participant PC as PaymentController
    participant PS as PaymentService
    participant REPO as PaymentRepo
    participant DB as PostgreSQL

    GW->>PC: POST /api/payments<br/>idempotencyKey: "IDEM-001" (sent before, network retry)

    PC->>PS: initiatePayment(req)

    PS->>REPO: findByIdempotencyKey("IDEM-001")
    REPO->>DB: SELECT * FROM payment WHERE idempotency_key = ?
    DB-->>REPO: Payment(id=pmt-abc, status=AUTHORISED, version=2)
    REPO-->>PS: Optional.of(existingPayment)

    Note over PS: Idempotency guard triggered — return existing result<br/>NO new payment created<br/>NO Kafka event published (prevent duplicate)

    PS-->>PC: PaymentDTO.from(existingPayment)<br/>status=AUTHORISED (idempotent response)

    PC-->>GW: 201 Created<br/>{"id":"pmt-abc","status":"AUTHORISED"}<br/>(same response as original — safe to retry)
```

---

## Flow 3: PCI-DSS Card Tokenisation via Vault

> **Scenario:** Customer submits a payment with card details. payment-service calls HashiCorp Vault Transit Secrets Engine to tokenise the PAN before any persistence. Raw PANs never touch the database or appear in logs — only the token is stored.  
> **Key patterns:** Vault Transit Secrets Engine · PAN → Token (one-way at DB boundary) · Token → PAN only for settlement (secure detokenisation call) · PCI-DSS: no raw card data in logs, DB, or Kafka events.

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile Client<br/>(TLS 1.3)
    participant GW as Spring Cloud Gateway<br/>(PCI-DSS boundary)
    participant PC as PaymentController
    participant TOKEN as CardTokenisationService
    participant VAULT as HashiCorp Vault<br/>Transit Secrets Engine
    participant REPO as PaymentRepo
    participant DB as PostgreSQL<br/>payment-db (PCI scope)<br/>NO raw PANs

    Note over GW: PCI-DSS boundary: RemoveRequestHeader=X-Card-Number<br/>Gateway strips raw card data from forwarded headers

    CLIENT->>GW: POST /api/payments<br/>{"cardDetails":{"pan":"4111111111111111","cvv":"123","expiry":"12/26"}}<br/>TLS 1.3 end-to-end

    GW->>PC: POST /api/payments (JWT verified, rate-limited)<br/>cardDetails still in body — NOT a header

    Note over PC: @Valid validates card format before touching Vault

    PC->>TOKEN: tokenise(CardDetails{pan="4111...", cvv="123"})

    TOKEN->>TOKEN: Build Vault API request<br/>Mask PAN for logging: "411111*****1111"<br/>NEVER log raw PAN or CVV

    TOKEN->>VAULT: POST /v1/transit/encrypt/card-key<br/>{"plaintext": base64(PAN)}<br/>Auth: X-Vault-Token (K8s service account injected)

    VAULT->>VAULT: AES-256-GCM encryption<br/>Key ID: card-key/v3 (versioned)

    VAULT-->>TOKEN: {"ciphertext":"vault:v3:ZmluZ...","key_version":3}

    TOKEN->>TOKEN: card_token = SHA256(ciphertext)[0:32]<br/>Discard raw PAN from memory<br/>CVV is NEVER stored or tokenised (PCI-DSS req 3.2)

    TOKEN-->>PC: CardToken("tok_4111x1111", keyVersion=3)

    Note over PC, DB: PAN leaves memory — only token persists from here

    PC->>REPO: save(Payment{cardToken="tok_4111x1111", cvv=null, pan=null})
    REPO->>DB: INSERT INTO payment (card_token, ...) — NO raw PAN column

    DB-->>REPO: saved

    Note over TOKEN, VAULT: Settlement path — detokenise only when submitting to card network
    TOKEN->>VAULT: POST /v1/transit/decrypt/card-key<br/>{"ciphertext":"vault:v3:ZmluZ..."}<br/>Requires SETTLEMENT_ROLE (separate Vault policy)
    VAULT-->>TOKEN: {"plaintext": base64(originalPAN)}<br/>PAN reconstructed in memory for settlement only
    Note over TOKEN: PAN sent to card network · immediately zeroed from memory
```

---

## Flow 4: Account Service — Redis Cache-Aside (HIT and MISS Paths)

> **Scenario:** Client requests account balance. The account-service first checks Redis. On a cache hit, it returns immediately. On a cache miss, it queries PostgreSQL, caches the result, and returns. Write operations invalidate the cache.  
> **Key patterns:** Cache-aside (Lazy Loading) · `@Version` optimistic locking for concurrent writes · Redis key `account:balance:{id}` TTL=30s · Cache invalidation-on-write.

### 4a — Cache HIT Path

```mermaid
sequenceDiagram
    autonumber
    participant GW as Spring Cloud Gateway
    participant AC as AccountController
    participant AS as AccountService
    participant REDIS as Redis 7<br/>account:balance:{id}
    participant REPO as AccountRepo
    participant DB as PostgreSQL<br/>account-db

    GW->>AC: GET /api/accounts/{id}/balance<br/>X-Authenticated-UserId: cust-001

    AC->>AC: @PreAuthorize: id matches authenticated user ✓<br/>or role ADVISOR/ADMIN

    AC->>AS: getBalance(accountId)

    AS->>REDIS: GET account:balance:{id}
    REDIS-->>AS: "1500.00" ← CACHE HIT

    Note over AS: Cache hit — no DB query needed<br/>P99 latency: ~1ms vs ~5ms DB

    AS-->>AC: BigDecimal("1500.00")
    AC-->>GW: 200 OK<br/>{"accountId":"...","balance":1500.00,"currency":"GBP","cached":true}

    Note over GW: X-Cache: HIT (observability header for debugging)
```

### 4b — Cache MISS Path (DB fallback + cache population)

```mermaid
sequenceDiagram
    autonumber
    participant GW as Spring Cloud Gateway
    participant AC as AccountController
    participant AS as AccountService
    participant REDIS as Redis 7
    participant REPO as AccountRepo
    participant DB as PostgreSQL<br/>account-db

    GW->>AC: GET /api/accounts/{id}/balance (first request or after TTL expiry)

    AC->>AS: getBalance(accountId)

    AS->>REDIS: GET account:balance:{id}
    REDIS-->>AS: (nil) ← CACHE MISS

    Note over AS: Cache miss — fall through to PostgreSQL

    AS->>REPO: findById(accountId)
    REPO->>DB: SELECT a.balance FROM account a WHERE a.id = ?<br/>@Transactional(readOnly = true) — no write lock
    DB-->>REPO: Account{balance=1500.00, version=7}
    REPO-->>AS: Optional<Account>

    AS->>REDIS: SET account:balance:{id} "1500.00" EX 30
    REDIS-->>AS: OK

    AS-->>AC: BigDecimal("1500.00")
    AC-->>GW: 200 OK<br/>{"accountId":"...","balance":1500.00,"currency":"GBP","cached":false}

    Note over GW: X-Cache: MISS
```

### 4c — Write Path (balance update → cache invalidation)

```mermaid
sequenceDiagram
    autonumber
    participant PS as payment-service<br/>(Kafka consumer: payment.completed)
    participant AS as AccountService
    participant REPO as AccountRepo
    participant DB as PostgreSQL<br/>account-db
    participant REDIS as Redis 7

    Note over PS: Kafka consumer receives payment.completed event<br/>→ debit/credit account balance

    PS->>AS: updateBalance(accountId, delta=-100.00)

    Note over AS: Begin @Transactional (read-write)

    AS->>REPO: findByIdForUpdate(accountId)
    REPO->>DB: SELECT * FROM account WHERE id = ?<br/>FOR UPDATE (pessimistic lock for balance update)
    DB-->>REPO: Account{balance=1500.00, version=7}
    REPO-->>AS: account

    AS->>AS: account.applyDelta(-100.00)<br/>→ balance = 1400.00<br/>@Version incremented to 8

    AS->>REPO: save(account)
    REPO->>DB: UPDATE account SET balance=1400.00, version=8 WHERE id=? AND version=7
    DB-->>REPO: 1 row updated (version match ✓)

    Note over AS: Commit @Transactional

    AS->>REDIS: DEL account:balance:{id} ← invalidate stale cache
    REDIS-->>AS: 1 (key deleted)

    Note over AS: Next GET will be a cache MISS → repopulate with 1400.00

    AS-->>PS: void (success — balance updated and cache invalidated)
```

---

## Flow 5: Kafka Event-Driven Compliance / KYC Check

> **Scenario:** payment-service publishes `payment.initiated` to Kafka. compliance-service consumes from `compliance.group`, runs AML risk scoring, and re-publishes `kyc.passed` or `kyc.flagged`. payment-service reacts to the KYC outcome to advance or block the payment.  
> **Key patterns:** Kafka consumer group manual ack · Dead Letter Topic on retry exhaustion · `isolation.level=read_committed` (EOS consumer) · MDC traceId propagation for correlated logging · `@Version` optimistic locking on compliance case entity.

### 5a — KYC Pass Flow

```mermaid
sequenceDiagram
    autonumber
    participant PMT_SVC as payment-service<br/>(Kafka Producer)
    participant KAFKA as Apache Kafka<br/>payment.initiated
    participant COMP_SVC as compliance-service<br/>Consumer group: compliance.group
    participant RISK as RiskEngine
    participant SCREEN as SanctionsScreeningClient<br/>(Refinitiv WorldCheck)
    participant COMP_DB as PostgreSQL<br/>compliance-db
    participant KAFKA_KYC as Apache Kafka<br/>kyc.passed
    participant PMT_CONS as payment-service<br/>Consumer group: kyc.result.group

    PMT_SVC->>KAFKA: ProducerRecord{topic="payment.initiated",<br/>key="cust-001",<br/>value=PaymentInitiatedEvent{amount=500,currency=GBP,customerId=cust-001},<br/>headers={traceparent: 00-abc123-def456-01}}

    Note over KAFKA: Partition assignment: partition=hash(cust-001)%12<br/>All events for same customer → same partition → ordered

    KAFKA->>COMP_SVC: poll() → ConsumerRecord<br/>topic=payment.initiated, partition=3, offset=1042<br/>key=cust-001, value=PaymentInitiatedEvent

    COMP_SVC->>COMP_SVC: Extract W3C traceparent header<br/>MDC.put("traceId", "abc123") — correlated logging

    COMP_SVC->>RISK: assess(PaymentInitiatedEvent)

    RISK->>SCREEN: screenCustomer(customerId="cust-001")
    SCREEN-->>RISK: SanctionsResult{match=false, checked_at=...}

    RISK->>RISK: Velocity check: 3 payments in last 24h (< limit 10)<br/>Amount score: 500 GBP (< daily limit 5000)<br/>Geography: GB→GB (low risk)<br/>→ riskScore = 12/100

    RISK-->>COMP_SVC: RiskAssessment{outcome=PASS, score=12, factors=[...]}

    COMP_SVC->>COMP_DB: INSERT INTO compliance_check<br/>(payment_id, outcome=PASS, score=12, details=JSON)
    COMP_DB-->>COMP_SVC: saved

    COMP_SVC->>KAFKA_KYC: send("kyc.passed", "cust-001",<br/>KycPassedEvent{paymentId, customerId, score=12})

    COMP_SVC->>COMP_SVC: consumer.commitSync() ← manual ack after successful processing

    KAFKA_KYC->>PMT_CONS: poll() ConsumerRecord{kyc.passed, paymentId=pmt-abc}

    PMT_CONS->>PMT_CONS: paymentService.advanceToAuthorised(paymentId)
    Note over PMT_CONS: UPDATE payment SET status=AUTHORISED, version=version+1<br/>WHERE id=? AND version=? (optimistic lock)<br/>Emit payment.completed → Kafka
```

### 5b — KYC Block Flow (SAR Filing)

```mermaid
sequenceDiagram
    autonumber
    participant KAFKA as Apache Kafka<br/>payment.initiated
    participant COMP_SVC as compliance-service
    participant RISK as RiskEngine
    participant SAR as SarFilingService
    participant COMP_DB as PostgreSQL<br/>compliance-db
    participant KAFKA_FLAG as Apache Kafka<br/>kyc.flagged + audit.trail.sar
    participant PMT_CONS as payment-service consumer
    participant DLT as Dead Letter Topic<br/>payment.initiated.DLT

    KAFKA->>COMP_SVC: ConsumerRecord{payment.initiated, amount=95000 GBP}

    COMP_SVC->>RISK: assess(event)
    RISK-->>COMP_SVC: RiskAssessment{outcome=SAR, score=98,<br/>factors=["AMOUNT_EXCEEDS_THRESHOLD","RAPID_SUCCESSION","SANCTIONS_PARTIAL_MATCH"]}

    COMP_SVC->>SAR: fileReport(event, assessment)
    SAR->>COMP_DB: INSERT INTO sar_report<br/>(customer_id, payment_id, risk_factors, status=FILED_PENDING_REVIEW)
    SAR->>KAFKA_FLAG: send("audit.trail.sar", SarFiledEvent)<br/>retention=7years (regulatory)
    COMP_SVC->>KAFKA_FLAG: send("kyc.flagged", "cust-001",<br/>KycFlaggedEvent{paymentId, reason=SAR_FILED, blockPayment=true})

    COMP_SVC->>COMP_SVC: consumer.commitSync()

    KAFKA_FLAG->>PMT_CONS: ConsumerRecord{kyc.flagged, blockPayment=true}
    PMT_CONS->>PMT_CONS: paymentService.blockPayment(paymentId)<br/>UPDATE payment SET status=BLOCKED<br/>Emit payment.failed → Kafka (for notification-service)

    Note over DLT: If compliance-service crashes during processing<br/>DefaultErrorHandler retries 3× with 1s backoff<br/>→ publishes to payment.initiated.DLT for manual review
```

---

## Flow 6: Trading Service — Order Placement, Optimistic Locking, MiFID II Audit

> **Scenario:** Customer places a buy order for FTSE 100 equity. trading-service validates funds via account-service, persists the order with `@Version`, emits a MiFID II Transaction Report to Kafka, and handles the case where two concurrent execution attempts collide via optimistic locking.  
> **Key patterns:** `@Version` optimistic locking · MiFID II LEI-based order ID · Kafka append-only audit trail (7-year retention) · Feign client resilience (circuit breaker + retry) · Optimistic lock collision recovery.

### 6a — Successful Order Placement

```mermaid
sequenceDiagram
    autonumber
    participant GW as Spring Cloud Gateway
    participant TC as TradingController
    participant TS as TradingService
    participant ACCT_CLIENT as account-service FeignClient
    participant ACCT_SVC as account-service :8081
    participant REPO as OrderRepo
    participant DB as PostgreSQL<br/>trading-db
    participant MIFID as MifidReportService
    participant KAFKA_TRADE as Apache Kafka<br/>trade.order.placed
    participant KAFKA_AUDIT as Apache Kafka<br/>audit.trail.mifid

    GW->>TC: POST /api/trading/orders<br/>{"customerId":"cust-001","instrument":"GB0001383545",<br/>"side":"BUY","quantity":10,"limitPrice":7850.00}<br/>X-Authenticated-UserId: cust-001

    TC->>TC: @PreAuthorize: customerId matches principal ✓

    TC->>TS: placeOrder(PlaceOrderRequest)

    TS->>ACCT_CLIENT: validateFunds(accountId, estimatedCost=78500.00 GBP)
    Note over ACCT_CLIENT: Feign client: @CircuitBreaker(name="account-service")<br/>Timeout: 1000ms · Retry: 2× on 5xx

    ACCT_CLIENT->>ACCT_SVC: GET /api/accounts/{id}/margin-check?amount=78500
    ACCT_SVC-->>ACCT_CLIENT: 200 OK {"eligible":true,"availableFunds":150000.00}
    ACCT_CLIENT-->>TS: FundsValidationResult{eligible=true}

    TS->>TS: Generate MiFID II LEI order ID<br/>mifidOrderId = LEI(firm) + "-" + UUID

    TS->>REPO: save(Order{..., status=PENDING, version=0, mifidOrderId=...})
    REPO->>DB: INSERT INTO trade_order (id, customer_id, instrument,<br/>side, quantity, limit_price, status, version=0, mifid_order_id)
    DB-->>REPO: Order{id=ord-xyz, version=0}

    REPO-->>TS: saved Order

    TS->>MIFID: generateOrderPlacedReport(order)
    MIFID->>KAFKA_AUDIT: send("audit.trail.mifid", MifidOrderReport{<br/>reportType=ORDER_PLACED,<br/>mifidOrderId=..., lei=..., instrument="GB0001383545",<br/>reportedAt=Instant.now()})<br/>retention=7 years — immutable

    TS->>KAFKA_TRADE: send("trade.order.placed", "cust-001", TradeOrderPlacedEvent{orderId, instrument,...})

    TS-->>TC: OrderDTO{id=ord-xyz, status=PENDING, mifidOrderId=...}
    TC-->>GW: 201 Created<br/>Location: /api/trading/orders/ord-xyz

    Note over GW: MiFID II: order must be reported within T+1 ✓ (async via Kafka)
```

### 6b — Optimistic Locking Collision on Order Execution

```mermaid
sequenceDiagram
    autonumber
    participant EXEC_A as Market Execution Service<br/>Thread A (e.g., direct execution)
    participant EXEC_B as Market Execution Service<br/>Thread B (e.g., retry on error)
    participant TS as TradingService.executeOrder()
    participant REPO as OrderRepo
    participant DB as PostgreSQL<br/>trade_order table

    Note over EXEC_A, EXEC_B: Two execution messages for same order arrive concurrently<br/>(network partition recovery scenario)

    par Thread A
        EXEC_A->>TS: executeOrder(orderId="ord-xyz", execution={price=7848, qty=10})
        TS->>REPO: findByIdForUpdate("ord-xyz")
        REPO->>DB: SELECT * FROM trade_order WHERE id=? FOR UPDATE
        DB-->>REPO: Order{id=ord-xyz, status=PENDING, version=0}
    and Thread B (milliseconds later, before A commits)
        EXEC_B->>TS: executeOrder(orderId="ord-xyz", execution={price=7850, qty=10})
        TS->>REPO: findByIdForUpdate("ord-xyz")
        REPO->>DB: SELECT * FROM trade_order WHERE id=? FOR UPDATE
        DB-->>REPO: Order{id=ord-xyz, status=PENDING, version=0} (same version — both read before A writes)
    end

    EXEC_A->>TS: order.execute(execution) → status=EXECUTED, version will be 1
    TS->>REPO: save(order) → version=0 → 1
    REPO->>DB: UPDATE trade_order SET status=EXECUTED, version=1 WHERE id=? AND version=0
    DB-->>REPO: 1 row updated ✓ — Thread A commits

    Note over DB: DB version is now 1

    EXEC_B->>TS: order.execute(execution) → status=EXECUTED, version still 0 (stale)
    TS->>REPO: save(order)
    REPO->>DB: UPDATE trade_order SET status=EXECUTED, version=1 WHERE id=? AND version=0
    DB-->>REPO: 0 rows updated ✗ (version mismatch — DB is now version=1)

    REPO->>TS: throws ObjectOptimisticLockingFailureException

    TS->>TS: @Retryable catches exception<br/>Reload order: findById → status=EXECUTED<br/>throw OrderAlreadyProcessedException

    TS-->>EXEC_B: OrderAlreadyProcessedException{orderId, currentStatus=EXECUTED}<br/>← safe: duplicate execution prevented by @Version

    Note over EXEC_A, EXEC_B: Result: Order executed exactly once.<br/>MiFID II audit report generated only once by Thread A.
```

---

## Flow 7: OAuth2 JWT Resource Server Validation Chain

> **Scenario:** An incoming request carries a Bearer JWT. The microservice (running as OAuth2 Resource Server) validates the token without calling auth-service — it fetches the JWKS public key once, caches it, and verifies locally. Roles are extracted and placed into the Spring `SecurityContext`.  
> **Key patterns:** `spring-security-oauth2-resource-server` · JWKS public key caching (5min TTL) · RS256 signature verify · `JwtGrantedAuthoritiesConverter` · `SecurityContext` population for `@PreAuthorize` · PEM rotation safe (multi-key JWKS).

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Calling Service / MFE<br/>(Bearer JWT)
    participant FILTER as Spring Security Filter Chain<br/>BearerTokenAuthenticationFilter
    participant PM as JwtAuthenticationProvider
    participant DECODER as NimbusJwtDecoder<br/>(cached JWKS)
    participant AUTH_SVC as auth-service :8085<br/>/.well-known/jwks.json
    participant CTX as SecurityContextHolder<br/>(ThreadLocal / VirtualThread)
    participant CONTROLLER as @RestController + @PreAuthorize

    CLIENT->>FILTER: HTTP Request<br/>Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

    FILTER->>FILTER: Extract Bearer token<br/>BearerTokenExtractor.extract(request)

    FILTER->>PM: authenticate(BearerTokenAuthenticationToken(token))

    PM->>DECODER: decode(token)

    alt JWKS cache VALID (< 5 min old)
        DECODER->>DECODER: Use cached public key (JWK set)
        Note over DECODER: No network call — sub-millisecond key lookup
    else JWKS cache EXPIRED or first request
        DECODER->>AUTH_SVC: GET /.well-known/jwks.json<br/>(configured via spring.security.oauth2.resourceserver.jwt.jwk-set-uri)
        AUTH_SVC-->>DECODER: {"keys":[{"kty":"RSA","kid":"key-v3","n":"...","e":"AQAB"},...]}
        DECODER->>DECODER: Cache JWK set (5 min TTL)<br/>Multi-key JWKS supports key rotation — find by kid header
    end

    DECODER->>DECODER: Find key by kid="key-v3" (from JWT header)<br/>RS256 signature verify using RSA public key

    DECODER->>DECODER: Validate standard claims:<br/>exp: must be in future ✓<br/>iss: must equal https://auth.fintechbank.com ✓<br/>aud: must include resource-server-id ✓<br/>nbf: if present, must be in past ✓

    DECODER->>DECODER: Build Jwt object with all claims

    DECODER-->>PM: Jwt{sub="cust-001", roles=["CUSTOMER"],<br/>client_id="mfe-shell", exp=..., jti="uuid4"}

    PM->>PM: JwtGrantedAuthoritiesConverter<br/>Extract "roles" claim → ["ROLE_CUSTOMER"]<br/>Build JwtAuthenticationToken

    PM-->>FILTER: JwtAuthenticationToken{principal="cust-001", authorities=["ROLE_CUSTOMER"]}

    FILTER->>CTX: SecurityContextHolder.setContext(<br/>SecurityContextImpl(JwtAuthenticationToken))

    Note over CTX: Virtual Thread: SecurityContext bound to thread via<br/>DelegatingSecurityContextExecutor — propagates across async boundaries

    FILTER->>CONTROLLER: Continue filter chain → reach @RestController

    CONTROLLER->>CONTROLLER: @PreAuthorize("hasRole('CUSTOMER') and<br/>#req.customerId().toString() == authentication.name")<br/>authentication.name = "cust-001" ✓<br/>req.customerId = "cust-001" ✓

    CONTROLLER-->>CLIENT: 200 OK (authorised response)

    alt Token expired (exp in past)
        DECODER->>PM: throws JwtValidationException("Token has expired")
        PM->>FILTER: throw AuthenticationException
        FILTER-->>CLIENT: 401 Unauthorized<br/>WWW-Authenticate: Bearer error="invalid_token", error_description="Token expired"
    end

    alt Token audience mismatch
        DECODER->>PM: throws JwtValidationException("aud claim not valid")
        FILTER-->>CLIENT: 401 Unauthorized<br/>error="invalid_token", error_description="aud claim is not valid"
    end
```

---

## Flow 8: Testcontainers Integration Test Lifecycle

> **Scenario:** A QE runs the payment-service integration test suite. Testcontainers spins up real PostgreSQL 16, Kafka, and Redis containers via Docker, `@ServiceConnection` auto-wires Spring datasource/kafka/redis properties, Liquibase runs migrations, and the tests execute against real infrastructure before teardown.  
> **Key patterns:** `@Testcontainers` + `@SpringBootTest` colocated · `@ServiceConnection` (Spring Boot 3.1+) auto-maps container ports · `@Rollback` per test · `SKIP LOCKED` idempotency in outbox tests · Parallel test execution via thread-isolated containers.

```mermaid
sequenceDiagram
    autonumber
    participant JVM as JUnit 5 / Maven Failsafe Plugin
    participant TC as Testcontainers Framework
    participant DOCKER as Docker Engine<br/>(local / CI)
    participant PG_CTR as PostgreSQLContainer<br/>postgres:16
    participant KAFKA_CTR as KafkaContainer<br/>confluentinc/cp-kafka:7.6.0
    participant REDIS_CTR as RedisContainer<br/>redis:7-alpine
    participant SPRING as Spring ApplicationContext<br/>@SpringBootTest
    participant LB as Liquibase<br/>(schema migration)
    participant TEST as @Test Methods

    JVM->>TC: @Testcontainers annotation triggers lifecycle

    par Start containers (parallel)
        TC->>DOCKER: Pull + Start postgres:16
        DOCKER-->>TC: PostgreSQLContainer running<br/>host=localhost, port=49152 (ephemeral)
        TC->>DOCKER: Pull + Start confluentinc/cp-kafka:7.6.0
        DOCKER-->>TC: KafkaContainer running<br/>bootstrap=localhost:49153
        TC->>DOCKER: Pull + Start redis:7-alpine
        DOCKER-->>TC: RedisContainer running<br/>host=localhost, port=49154
    end

    Note over TC, SPRING: @ServiceConnection reads container metadata<br/>Auto-configures Spring properties:<br/>spring.datasource.url = jdbc:postgresql://localhost:49152/test<br/>spring.kafka.bootstrap-servers = localhost:49153<br/>spring.data.redis.host = localhost, port=49154<br/>No manual @DynamicPropertySource needed

    TC->>SPRING: Start Spring ApplicationContext with @ServiceConnection properties

    SPRING->>LB: Run Liquibase migrations against PostgreSQL container
    LB->>PG_CTR: CREATE TABLE payment (...)<br/>CREATE TABLE outbox_event (...)<br/>CREATE UNIQUE INDEX idx_idempotency_key (...)
    PG_CTR-->>LB: migrations applied ✓

    SPRING->>SPRING: All beans initialised — PaymentService, PaymentRepo,<br/>KafkaTemplate, RedisTemplate ready against real containers

    SPRING-->>JVM: ApplicationContext ready

    Note over JVM, TEST: Test execution begins — each @Test is @Transactional<br/>by default → rolled back after each test → clean state

    JVM->>TEST: Run: paymentLifecycle_happyPath()

    TEST->>SPRING: paymentService.initiatePayment(buildRequest("IDEM-001"))
    SPRING->>PG_CTR: INSERT INTO payment (...)<br/>INSERT INTO outbox_event (...)
    PG_CTR-->>SPRING: saved (2 rows)

    SPRING->>KAFKA_CTR: KafkaTemplate.send("payment.initiated", event)<br/>(via OutboxRelay triggered in test)
    KAFKA_CTR-->>SPRING: RecordMetadata ✓

    TEST->>TEST: await().atMost(5, SECONDS).untilAsserted(...)
    Note over TEST: Awaitility waits for Kafka consumer to process event

    SPRING->>KAFKA_CTR: @KafkaListener consumer polls payment.initiated
    KAFKA_CTR-->>SPRING: ConsumerRecord delivered

    TEST->>PG_CTR: assertThat(paymentRepo.findById(id)).isPresent()
    PG_CTR-->>TEST: Optional<Payment> ✓

    TEST-->>JVM: Test PASSED ✓

    Note over JVM, PG_CTR: @Transactional rollback — DB state restored for next test

    JVM->>TEST: Run: paymentUpdate_concurrentModification_throwsOptimisticLockException()

    TEST->>PG_CTR: save(buildPayment()) → payment with version=0
    TEST->>PG_CTR: copy1 = findById() → version=0
    TEST->>PG_CTR: copy2 = findById() → version=0
    TEST->>PG_CTR: saveAndFlush(copy1.setStatus(AUTHORISED)) → version=1 ✓
    TEST->>PG_CTR: saveAndFlush(copy2.setStatus(FAILED)) → version mismatch!
    PG_CTR-->>TEST: throws ObjectOptimisticLockingFailureException
    TEST->>TEST: assertThatThrownBy(...).isInstanceOf(ObjectOptimisticLockingFailureException.class) ✓

    TEST-->>JVM: Test PASSED ✓

    JVM->>TC: All tests complete — stop containers

    par Stop containers (parallel)
        TC->>DOCKER: Stop + remove PostgreSQLContainer
        TC->>DOCKER: Stop + remove KafkaContainer
        TC->>DOCKER: Stop + remove RedisContainer
    end

    DOCKER-->>TC: All containers removed (ephemeral — no leftover state)
    TC-->>JVM: Testcontainers lifecycle complete
```

---

## Flow 9: OpenTelemetry Distributed Tracing Across Services

> **Scenario:** A payment initiation request generates a single distributed trace that spans API Gateway, payment-service, Kafka producer, and compliance-service consumer. All spans share the same `traceId` via W3C Trace Context headers, enabling end-to-end request correlation in Grafana/Tempo.  
> **Key patterns:** W3C Trace Context (`traceparent`) header propagation · OpenTelemetry OTLP export · `micrometer-tracing` + `opentelemetry-spring-boot-starter` · MDC `traceId`/`spanId` in structured JSON logs · Kafka header propagation via `ObservationRegistry`.

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile Client
    participant GW as Spring Cloud Gateway<br/>Span: gateway.payment.initiate
    participant OTEL_COLL as OpenTelemetry Collector<br/>(sidecar)
    participant PMT_SVC as payment-service<br/>Span: payment.initiate
    participant KAFKA_PROD as Apache Kafka Producer<br/>Span: kafka.send.payment.initiated
    participant KAFKA_BROKER as Apache Kafka Broker
    participant COMP_SVC as compliance-service<br/>Span: compliance.kyc.check
    participant TEMPO as Grafana Tempo<br/>(trace backend)

    CLIENT->>GW: POST /api/payments<br/>(no traceparent — new trace)

    GW->>GW: OTel auto-instrumentation creates ROOT SPAN<br/>traceId=abc123def456<br/>spanId=gateway-001<br/>traceparent=00-abc123def456-gateway001-01

    GW->>PMT_SVC: Forward with headers:<br/>traceparent: 00-abc123def456-gateway001-01

    Note over GW: OTel Collector sidecar intercepts span export<br/>Export: ROOT SPAN (gateway) → OTLP → Tempo

    PMT_SVC->>PMT_SVC: OTel extracts traceparent header<br/>Create CHILD SPAN:<br/>traceId=abc123def456 (same!)<br/>spanId=payment-002<br/>parentSpanId=gateway-001<br/>MDC: traceId=abc123def456, spanId=payment-002

    Note over PMT_SVC: All log lines include traceId=abc123def456<br/>{"level":"INFO","traceId":"abc123def456","spanId":"payment-002",<br/>"message":"Initiating payment for customer cust-001"}

    PMT_SVC->>PMT_SVC: Business logic: fraud check, DB write, outbox write

    PMT_SVC->>KAFKA_BROKER: KafkaTemplate.send("payment.initiated", event)<br/>KafkaTemplate has ObservationEnabled=true<br/>Kafka record headers include:<br/>traceparent: 00-abc123def456-kafka-003-01

    Note over PMT_SVC: CHILD SPAN: kafka.send.payment.initiated<br/>traceId=abc123def456, spanId=kafka-003<br/>parentSpanId=payment-002

    Note over PMT_SVC: Export CHILD SPANS (payment + kafka.send) → OTLP → Tempo

    KAFKA_BROKER-->>PMT_SVC: Ack (partition=3, offset=1042)
    PMT_SVC-->>GW: 201 Created
    GW-->>CLIENT: 201 Created<br/>X-Trace-Id: abc123def456

    Note over KAFKA_BROKER, COMP_SVC: Asynchronous boundary — trace continues via Kafka headers

    KAFKA_BROKER->>COMP_SVC: poll() ConsumerRecord<br/>value=PaymentInitiatedEvent<br/>headers=[traceparent: 00-abc123def456-kafka-003-01]

    COMP_SVC->>COMP_SVC: OTel Kafka consumer instrumentation<br/>Extract traceparent from record headers<br/>Create CHILD SPAN:<br/>traceId=abc123def456 (same trace!)<br/>spanId=comp-004<br/>parentSpanId=kafka-003<br/>MDC: traceId=abc123def456, spanId=comp-004

    Note over COMP_SVC: All compliance logs correlated with same traceId<br/>{"traceId":"abc123def456","spanId":"comp-004",<br/>"message":"KYC assessment started for customer cust-001"}

    COMP_SVC->>COMP_SVC: Risk engine + sanctions screening

    Note over COMP_SVC: Export CHILD SPAN (compliance.kyc.check) → OTLP → Tempo

    OTEL_COLL->>TEMPO: Batch export all spans:<br/>[gateway.payment.initiate (root),<br/> payment.initiate (child),<br/> kafka.send.payment.initiated (child),<br/> compliance.kyc.check (child)]<br/>All linked by traceId=abc123def456

    Note over TEMPO: Full trace reconstructed in Grafana:<br/>Timeline: GW(12ms) → Payment(45ms) → Kafka(2ms) → Compliance(380ms)<br/>Total: 439ms end-to-end
```

---

## Flow 10: Microservice Canary Deploy — Kubernetes Rolling Update + Argo Rollouts

> **Scenario:** CI/CD pipeline publishes a new payment-service image (v2.3.1). GitHub Actions triggers an Argo Rollouts canary deploy: 10% of traffic to the new version, Prometheus metric gate checks error rate and P99 latency, then progressively promotes to 100% or auto-rollback on threshold breach.  
> **Key patterns:** Argo Rollouts `Rollout` CRD · `AnalysisTemplate` metrics gate · Weighted traffic split · Automated rollback on SLO breach · `readinessProbe` health gate before traffic shift · `PodDisruptionBudget` maintains availability.

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub Actions<br/>CI/CD Pipeline
    participant GHCR as Container Registry<br/>registry.fintechbank.com
    participant HELM as Helm<br/>helm upgrade
    participant ARGO as Argo Rollouts Controller<br/>(K8s custom controller)
    participant K8S as Kubernetes API<br/>EKS / AKS
    participant PROM as Prometheus<br/>(metrics store)
    participant NOTIFY as Slack / PagerDuty

    GH->>GH: Unit tests ✓ · Integration tests ✓<br/>OWASP check ✓ · Pact verification ✓

    GH->>GHCR: docker push payment-service:v2.3.1
    GHCR-->>GH: Image digest: sha256:abc...

    GH->>HELM: helm upgrade payment-service ./helm/payment-service<br/>--set image.tag=v2.3.1<br/>--set rollout.strategy=canary<br/>--namespace production

    HELM->>K8S: Update Argo Rollout CRD (kind: Rollout)<br/>spec.template.spec.containers[0].image=payment-service:v2.3.1

    K8S->>ARGO: Rollout resource updated — begin canary strategy

    Note over ARGO: Canary steps:<br/>1. setWeight: 10% (1 of 10 pods)<br/>2. analysis: wait 5 min + check metrics<br/>3. setWeight: 30%, pause: 5m<br/>4. setWeight: 50%, analysis again<br/>5. setWeight: 100% (full promotion)

    ARGO->>K8S: Create canary ReplicaSet (v2.3.1) — 1 replica
    K8S-->>ARGO: ReplicaSet created

    Note over K8S: readinessProbe: GET /actuator/health/readiness<br/>New pod must pass probe before receiving ANY traffic

    K8S->>K8S: payment-service-v2-pod-1<br/>readinessProbe: /actuator/health/readiness → 200 OK ✓<br/>→ pod marked READY

    ARGO->>K8S: Shift 10% traffic to canary<br/>(update Kubernetes Service weighted endpoints<br/> or Istio VirtualService weight=10)

    Note over ARGO: Analysis begins — 5 minute observation window

    loop every 30s for 5 minutes
        ARGO->>PROM: AnalysisTemplate metric query:<br/>rate(http_server_requests_seconds_count{status=~"5xx",version="v2.3.1"}[2m])<br/>/ rate(http_server_requests_seconds_count{version="v2.3.1"}[2m])
        PROM-->>ARGO: error_rate = 0.002 (0.2% < 1% threshold ✓)

        ARGO->>PROM: histogram_quantile(0.99,<br/>rate(http_server_requests_seconds_bucket{version="v2.3.1"}[2m]))
        PROM-->>ARGO: p99 = 0.85s (< 2s threshold ✓)
    end

    ARGO->>ARGO: AnalysisRun result: SUCCESS (all metrics within thresholds)

    ARGO->>K8S: setWeight: 30% — scale canary replicas to 3

    Note over ARGO, K8S: Progressive traffic shift continues: 30% → 50% → 100%<br/>Each step has 5-minute analysis gate

    ARGO->>K8S: setWeight: 100% — promote canary to stable
    K8S->>K8S: Update stable ReplicaSet image to v2.3.1<br/>Scale down old (v2.3.0) ReplicaSet to 0

    ARGO->>NOTIFY: Slack: ✅ payment-service v2.3.1 fully promoted to production<br/>Canary duration: 25 minutes · Zero errors during rollout

    Note over GH, NOTIFY: ⚠️ AUTO-ROLLBACK SCENARIO (if metrics breach threshold)

    rect rgb(255, 240, 240)
        ARGO->>PROM: error_rate = 0.035 (3.5% ❌ > 1% threshold)
        ARGO->>ARGO: AnalysisRun result: FAILED<br/>Immediate rollback triggered
        ARGO->>K8S: setWeight: 0% canary<br/>Scale down v2.3.1 ReplicaSet to 0<br/>Traffic returns 100% to v2.3.0 (stable)
        ARGO->>NOTIFY: PagerDuty + Slack: 🔴 payment-service v2.3.1 ROLLED BACK<br/>error_rate=3.5% exceeded threshold 1%<br/>Stable v2.3.0 restored — zero customer impact
    end
```

---

*Generated 2025 · Digital Banking & Wealth Platform — Back-End Sequence Diagrams Reference*  
*Stack: Java 21 · Spring Boot 3.3 · Spring Cloud 2023 · Apache Kafka · PostgreSQL 16 · Redis 7 · Kubernetes*  
*Regulatory scope: PCI-DSS Level 1 · SOC 2 Type II · PSD2 · MiFID II*  
*Perspective: Principal Back-End Engineer · Solution Architect · Data Engineer · QE*  
*Flows: 10 — Gateway Security · Payment Pipeline · PCI Tokenisation · Redis Cache-Aside · Kafka KYC · Trading Locking · OAuth2 JWT · Testcontainers · Distributed Tracing · Canary Deploy*
