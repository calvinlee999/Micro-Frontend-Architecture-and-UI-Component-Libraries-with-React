# FinTech Enterprise Micro-Frontend Architecture — Sequence Diagrams

> **Platform:** Digital Banking & Wealth Platform · Webpack Module Federation · React 18 · Storybook 8 · TypeScript  
> **Perspective:** Principal Front-End Solution Architect · Principal Front-End Quality Engineer  
> **Regulatory scope:** PCI-DSS Level 1 · SOC 2 Type II · PSD2 / Open Banking · WCAG 2.1 AA  
> **Ordering:** Flows are aligned 1-to-1 with §5 Layer Summary Tables in [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Diagram Index

| # | Flow | Trigger | Layers Covered | §5 Alignment |
|---|---|---|---|---|
| 1 | Shell bootstrap + OAuth2 PKCE auth + Module Federation negotiation | First page load | Browser → Shell entry → IdP → MF runtime → AuthContext | §5.0 Shell |
| 2 | Dashboard MFE lazy load with auth guard | Navigate to `/dashboard` | Shell → AuthGuard → remoteEntry.js → DashboardApp | §5.1 Dashboard |
| 3 | Design System component resolution (Storybook → npm → MFE) | MFE renders `<Button>` | Storybook → Chromatic → npm publish → MFE build | §3 Design System |
| 4 | Cross-MFE payment initiation with PCI-DSS boundary + audit trail | User initiates payment | Payments MFE → PCI iframe → PCI vault → Payment API → Audit | §5.2 Payments |
| 5 | Feature flag evaluation for new trading instrument rollout | User opens Trading MFE | LaunchDarkly → feature flag client → conditional render | §5.3 Trading |
| 6 | Shared singleton version negotiation + auth context reuse | Dashboard MFE loads | MF runtime → shared scope → auth-context singleton | §5.5 Shared Modules |
| 7 | Canary deployment + automated metric gate + rollback | CI/CD push to `payments` | GitHub Actions → webpack → CDN canary → metric gate | §5.6 Infrastructure |
| 8 | Test pyramid — Storybook interaction → axe-core → Chromatic → Playwright E2E | `npm test` / CI | Storybook → axe → Chromatic → Playwright | §5.7 Testing |
| 9 | Silent token refresh mid-session + 401 recovery | access_token nearing expiry | AuthContext → IdP /token → silent refresh → API retry | §4 Auth Layer |
| 10 | Compliance audit trail — append-only event flow | User submits payment | AuditClient → Audit API → Kafka → Compliance MFE / SIEM | §6.1 Audit Trail |

---

## Flow 1: Shell Bootstrap · OAuth2 PKCE Auth · Module Federation Negotiation

> **§5.0 Shell — Auth-Gated Host Application**  
> **Feature:** `index.ts` async bootstrap indirection + OAuth2 PKCE auth flow + Module Federation shared singleton negotiation.  
> The browser downloads the shell bundle. The async import boundary gives the MF runtime a tick to negotiate shared module versions. In parallel, `AuthProvider` detects no access token in memory and initiates the OAuth2 PKCE redirect. On return from the IdP, the auth code is exchanged for an access token, stored in React context (memory only), and the platform shell renders with the user's role context loaded.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser)
    participant Entry as shell/src/index.ts
    participant MFRuntime as Module Federation Runtime
    participant SharedScope as Shared Scope (singleton registry)
    participant Bootstrap as shell/src/bootstrap.tsx
    participant AuthProvider as AuthProvider (PKCE client)
    participant IdP as Identity Provider (/authorize · /token)
    participant AuthCtx as AuthContext (memory token store)
    participant FlagClient as FeatureFlagProvider (LaunchDarkly)
    participant AuditClient as AuditProvider
    participant Router as BrowserRouter + AuthGuard routes
    participant Nav as GlobalNavbar

    User->>Entry: GET https://app.fintechbank.com (page load)
    Note over Entry: index.ts — async boundary via import("./bootstrap") gives MF runtime an event-loop tick

    Entry->>MFRuntime: initContainerAsync()
    activate MFRuntime
    MFRuntime->>SharedScope: register react@18.3.x {singleton:true}
    MFRuntime->>SharedScope: register react-dom@18.3.x {singleton:true}
    MFRuntime->>SharedScope: register react-router-dom@6.22.x {singleton:true}
    MFRuntime->>SharedScope: register @fintechbank/auth-context {singleton:true}
    MFRuntime->>SharedScope: register @fintechbank/feature-flags {singleton:true}
    MFRuntime->>SharedScope: register @fintechbank/audit-client {singleton:true}
    SharedScope-->>MFRuntime: shared scope sealed — versions locked
    deactivate MFRuntime

    Entry->>Bootstrap: import("./bootstrap") resolves
    activate Bootstrap

    Bootstrap->>AuthProvider: initialise PKCE client — check memory for access_token
    activate AuthProvider
    AuthProvider->>AuthProvider: access_token = null — no session in memory
    AuthProvider->>IdP: redirect to /authorize?response_type=code&code_challenge=...
    Note over AuthProvider,IdP: PKCE — code_verifier in JS, code_challenge = BASE64URL(SHA-256(code_verifier))

    User->>IdP: enters username + password + TOTP
    IdP-->>AuthProvider: redirect back to /callback?code=AUTH_CODE

    AuthProvider->>IdP: POST /token { code, code_verifier, client_id }
    IdP-->>AuthProvider: { access_token (15 min TTL), id_token, set-cookie: refresh_token (httpOnly) }
    Note over AuthProvider,IdP: access_token in memory only (never localStorage), refresh_token via httpOnly SameSite=Strict cookie

    AuthProvider->>AuthCtx: setAuthState({ user, access_token, roles: ["customer"] })
    deactivate AuthProvider

    Bootstrap->>FlagClient: initialise LaunchDarkly with user context (userId, roles)
    Bootstrap->>AuditClient: initialise audit client (userId, sessionId)

    Bootstrap->>Router: render(<BrowserRouter><AuthGuard><App/></AuthGuard></BrowserRouter>)
    activate Router
    Router->>Nav: render <GlobalNavbar user={user} />
    Nav-->>Router: navigation rendered with account switcher
    Router-->>Bootstrap: shell mounted — route "/" active
    deactivate Router
    deactivate Bootstrap

    Bootstrap-->>User: Shell visible — authenticated nav bar, route "/" active
```

### Flow 1 — Layer Call Chain

```
Browser GET https://app.fintechbank.com
    │
    ▼
shell/src/index.ts   ← async boundary import("./bootstrap")
    │
    ▼
Module Federation Runtime  ← negotiates 6 shared singletons
    │  seals shared scope — react, auth-context, feature-flags, audit-client locked
    ▼
shell/src/bootstrap.tsx
    │  AuthProvider — no token in memory → PKCE redirect to IdP
    │  User authenticates → code returned to /callback
    │  POST /token → access_token (memory) + refresh_token (httpOnly cookie)
    │  FeatureFlagProvider.init(userId, roles)
    │  AuditProvider.init(userId, sessionId)
    ▼
React tree mounted
    │  BrowserRouter → AuthGuard → GlobalNavbar
    ▼
User sees authenticated platform shell
```

### Flow 1 — Security Properties of Bootstrap

| Property | Mechanism | Attack Mitigated |
|---|---|---|
| PKCE code_challenge | SHA-256 of code_verifier in redirect | Auth code interception (MITM) |
| access_token in memory | React context only — no persistence | XSS token exfiltration |
| refresh_token httpOnly cookie | SameSite=Strict; Secure | XSS + CSRF token theft |
| Async MF bootstrap boundary | `import("./bootstrap")` | Module Federation shared scope race condition |
| CSP nonce on script tags | Server-generated per-request | Inline script injection |

---

## Flow 2: Dashboard MFE Lazy Load with Auth Guard

> **§5.1 Dashboard — Account Overview Micro-Frontend**  
> **Feature:** Route-level AuthGuard check → React.lazy() → Module Federation remoteEntry fetch → Dashboard MFE mount with inherited auth context.  
> When the user navigates to `/dashboard`, the AuthGuard validates the access token before React.lazy() fires. If the token is valid, Module Federation fetches the Dashboard remoteEntry, resolves the auth-context singleton from the shared scope, and mounts the DashboardApp — which can immediately call APIs using the inherited access token without re-authenticating.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Router as BrowserRouter (Shell)
    participant AuthGuard as AuthGuard (Shell)
    participant AuthCtx as AuthContext (singleton)
    participant Suspense as <Suspense fallback={<PageLoader/>}>
    participant LazyLoader as React.lazy(() => import("dashboard/App"))
    participant MFRuntime as Module Federation Runtime
    participant CDN as Dashboard CDN origin
    participant SharedScope as Shared Scope (singleton registry)
    participant DashboardApp as DashboardApp (dashboard/src/App.tsx)
    participant ReactQuery as React Query (dashboard cache)
    participant AccountAPI as Account API (REST)

    User->>Router: navigates to /dashboard
    Router->>AuthGuard: route matched — check auth
    activate AuthGuard
    AuthGuard->>AuthCtx: isAuthenticated? token expiry check
    AuthCtx-->>AuthGuard: ✅ valid — user has role "customer"
    deactivate AuthGuard

    Router->>Suspense: render lazy component slot
    Suspense-->>User: show <PageLoader /> (accessible — aria-busy="true")

    Suspense->>LazyLoader: trigger React.lazy() resolution
    activate LazyLoader

    LazyLoader->>MFRuntime: import("dashboard/App")
    activate MFRuntime

    MFRuntime->>CDN: GET /remoteEntry.js (Cache-Control: no-cache,no-store)
    CDN-->>MFRuntime: remoteEntry.js — module manifest (~3 KB)

    MFRuntime->>SharedScope: check @fintechbank/auth-context singleton
    SharedScope-->>MFRuntime: auth-context@3.x.x registered by Shell ✅

    MFRuntime->>SharedScope: check react singleton
    SharedScope-->>MFRuntime: react@18.3.x registered ✅

    MFRuntime->>CDN: GET /dashboard.[contenthash].js (~250 KB — no React, no auth-context)
    CDN-->>MFRuntime: dashboard domain bundle

    MFRuntime-->>LazyLoader: DashboardApp component resolved
    deactivate MFRuntime

    LazyLoader-->>Suspense: component ready
    deactivate LazyLoader

    Suspense-->>User: hide <PageLoader />
    Suspense->>DashboardApp: mount <DashboardApp />
    activate DashboardApp

    DashboardApp->>AuthCtx: useAuthContext() — reads access_token from singleton
    AuthCtx-->>DashboardApp: { user, getAccessToken() }

    DashboardApp->>ReactQuery: useQuery("account-summary", accountId)
    ReactQuery->>AccountAPI: GET /api/accounts/{id}/summary (Authorization: Bearer <token>)
    AccountAPI-->>ReactQuery: { balance: £12,450.00, accountNumber: "****1234" }
    ReactQuery-->>DashboardApp: { data: accountSummary, isLoading: false }

    DashboardApp-->>User: Dashboard page fully rendered — balance, transactions, portfolio
    deactivate DashboardApp
```

### Flow 2 — Layer Call Chain

```
User navigates to /dashboard
    │
    ▼
BrowserRouter (Shell) — route matched
    │
    ▼
AuthGuard — validates access_token in AuthContext
    │ token valid → continue
    │ token expired → trigger silent refresh (see Flow 9) → continue
    │ no token → redirect to /login
    ▼
<Suspense fallback={<PageLoader aria-busy="true" />}>
    │ React.lazy(() => import("dashboard/App"))
    │ → PageLoader renders immediately
    ▼
Module Federation Runtime
    │ GET remoteEntry.js (no-cache,no-store)
    │ Resolve auth-context singleton from Shell's shared scope
    │ Resolve react singleton from Shell's shared scope
    │ GET dashboard bundle (no React, no auth-context — both are singletons)
    ▼
DashboardApp mounts
    │ useAuthContext() — reads access_token from shared singleton
    │ React Query — GET /api/accounts with Authorization: Bearer header
    ▼
User sees account summary, portfolio, recent transactions
```

---

## Flow 3: Design System Component Resolution — Storybook → Chromatic → npm → MFE Build

> **§3 Design System Library Architecture**  
> **Feature:** End-to-end lifecycle of a new Design System component: story written in Storybook → visual regression via Chromatic → accessibility gate → npm publish → consumed in Payments MFE.  
> This flow traces a `<CurrencyInput>` component from creation through the Storybook-powered CI pipeline to production use in the Payments MFE. It illustrates how the Design System is the single source of truth — no MFE rebuilds its own input fields.

```mermaid
sequenceDiagram
    autonumber
    actor DSEngineer as Design System Engineer
    participant GitHub as GitHub (design-system repo)
    participant StorybookCI as Storybook Build CI
    participant Storybook as Storybook 8 (localhost:6006 / CI)
    participant AxeAddon as axe-core Accessibility Addon
    participant Chromatic as Chromatic (visual regression)
    participant Designer as Designer (Chromatic review)
    participant npmRegistry as Private npm Registry (GitHub Packages)
    actor PaymentsTeam as Payments Team
    participant RenovateBot as Renovate Bot
    participant PaymentsMFE as Payments MFE webpack build
    participant PaymentsApp as <PaymentForm> (runtime)

    DSEngineer->>GitHub: PR: add <CurrencyInput> atom + story
    activate GitHub
    GitHub->>StorybookCI: trigger CI

    StorybookCI->>StorybookCI: TypeScript typecheck (tsc --noEmit)
    StorybookCI->>StorybookCI: ESLint + Prettier format check

    StorybookCI->>Storybook: build Storybook (storybook build)
    activate Storybook
    Storybook->>AxeAddon: run axe-core on all stories including CurrencyInput
    AxeAddon-->>Storybook: CurrencyInput — 0 violations ✅ (label association, aria-describedby)
    Storybook-->>StorybookCI: Storybook build complete
    deactivate Storybook

    StorybookCI->>Chromatic: publish Storybook build for visual snapshot diff
    activate Chromatic
    Chromatic->>Chromatic: compare CurrencyInput snapshots vs baseline
    Chromatic-->>StorybookCI: ⚠️ visual change detected — needs review
    deactivate Chromatic

    Chromatic->>Designer: notify: CurrencyInput visual change for approval
    activate Designer
    Designer->>Chromatic: reviews diffs — currency symbol position, focus ring
    Designer->>Chromatic: ✅ approves visual change
    deactivate Designer

    Chromatic-->>GitHub: all checks passed ✅

    StorybookCI->>StorybookCI: Jest unit tests (renders, controlled input behaviour)
    StorybookCI->>StorybookCI: Storybook interaction tests (play() — user types amount)
    StorybookCI-->>GitHub: all CI gates passed

    DSEngineer->>GitHub: merge PR
    deactivate GitHub

    GitHub->>npmRegistry: publish @fintechbank/design-system@2.4.0 (minor — new component)
    npmRegistry-->>GitHub: package published ✅

    npmRegistry->>RenovateBot: Renovate detects new minor version 2.4.0
    RenovateBot->>PaymentsTeam: open automated PR: bump design-system 2.3.1 → 2.4.0

    PaymentsTeam->>PaymentsMFE: review + merge Renovate PR
    activate PaymentsMFE
    PaymentsMFE->>PaymentsMFE: import { CurrencyInput } from "@fintechbank/design-system"
    PaymentsMFE->>PaymentsMFE: webpack build — tree-shake design-system bundle
    Note over PaymentsMFE: CurrencyInput included, KYCDocumentCard tree-shaken (unused organism)
    PaymentsMFE-->>PaymentsTeam: build complete — new remoteEntry.js deployed
    deactivate PaymentsMFE

    PaymentsTeam-->>PaymentsApp: <CurrencyInput> now available in Payments MFE at runtime
```

### Flow 3 — Design System Governance Gates

```
Pull Request opened to design-system repo
    │
    ├── 1. TypeScript typecheck (0 errors)
    ├── 2. ESLint / Prettier (0 lint errors)
    ├── 3. Jest unit tests (component logic — controlled input, prop types)
    ├── 4. Storybook interaction tests (play() functions — user events)
    ├── 5. axe-core accessibility scan (0 WCAG 2.1 AA violations) ← HARD GATE
    ├── 6. Chromatic visual snapshot diff ← requires designer approval on change
    └── 7. Coverage threshold ≥ 90% statements ← HARD GATE
              │
              ▼
         All gates pass → merge → npm publish (semver)
              │
              ▼
         Renovate bot opens PRs in all downstream MFEs (patch = auto-merge, minor = review)
```

---

## Flow 4: Cross-MFE Payment Initiation · PCI-DSS Boundary · Audit Trail

> **§5.2 Payments — Regulated Payment Micro-Frontend**  
> **Feature:** User initiates a card payment → Payments MFE hands off to PCI iframe → card tokenisation → payment submission → audit events dispatched throughout.  
> This flow is the highest-stakes sequence in the platform. It crosses the PCI-DSS boundary (via sandboxed iframe), triggers PSD2 Strong Customer Authentication (SCA), and dispatches three immutable audit events. No raw card data (PAN, CVV, expiry) ever enters the React application or the Payments MFE's JS heap.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant PaymentsApp as PaymentsApp (Payments MFE)
    participant AuditClient as AuditClient (singleton)
    participant AuditAPI as Audit API (append-only)
    participant AuthCtx as AuthContext (access_token)
    participant PaymentAPI as Payment API
    participant SCAService as SCA / Step-Up Auth Service
    participant PCIFrame as <PCIFrame> (https://pci.fintechbank.com)
    participant PCIVault as PCI Vault (tokenisation service)

    User->>PaymentsApp: fills in recipient + amount → clicks "Pay with Card"

    PaymentsApp->>AuditClient: dispatch({ event: "PAYMENT_INITIATED", amount, recipientMasked })
    AuditClient->>AuditAPI: POST /audit/events (async, non-blocking)
    AuditAPI-->>AuditClient: 202 Accepted (event queued)

    PaymentsApp->>PaymentsApp: render <PCIFrame src="https://pci.fintechbank.com/capture" />
    activate PCIFrame
    Note over PaymentsApp,PCIFrame: Cross-origin iframe — JS cannot inspect DOM, card data never enters app heap

    PaymentsApp-->>User: PCI card capture form visible inside iframe

    User->>PCIFrame: enters card number, expiry, CVV
    PCIFrame->>PCIVault: POST /tokenise { pan, expiry, cvv } (direct — never via app)
    activate PCIVault
    PCIVault-->>PCIFrame: { payment_method_token: "pm_tok_8f3a1b..." }
    deactivate PCIVault

    PCIFrame->>PaymentsApp: postMessage({ type: "PAYMENT_TOKEN", token: "pm_tok_8f3a1b..." }, "https://app.fintechbank.com")
    Note over PCIFrame,PaymentsApp: target origin MUST be exact — prevents token leakage to other origins

    deactivate PCIFrame

    PaymentsApp->>PaymentsApp: window.addEventListener("message") handler
    PaymentsApp->>PaymentsApp: validate event.origin === "https://pci.fintechbank.com"

    PaymentsApp->>SCAService: POST /sca/challenge { userId, amount, payment_method_token }
    activate SCAService
    SCAService-->>PaymentsApp: { challengeId, method: "TOTP" }
    PaymentsApp-->>User: SCA prompt — enter authenticator code
    User->>PaymentsApp: enters TOTP code
    PaymentsApp->>SCAService: POST /sca/verify { challengeId, otp }
    SCAService-->>PaymentsApp: { sca_token: "sca_verified_xyz" }
    deactivate SCAService

    PaymentsApp->>AuthCtx: getAccessToken() (may trigger silent refresh)
    AuthCtx-->>PaymentsApp: Bearer <access_token>

    PaymentsApp->>PaymentAPI: POST /api/payments { amount, recipient, payment_method_token, sca_token }
    Note over PaymentsApp,PaymentAPI: Authorization: Bearer header — access_token sent in header (never URL)
    activate PaymentAPI
    PaymentAPI-->>PaymentsApp: 201 Created { transactionId: "TXN-29481", status: "PENDING" }
    deactivate PaymentAPI

    PaymentsApp->>AuditClient: dispatch({ event: "PAYMENT_SUBMITTED", transactionId: "TXN-29481" })
    AuditClient->>AuditAPI: POST /audit/events
    AuditAPI-->>AuditClient: 202 Accepted

    PaymentsApp-->>User: Payment submitted — "TXN-29481 pending confirmation"
```

### Flow 4 — PCI-DSS Boundary Rules

| Rule | Implementation | PCI Requirement |
|---|---|---|
| No raw card data in app JS heap | iframe is cross-origin — parent cannot read DOM | PCI-DSS Req 3 (protect stored data) |
| postMessage target origin is exact | `"https://pci.fintechbank.com"` only — wildcard `"*"` is forbidden | Prevents token exfiltration |
| Tokenisation before leaving PCI scope | PCIVault returns opaque token — PAN never leaves PCI iframe | PCI-DSS Req 4 (encrypt in transit) |
| SCA for payments above threshold | PSD2 RTS Article 5 — €30+ requires Strong Customer Authentication | Regulatory compliance |
| Audit event on initiate + submit | AuditClient dispatches immutable events on every payment action | PCI-DSS Req 10 (logging) |

---

## Flow 5: Feature Flag Evaluation for Trading Instrument Rollout

> **§5.3 Trading — Market Data and Order Management Micro-Frontend**  
> **Feature:** LaunchDarkly feature flag evaluation controls which order types are visible to which user segments before regulatory approval has been granted for all users.  
> The `useFeatureFlag` hook connects to the LaunchDarkly client (a shared singleton initialised in the Shell with the user's context). Flag evaluation is synchronous after initialisation — no network call per evaluation. A kill switch can disable a trading feature for all users within seconds, without a code deployment.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Beta Customer)
    participant TradingApp as TradingApp (Trading MFE)
    participant FlagClient as FeatureFlagClient (LaunchDarkly singleton)
    participant LDEdge as LaunchDarkly Edge (streaming connection)
    participant OrderPanel as <OrderPanel>
    participant EquityForm as <EquityOrderForm>
    participant OptionsChain as <OptionsChain>
    participant CryptoForm as <CryptoOrderForm>
    actor ComplianceOfficer as Compliance Officer (LaunchDarkly dashboard)

    Note over FlagClient,LDEdge: Flag client initialised in Shell bootstrap with user context (userId, roles, segment)

    FlagClient->>LDEdge: SSE streaming connection established (persistent)
    LDEdge-->>FlagClient: flags — options_chain.enabled=true (beta), crypto_pairs.enabled=false (all users)

    User->>TradingApp: navigates to /trading
    TradingApp->>OrderPanel: render <OrderPanel instrument="AAPL" />
    activate OrderPanel

    OrderPanel->>FlagClient: useFeatureFlag("trading.options_chain.enabled")
    FlagClient-->>OrderPanel: true (user is in beta segment)

    OrderPanel->>FlagClient: useFeatureFlag("trading.crypto_pairs.enabled")
    FlagClient-->>OrderPanel: false (rolled out to 0% of users)

    OrderPanel->>EquityForm: render <EquityOrderForm /> (always shown)
    OrderPanel->>OptionsChain: render <OptionsChain /> (flag = true)
    Note over OrderPanel: CryptoOrderForm NOT rendered — flag is false
    OrderPanel-->>User: order panel shows Equity + Options (Crypto hidden)

    Note over ComplianceOfficer: Approval received for crypto trading for all EU customers

    ComplianceOfficer->>LDEdge: update flag trading.crypto_pairs.enabled — rule: EU_customers segment = true

    LDEdge-->>FlagClient: streaming update received — flag changed
    FlagClient-->>OrderPanel: useFeatureFlag("trading.crypto_pairs.enabled") → re-evaluate → true

    OrderPanel->>CryptoForm: render <CryptoOrderForm /> (flag now = true)
    OrderPanel-->>User: crypto order form appears — no page reload required
```

### Flow 5 — Feature Flag Kill Switch (Emergency)

```
Scenario: A critical bug found in OptionsChain — must disable immediately

Compliance Officer / On-call Engineer:
    │
    └── LaunchDarkly dashboard: toggle trading.options_chain.enabled → false (all users)
              │
              ▼
    LaunchDarkly edge SDKs receive streaming update within ~500ms
              │
              ▼
    FlagClient (singleton in Shell) fires change event
              │
              ▼
    React re-renders OrderPanel:
      useFeatureFlag("trading.options_chain.enabled") → false
      <OptionsChain> unmounts immediately
              │
              ▼
    All users: OptionsChain hidden — zero deployment required
    Options orders in progress: gracefully interrupted — user sees "Feature temporarily unavailable"
```

---

## Flow 6: Shared Singleton Version Negotiation + auth-context Reuse

> **§5.5 Shared Module Design**  
> **Feature:** Module Federation runtime shared scope negotiation — all six singletons (react, auth-context, feature-flags, audit-client, etc.) are resolved from the shell's registered scope when Dashboard MFE loads.  
> This flow shows the exact sequence: the Dashboard MFE's `initContainerAsync()` queries the shared scope for each singleton, finds them already registered by the Shell, and receives a reference to the same instance — not a duplicate. Critically, the `access_token` stored in the auth-context singleton is **the same object in the JS heap** that the Shell manages. Dashboard's API calls use the correct, latest token without any re-authentication.

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Browser JS Runtime
    participant ShellMF as Shell MF Plugin (already initialised)
    participant SharedScope as Shared Scope (global singleton registry)
    participant DashboardMF as Dashboard MF Plugin (remoteEntry.js loading)
    participant ReactSingleton as react@18.3.x (one instance in heap)
    participant AuthSingleton as @fintechbank/auth-context (one instance in heap)
    participant FlagSingleton as @fintechbank/feature-flags (one instance in heap)
    participant AuditSingleton as @fintechbank/audit-client (one instance in heap)

    Note over Browser,AuditSingleton: Shell bootstrap complete — all six singletons registered in SharedScope

    Browser->>DashboardMF: import("dashboard/App") — React.lazy() triggered

    DashboardMF->>SharedScope: initContainerAsync() — check registered singletons

    SharedScope->>SharedScope: react — registered: 18.3.x, requiredVersion: ^18.3.0
    SharedScope-->>DashboardMF: ✅ compatible — reuse Shell's react@18.3.x (DO NOT bundle own)
    DashboardMF->>ReactSingleton: reference acquired — same heap instance

    SharedScope->>SharedScope: @fintechbank/auth-context — registered: 3.2.0, requiredVersion: ^3.0.0
    SharedScope-->>DashboardMF: ✅ compatible — reuse Shell's auth-context instance
    DashboardMF->>AuthSingleton: reference acquired — Dashboard reads SAME access_token

    SharedScope->>SharedScope: @fintechbank/feature-flags — registered: 2.1.0
    SharedScope-->>DashboardMF: ✅ compatible — reuse Shell's LaunchDarkly client instance
    DashboardMF->>FlagSingleton: reference acquired — same streaming SSE connection

    SharedScope->>SharedScope: @fintechbank/audit-client — registered: 1.3.0
    SharedScope-->>DashboardMF: ✅ reuse — Dashboard audit events go through same queue
    DashboardMF->>AuditSingleton: reference acquired — same batch queue

    DashboardMF-->>Browser: DashboardApp component resolved
    Note over Browser,AuditSingleton: Dashboard bundle excludes react + auth-context + feature-flags + audit-client (~195KB saved)
```

### Flow 6 — Singleton Resolution Failure Scenarios

```
Scenario A — Happy path (all singletons compatible):
┌──────────────────────────────────────────────────────────────────────┐
│  Shell seals: react@18.3.0, auth-context@3.2.0, feature-flags@2.1.0 │
│  Dashboard requires: ^18.3.0, ^3.0.0, ^2.0.0                        │
│  → All semver compatible → singletons REUSED                         │
│  → Dashboard bundle: no duplicate auth/react/flags code              │
└──────────────────────────────────────────────────────────────────────┘

Scenario B — auth-context minor mismatch (3.2.0 vs 3.0.0):
┌──────────────────────────────────────────────────────────────────────┐
│  Shell: auth-context@3.2.0, Dashboard requires: ^3.0.0              │
│  → ^3.0.0 SATISFIED by @3.2.0 → reuse Shell's instance              │
│  → New minor features in 3.2.0 available to Dashboard               │
└──────────────────────────────────────────────────────────────────────┘

Scenario C — auth-context major version mismatch (CRITICAL):
┌──────────────────────────────────────────────────────────────────────┐
│  Shell: auth-context@3.2.0, Dashboard requires: ^4.0.0              │
│  → Major version mismatch → Module Federation warns and loads own    │
│  → TWO auth-context instances in heap                                 │
│  → Dashboard's instance has NO access_token (Shell set it in v3 instance)│
│  → All Dashboard API calls get 401 Unauthorized                      │
│  FIX: Pin all MFEs and Shell to same auth-context major version.     │
│  Coordinate via RFC process before any major version bump.           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Flow 7: Canary Deployment · Metric Gate · Automated Rollback

> **§5.6 Infrastructure and Deployment**  
> **Feature:** Payments MFE code pushed to `main` → CI quality gates → canary deploy (5% traffic) → metric gate evaluation → full promotion or automated rollback.  
> In a regulated environment, a broken payment form is a P0 incident requiring FCA incident reporting. The canary strategy limits blast radius to 5% of traffic. Automated metric gates (error rate, p99 latency) gate promotion without manual intervention — promotion happens in minutes if metrics are nominal, rollback in seconds if they are not.

```mermaid
sequenceDiagram
    autonumber
    actor PaymentsTeam as Payments Team
    participant GitHub as GitHub (main branch)
    participant CI as GitHub Actions (payments-deploy.yml)
    participant Webpack as webpack (payments build)
    participant CDNCanary as CDN Canary Origin (/canary/remoteEntry.js)
    participant CDNProd as CDN Production Origin (/remoteEntry.js)
    participant EdgeRule as CDN Edge Rule (5% → canary)
    participant MetricGate as Metric Gate (Datadog / CloudWatch)
    actor UserA as 95% Traffic (existing users)
    actor UserB as 5% Canary Traffic

    PaymentsTeam->>GitHub: git push origin main (payments/ path change)
    GitHub->>CI: trigger: paths: ["packages/payments/**"]
    activate CI

    CI->>CI: TypeScript typecheck (tsc --noEmit)
    CI->>CI: ESLint + Prettier
    CI->>CI: Vitest unit tests (--coverage threshold 80%)
    CI->>CI: axe-core accessibility scan (payments forms)
    CI->>CI: Storybook interaction tests (PaymentForm play() functions)
    CI-->>CI: ✅ all quality gates passed

    CI->>Webpack: npm run build (webpack production, TypeScript target)
    activate Webpack
    Webpack->>Webpack: bundle payments domain code (no react, no auth-context — singletons)
    Webpack->>Webpack: emit dist/remoteEntry.js (~3 KB manifest)
    Webpack->>Webpack: emit dist/payments.[NEW_HASH].js (~300 KB content-hashed)
    Webpack-->>CI: build complete
    deactivate Webpack

    CI->>CDNCanary: upload payments.[NEW_HASH].js (Cache-Control: max-age=31536000,immutable)
    CI->>CDNCanary: upload remoteEntry.js (Cache-Control: no-cache,no-store)
    CDNCanary-->>CI: canary assets deployed ✅

    CI->>EdgeRule: activate canary: route 5% traffic to /canary/remoteEntry.js
    EdgeRule-->>CI: canary routing active

    deactivate CI

    UserB->>CDNCanary: GET /canary/remoteEntry.js → payments.[NEW_HASH].js
    UserB-->>UserB: receives new Payments MFE
    UserA->>CDNProd: GET /remoteEntry.js → payments.[OLD_HASH].js (unchanged)
    UserA-->>UserA: receives previous Payments MFE

    loop Every 60s for 30 minutes
        MetricGate->>MetricGate: evaluate canary:
        MetricGate->>MetricGate:   error_rate < 0.1% ?
        MetricGate->>MetricGate:   p99_latency < 800ms ?
        MetricGate->>MetricGate:   payment_success_rate > 99% ?
    end

    alt Metrics nominal for 30 minutes
        MetricGate->>CDNProd: promote: copy canary/remoteEntry.js → /remoteEntry.js
        MetricGate->>EdgeRule: deactivate canary routing — all traffic to production
        CDNProd-->>UserA: next page load — receives promoted new Payments MFE ✅
        Note over MetricGate: Shell not redeployed — auto-discovery via remoteEntry.js
    else Error rate spike detected
        MetricGate->>EdgeRule: rollback: disable canary routing immediately
        MetricGate->>CDNCanary: flag canary version as bad
        Note over MetricGate,UserB: 5% canary users: next page load returns to production version
        MetricGate-->>PaymentsTeam: alert: canary deployment rolled back — metrics: error_rate=1.2%
    end
```

### Flow 7 — CDN Cache Strategy (FinTech Tightening)

| Asset | Cache-Control | Reason |
|---|---|---|
| `remoteEntry.js` (production) | `no-cache, no-store` | Zero TTL — regulatory requirement: deploy must be immediately visible |
| `remoteEntry.js` (canary) | `no-cache, no-store` | Same — canary must be promotable without cache flush |
| `payments.[hash].js` | `max-age=31536000, immutable` | Content-hashed — hash guarantees immutability |
| `shell/index.html` | `no-cache` | Entry point — must always serve current shell version |
| API responses | `no-store` | Financial data must never be cached at CDN layer |

---

## Flow 8: Test Pyramid — Storybook Interaction → axe-core → Chromatic → Playwright E2E

> **§5.7 Testing Strategy**  
> **Feature:** Full FinTech-grade quality pipeline — from Storybook interaction tests through WCAG accessibility gates to Chromatic visual regression and real-browser Playwright E2E with PCI iframe verification.  
> Each layer provides a distinct quality signal. Storybook interaction tests catch component-level regressions in seconds. axe-core catches accessibility regressions before they reach users. Chromatic catches visual regressions before they reach the Design System consumer MFEs. Playwright verifies real Module Federation wiring, real auth context, and real payment form flows.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer / CI
    participant StorybookTest as Storybook Interaction Tests (play() functions)
    participant AxeCore as axe-core Accessibility Scanner
    participant Chromatic as Chromatic (visual regression)
    participant Vitest as Vitest + React Testing Library (jsdom)
    participant MockMF as moduleNameMapper (MF mock)
    participant MSW as Mock Service Worker (API mock)
    participant ContractTest as MF Contract Test (remoteEntry.js assertion)
    participant LighthouseCI as Lighthouse CI (perf + a11y budget)
    participant Playwright as Playwright (real Chromium browser)
    participant ShellDev as Shell devServer (:3000)
    participant PaymentsDev as Payments devServer (:3002)
    participant PCIStub as PCI iframe stub (test origin)

    rect rgb(220, 240, 255)
        Note over Dev,AxeCore: LAYER 1 — Storybook Interaction + Accessibility (< 2 min, every commit)

        Dev->>StorybookTest: storybook test (CI mode)
        StorybookTest->>StorybookTest: run play() on CurrencyInput, PaymentForm, KYCDocumentCard
        Note over StorybookTest: play() — real Playwright in Storybook CI, userEvent.type and userEvent.click assert DOM state
        StorybookTest-->>Dev: ✅ interaction tests passed

        StorybookTest->>AxeCore: axe-core scan all stories
        AxeCore-->>Dev: ✅ 0 WCAG 2.1 AA violations — HARD GATE
    end

    rect rgb(220, 255, 220)
        Note over Dev,Chromatic: LAYER 2 — Visual Regression (< 5 min, every PR)

        Dev->>Chromatic: chromatic --auto-accept-changes=false
        Chromatic->>Chromatic: compare all story snapshots vs approved baseline
        Note over Chromatic: Changed CurrencyInput focus ring → blocks PR until designer approves
        Chromatic-->>Dev: ✅ no unexpected visual changes (or blocked pending approval)
    end

    rect rgb(255, 250, 220)
        Note over Dev,MSW: LAYER 3 — Unit / Integration (< 30s, every commit)

        Dev->>Vitest: npm test (Payments MFE)
        Vitest->>MockMF: intercept import("payments/App") via moduleNameMapper
        MockMF-->>Vitest: resolve to __mocks__/paymentsApp.tsx

        Vitest->>MSW: register handlers: GET /api/accounts → mock balances
        Vitest->>Vitest: render(<PaymentForm />) in jsdom
        Vitest->>Vitest: assert: recipient field, amount validation, submit button disabled
        Vitest-->>Dev: ✅ 847 unit tests passed — 82% line coverage
    end

    rect rgb(240, 220, 255)
        Note over Dev,ContractTest: LAYER 4 — MF Contract + Performance Budget (< 2 min, every PR)

        Dev->>ContractTest: npm run test:contract
        ContractTest->>ContractTest: read packages/payments/dist/remoteEntry.js
        ContractTest->>ContractTest: assert exposes["./App"] exists ✅
        ContractTest->>ContractTest: assert shared[] includes react, auth-context, feature-flags ✅
        ContractTest-->>Dev: ✅ contract intact

        Dev->>LighthouseCI: lhci autorun (shell + dashboard routes)
        LighthouseCI-->>Dev: performance: 94 ✅ | accessibility: 100 ✅ | LCP: 2.1s ✅
    end

    rect rgb(255, 235, 220)
        Note over Dev,PCIStub: LAYER 5 — E2E Cross-MFE (5 min smoke, nightly full)

        Dev->>Playwright: npx playwright test e2e/payment-journey.spec.ts
        Playwright->>ShellDev: launch Chromium → GET http://localhost:3000
        ShellDev-->>Playwright: Shell loaded — auth stubbed via Playwright fixture

        Playwright->>ShellDev: navigate to /payments
        ShellDev->>PaymentsDev: fetch remoteEntry.js (real Module Federation)
        PaymentsDev-->>ShellDev: Payments MFE loaded + auth-context singleton shared ✅

        Playwright->>ShellDev: fill recipient, amount → click "Pay with Card"
        ShellDev->>PCIStub: render PCI iframe stub (test origin — returns mock token)
        PCIStub-->>ShellDev: postMessage({ type: "PAYMENT_TOKEN", token: "pm_test_..." })

        Playwright->>ShellDev: verify payment confirmation screen visible
        Playwright->>Playwright: assert audit events dispatched (mock audit API called)
        Playwright-->>Dev: ✅ full payment journey E2E passed — real MF wiring verified
    end
```

### Flow 8 — Quality Engineer's FinTech Test Matrix

| What is tested | Tool | MF wiring | Auth | Network | Speed |
|---|---|---|---|---|---|
| Component renders, variants, interactions | Storybook play() + axe | Mocked | Not needed | Mocked | < 2 min |
| WCAG 2.1 AA compliance | axe-core (jest-axe + Storybook) | Mocked | Not needed | None | < 1 min |
| Visual pixel regression vs design baseline | Chromatic | Storybook build | Not needed | None | 2–5 min |
| Payment form validation, useQuery hooks | Vitest + RTL + MSW | Mocked (moduleNameMapper) | Mocked | Mocked (MSW) | < 30s |
| remoteEntry.js exposes correct keys + shared[] | Contract test | Real built artifact | Not needed | None | < 30s |
| Core Web Vitals, performance budget | Lighthouse CI | Real webpack build | Mock user | Real localhost | 3–5 min |
| Full auth + MFE load + payment journey | Playwright | REAL devServer | Auth fixture | Real / stub API | 5–10 min |
| Cross-MFE audit trail event dispatch | Playwright + mock audit API | REAL | Auth fixture | Mock audit | ~2 min |

### Flow 8 — QE Principal Rules (FinTech Addendum)

```
                          /\
                         /  \    5% — E2E (Playwright)
                        /    \   Full auth + payment journey + audit trail
                       /  E2E \
                      /────────\
                     /          \  10% — Contract + Lighthouse CI
                    / Contract    \  remoteEntry keys + perf budget
                   / Performance   \
                  /──────────────────\
                 /                    \  25% — Chromatic + axe-core
                / Visual + A11y Tests  \  Visual regression + WCAG gates
               /──────────────────────────\
              /                            \
             /        Unit Tests            \  60% — Vitest + RTL + Storybook play()
            /  (per component + per MFE)     \  Controlled components, hooks, state machines
           /──────────────────────────────────\

FinTech additional rule: Accessibility (axe) is HORIZONTAL — it runs at EVERY layer.
  Not just unit tests. Not just E2E. Every story, every integration test, every Playwright spec
  must produce 0 axe-core WCAG 2.1 AA violations. This is non-negotiable under the EAA (June 2025).
```

---

## Flow 9: Silent Token Refresh Mid-Session + 401 Recovery

> **§4 Authentication and Authorization Layer**  
> **Feature:** Access token nears expiry while user is on the Payments MFE → `AuthContext` silently refreshes using the httpOnly refresh token cookie → API calls resume transparently.  
> This is the most common and most invisible authentication flow. The user is filling in a payment form when their 15-minute access token expires. Without silent refresh, the next API call returns 401 and the user loses their form input. With the interceptor pattern, the access_token is refreshed in the background and the failing request is retried — the user never knows it happened.

```mermaid
sequenceDiagram
    autonumber
    participant TradingApp as TradingApp (Trading MFE — mid-session)
    participant ReactQuery as React Query (useQuery hook)
    participant AuthCtx as AuthContext (access_token in memory)
    participant ApiInterceptor as API fetch interceptor (auth-context)
    participant PaymentAPI as Payment API
    participant IdP as Identity Provider (/token endpoint)

    Note over TradingApp,IdP: User on platform 14 min — access_token expires in 60s, silent refresh threshold reached

    AuthCtx->>AuthCtx: token expiry timer fires — 60s before access_token expires
    activate AuthCtx
    AuthCtx->>IdP: POST /token { grant_type: "refresh_token" } — browser auto-attaches httpOnly refresh_token cookie
    activate IdP
    IdP-->>AuthCtx: { access_token: NEW_TOKEN (15 min TTL) }
    deactivate IdP
    AuthCtx->>AuthCtx: setAuthState({ access_token: NEW_TOKEN })
    Note over AuthCtx: Silent refresh complete — user unaware
    deactivate AuthCtx

    TradingApp->>ReactQuery: useQuery("market-data", { refetchInterval: 5000 })
    ReactQuery->>ApiInterceptor: GET /api/market/AAPL/quote (5s interval)

    ApiInterceptor->>AuthCtx: getAccessToken()
    AuthCtx-->>ApiInterceptor: NEW_TOKEN (fresh — just refreshed)
    ApiInterceptor->>PaymentAPI: GET /api/market/AAPL/quote (Authorization: Bearer NEW_TOKEN)
    PaymentAPI-->>ApiInterceptor: 200 { price: £182.50, change: +1.2% }
    ApiInterceptor-->>ReactQuery: { price: £182.50 }
    ReactQuery-->>TradingApp: market data updated — chart refreshes

    Note over TradingApp,IdP: Scenario B: Silent refresh FAILED (refresh token expired / revoked)

    AuthCtx->>IdP: POST /token { grant_type: "refresh_token" }
    IdP-->>AuthCtx: 401 { error: "invalid_grant" } — refresh token expired / revoked
    activate AuthCtx
    AuthCtx->>AuthCtx: clearAuthState() — access_token = null
    AuthCtx->>AuthCtx: broadcast AUTH_EXPIRED event
    deactivate AuthCtx

    TradingApp->>ReactQuery: next refetch fires
    ReactQuery->>ApiInterceptor: GET /api/market/AAPL/quote
    ApiInterceptor->>AuthCtx: getAccessToken() → null
    ApiInterceptor-->>ReactQuery: throw AuthenticationError("session_expired")
    ReactQuery-->>TradingApp: { error: AuthenticationError }
    TradingApp->>TradingApp: render <SessionExpiredModal> — "Your session has expired"
    TradingApp-->>TradingApp: useNavigate("/login") after user acknowledges
```

---

## Flow 10: Compliance Audit Trail — Append-Only Event Pipeline

> **§6.1 Audit Trail Architecture**  
> **Feature:** From a user action in the Payments MFE through the AuditClient singleton → Audit API → append-only Kafka topic → consumed by the Compliance MFE and SIEM.  
> Every significant financial action dispatches an immutable audit event. Events are signed with HMAC, stored in an append-only event store (Apache Kafka), and consumed by the Compliance MFE for officer review and by the SIEM for real-time anomaly detection. This flow satisfies PCI-DSS Requirement 10 (audit logging) and GDPR Article 30 (records of processing activities).

```mermaid
sequenceDiagram
    autonumber
    participant PaymentsApp as PaymentsApp (Payments MFE)
    participant AuditClient as AuditClient (singleton — batched queue)
    participant AuditAPI as Audit API (write endpoint)
    participant EventSigner as Event Signer (HMAC — audit service key)
    participant KafkaTopic as Kafka Topic: audit.events (append-only, retention: 7 years)
    participant ComplianceConsumer as Compliance Consumer (Kafka consumer group)
    participant SIEMConsumer as SIEM Consumer (anomaly detection)
    participant ComplianceMFE as Compliance MFE (/compliance/audit-log)
    participant SIEM as SIEM Platform (Splunk / Elastic SIEM)
    actor ComplianceOfficer as Compliance Officer

    PaymentsApp->>AuditClient: dispatch({ event: "PAYMENT_INITIATED", amount: 5000, recipientMasked: "****1234", userId })
    Note over AuditClient: Enqueued in memory buffer (max 50 events) — non-blocking, PaymentsApp continues immediately

    PaymentsApp->>AuditClient: dispatch({ event: "PAYMENT_SUBMITTED", transactionId: "TXN-29481" })

    Note over AuditClient: Buffer flush timer fires (every 5s) OR buffer reaches 50 events

    AuditClient->>AuditAPI: POST /audit/events — batch { events: [...], sessionId, clientTimestamp }
    activate AuditAPI

    AuditAPI->>AuditAPI: enrich events: { serverTimestamp, ipAddress, userAgent }
    AuditAPI->>EventSigner: sign(event, auditServiceHmacKey)
    EventSigner-->>AuditAPI: signature appended to each event

    AuditAPI->>KafkaTopic: produce events to audit.events topic (partition by userId)
    Note over KafkaTopic: Append-only — no UPDATE or DELETE permitted. Retention: 7 years (FCA requirement)
    KafkaTopic-->>AuditAPI: offset committed ✅
    AuditAPI-->>AuditClient: 202 Accepted

    deactivate AuditAPI

    KafkaTopic->>ComplianceConsumer: consume offset (compliance consumer group)
    activate ComplianceConsumer
    ComplianceConsumer->>ComplianceConsumer: index events in audit read store (PostgreSQL / OpenSearch)
    deactivate ComplianceConsumer

    KafkaTopic->>SIEMConsumer: consume offset (siem consumer group — real-time)
    activate SIEMConsumer
    SIEMConsumer->>SIEM: stream events to SIEM platform
    SIEM->>SIEM: anomaly detection: £5000 payment at 02:47 from new device
    SIEM-->>SIEMConsumer: alert: HIGH_RISK_PAYMENT_PATTERN
    deactivate SIEMConsumer

    ComplianceOfficer->>ComplianceMFE: navigates to /compliance/audit-log
    ComplianceMFE->>ComplianceConsumer: GET /api/audit-log?userId=X&from=today
    ComplianceConsumer-->>ComplianceMFE: { events: [PAYMENT_INITIATED, PAYMENT_SUBMITTED] }
    ComplianceMFE-->>ComplianceOfficer: audit log rendered — events, timestamps, HMAC signatures
```

### Flow 10 — Audit Event Structure

```ts
// @fintechbank/audit-client/src/types.ts
interface AuditEvent {
  eventId:         string;           // UUID v4 — immutable identifier
  event:           AuditEventType;   // "PAYMENT_INITIATED" | "PAYMENT_SUBMITTED" | ...
  domain:          string;           // "payments" | "trading" | "compliance"
  userId:          string;           // from auth context
  sessionId:       string;           // browser session identifier
  clientTimestamp: string;           // ISO-8601 — when JS dispatched the event
  serverTimestamp: string;           // ISO-8601 — when Audit API received it (authoritative)
  ipAddress:       string;           // server-enriched (never from client JS)
  payload:         Record<string, unknown>; // domain-specific — PAN never included
  signature:       string;           // HMAC-SHA256(eventId + serverTimestamp + payload, auditKey)
}
```

### Flow 10 — Regulatory Compliance Property Map

| Requirement | How audit trail satisfies it |
|---|---|
| PCI-DSS Req 10 (track access to cardholder data) | Every PAYMENT_INITIATED event logged with userId, timestamp, masked recipient |
| GDPR Art 30 (records of processing activities) | Append-only events serve as processing records; 7-year retention |
| FCA SYSC 10A (trade surveillance) | SIEM consumer triggers alerts on anomalous trading patterns |
| SOC 2 CC7 (monitoring of system anomalies) | SIEM platform consumes audit topic in real time |
| GDPR right to erasure | Audit logs are NOT subject to erasure (regulatory retention obligation supersedes) |

---

*Generated 2026-03-06 · Principal Front-End Solution Architect · Principal Front-End Quality Engineer · Digital Banking & Wealth Platform (React 18 · Webpack Module Federation · Storybook 8 · TypeScript · OAuth2 PKCE · PCI-DSS · WCAG 2.1 AA · LaunchDarkly · Playwright · Chromatic)*

---

## Flow 1: Container Bootstrap + Module Federation Negotiation

> **§4.0 Container — Shell Host Application**  
> **Feature:** `index.js` bootstrap indirection + Module Federation shared singleton negotiation.  
> The browser downloads the container bundle. Before any React code runs, the Module Federation runtime intercepts the async import boundary (`import("./bootstrap")`) and negotiates shared module versions with all registered remotes. Only after this negotiation resolves does `ReactDOM.createRoot()` execute.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser)
    participant Entry as container/src/index.js
    participant MFRuntime as Module Federation Runtime
    participant SharedScope as Shared Scope (singleton registry)
    participant Bootstrap as container/src/bootstrap.js
    participant ReactDOM as ReactDOM.createRoot()
    participant Router as BrowserRouter + Routes
    participant Nav as GlobalNavbar

    User->>Entry: GET http://localhost:3000 (page load)
    Note over Entry: index.js — async boundary via import("./bootstrap") gives MF runtime an event-loop tick

    Entry->>MFRuntime: initContainerAsync()
    activate MFRuntime

    MFRuntime->>SharedScope: register react@18.x.x {singleton:true}
    MFRuntime->>SharedScope: register react-dom@18.x.x {singleton:true}
    MFRuntime->>SharedScope: register react-router-dom@6.x.x {singleton:true}
    SharedScope-->>MFRuntime: shared scope sealed — versions locked

    MFRuntime-->>Entry: negotiation complete
    deactivate MFRuntime

    Entry->>Bootstrap: import("./bootstrap") resolves
    activate Bootstrap

    Bootstrap->>ReactDOM: ReactDOM.createRoot(document.getElementById("root"))
    ReactDOM-->>Bootstrap: root created

    Bootstrap->>Router: render(<BrowserRouter><App /></BrowserRouter>)
    activate Router
    Router->>Nav: render <GlobalNavbar />
    Nav-->>Router: navbar rendered
    Router-->>Bootstrap: container shell mounted
    deactivate Router
    deactivate Bootstrap

    Bootstrap-->>User: Container shell visible — nav bar rendered, route "/" active
```

### Flow 1 — Layer Call Chain

```
Browser (HTTP GET /)
    │
    ▼
container/src/index.js   ← synchronous webpack entry
    │ import("./bootstrap")   ← async boundary — gives MF runtime a tick
    ▼
Module Federation Runtime  ← intercepts dynamic import
    │ initContainerAsync()
    │ registers react, react-dom, react-router-dom as singletons
    ▼
Shared Scope  ← version negotiation registry
    │ seals shared scope — versions cannot change after this point
    ▼
container/src/bootstrap.js  ← actual React initialisation
    │ ReactDOM.createRoot()
    │ render(<BrowserRouter><App /></BrowserRouter>)
    ▼
React tree mounted — BrowserRouter owns window.history
    │
    ▼
User sees container shell (nav + empty route slot)
```

### Flow 1 — Bootstrap Anti-Pattern

```js
// ❌ WRONG — direct import in index.js (no async boundary)
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
// ^^^ MF runtime has NO event-loop tick to negotiate shared modules
// → remotes may load their own bundled React instead of the singleton
ReactDOM.createRoot(document.getElementById("root")).render(<App />);

// ✅ CORRECT — async bootstrap indirection
// index.js:
import("./bootstrap");   // ← MF runtime negotiates BEFORE React initialises

// bootstrap.js:
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

---

## Flow 2: Listing MFE Lazy Load — React.lazy + Suspense + remoteEntry.js

> **§4.1 Listing — Product Listing Micro-Frontend**  
> **Feature:** `React.lazy()` dynamic import + Module Federation remote fetch + Suspense fallback lifecycle.  
> When the user navigates to `/listing`, React Router matches the route. The `React.lazy()` wrapper triggers a dynamic `import("listing/App")`. The browser has not yet downloaded the Listing bundle — Module Federation intercepts this import, fetches `remoteEntry.js` from the Listing server (port 3001), resolves the shared React singleton, and returns the `ListingApp` component. The Suspense boundary displays a loading fallback until the component is ready to render.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser)
    participant Router as BrowserRouter (Container)
    participant Suspense as <Suspense fallback={<Spinner/>}>
    participant LazyLoader as React.lazy(() => import("listing/App"))
    participant MFRuntime as Module Federation Runtime
    participant CDN as Listing Server / CDN (:3001)
    participant SharedScope as Shared Scope (singleton registry)
    participant ListingApp as ListingApp (listing/src/App.jsx)
    participant ProductGrid as <ProductGrid>

    User->>Router: clicks nav link → navigate("/listing")
    Router->>Suspense: route matched — render lazy component slot
    Suspense-->>User: render fallback <Spinner /> immediately

    Suspense->>LazyLoader: trigger React.lazy() resolution
    activate LazyLoader

    LazyLoader->>MFRuntime: import("listing/App")
    activate MFRuntime

    MFRuntime->>CDN: GET /remoteEntry.js
    CDN-->>MFRuntime: remoteEntry.js (module manifest — ~2 KB)

    MFRuntime->>SharedScope: check shared scope for react singleton
    SharedScope-->>MFRuntime: react@18.x.x already registered by Container ✅

    MFRuntime->>CDN: GET /listing.[contenthash].js (domain bundle)
    CDN-->>MFRuntime: listing bundle (~200 KB, no React/ReactDOM — singleton used)

    MFRuntime-->>LazyLoader: ListingApp component resolved
    deactivate MFRuntime

    LazyLoader-->>Suspense: component ready
    deactivate LazyLoader

    Suspense-->>User: hide <Spinner />
    Suspense->>ListingApp: mount <ListingApp />
    activate ListingApp

    ListingApp->>ProductGrid: render <ProductGrid />
    ProductGrid-->>ListingApp: product grid rendered

    ListingApp-->>User: /listing page fully visible
    deactivate ListingApp
```

### Flow 2 — Layer Call Chain

```
User clicks /listing nav link
    │
    ▼
BrowserRouter (Container)
    │ route matched: /listing → <ListingPage>
    ▼
<Suspense fallback={<LoadingSpinner />}>
    │ React.lazy(() => import("listing/App"))
    │ → Suspense immediately renders fallback (spinner)
    ▼
Module Federation Runtime
    │ GET remoteEntry.js from Listing server / CDN
    │ Resolve react singleton from Container's shared scope
    │ GET listing bundle (no React included — singleton reused)
    ▼
ListingApp component available
    │ Suspense hides spinner — mounts <ListingApp>
    ▼
ProductGrid → ProductCard[] renders
    │
    ▼
User sees product catalogue
```

### Flow 2 — Standalone vs Container Execution Paths

| Trigger | Path Taken | Bootstrap File Used | Router |
|---|---|---|---|
| `npm start` inside `/listing` | `listing/src/index.js` → `bootstrap.js` | `listing/src/bootstrap.js` | Own `<BrowserRouter>` (standalone) |
| Container navigates to `/listing` | `import("listing/App")` via MF | `container/src/bootstrap.js` | Container's `<BrowserRouter>` |
| Jest unit test | Direct `import App from "./App"` | Not used — import is synchronous in JSDOM | `<MemoryRouter>` in test |

---

## Flow 3: Product Search and Filter

> **§4.1 Listing — Product Listing Micro-Frontend (internal flow)**  
> **Feature:** Controlled input → React state → filtered render cycle — entirely within the Listing MFE boundary.  
> The user types in the `<SearchBar>`. This triggers a React state update in the `<App>` or a context/hook that holds the filter state. `<ProductGrid>` is a pure component — it re-renders with the filtered array. No cross-MFE communication occurs; this is a purely internal Listing MFE flow.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant SearchBar as <SearchBar> (controlled input)
    participant FilterState as filterState (useState / useReducer)
    participant CategoryFilter as <CategoryFilter>
    participant ProductGrid as <ProductGrid>
    participant ProductCard as <ProductCard> ×N
    participant ProductAPI as Products API (internal fetch)

    User->>SearchBar: types "headphones"
    SearchBar->>FilterState: setSearchQuery("headphones")
    activate FilterState
    FilterState-->>SearchBar: re-render with value="headphones"
    FilterState-->>ProductGrid: re-render with filtered={products.filter(query)}
    deactivate FilterState

    ProductGrid->>ProductCard: map filtered products → render <ProductCard>
    ProductCard-->>ProductGrid: each card rendered
    ProductGrid-->>User: grid shows only matching products

    User->>CategoryFilter: selects "Electronics"
    CategoryFilter->>FilterState: setCategoryFilter("Electronics")
    activate FilterState
    FilterState-->>CategoryFilter: re-render with active category
    FilterState-->>ProductGrid: re-render with category + search combined filter
    deactivate FilterState

    ProductGrid->>ProductCard: map doubly-filtered products → render
    ProductCard-->>ProductGrid: filtered cards rendered
    ProductGrid-->>User: grid shows "headphones" in "Electronics" only

    Note over SearchBar, ProductCard: All state is local to ListingApp — Container and other remotes are unaware

    User->>ProductCard: clicks product card
    ProductCard->>FilterState: navigate to product detail (internal sub-route)
    ProductCard-->>User: product detail view
```

### Flow 3 — Layer Call Chain

```
User types in <SearchBar>
    │ onChange → setSearchQuery(value)
    ▼
filterState (useState in ListingApp or dedicated hook)
    │ derivedProducts = useMemo(() => products.filter(q, cat), [query, category])
    ▼
<ProductGrid products={filteredProducts} />
    │ map() → <ProductCard key={id} product={p} onAddToCart={handler} />
    ▼
<ProductCard> renders: image | name | price | "Add to Cart" button
    │
    ▼
User sees filtered product grid
```

### Flow 3 — State Colocation Rule

```
Listing MFE state boundary:
┌─────────────────────────────────────────────────────┐
│  <ListingApp>                                       │
│  ├── filterState: { query: "", category: "" }       │  
│  ├── <SearchBar>  ← reads + writes filterState      │
│  ├── <CategoryFilter> ← reads + writes filterState  │
│  └── <ProductGrid products={filtered}> ← reads only │
│       └── <ProductCard> ×N  ← props only, no state  │
└─────────────────────────────────────────────────────┘
         │
         │ ← NO state leaves this boundary
         │    (only cart events cross to Cart MFE — see Flow 4)
```

---

## Flow 4: Add to Cart — Cross-MFE Custom Event Bus

> **§4.2 Cart — Shopping Cart Micro-Frontend**  
> **Feature:** Decoupled cross-MFE communication via `window` Custom Events.  
> The user clicks "Add to Cart" inside the Listing MFE. Listing knows nothing about the Cart MFE's internal state or API. It fires a browser `CustomEvent` on `window`. The Cart MFE has registered a `window.addEventListener("cart:add", handler)` at mount. This event-driven model maintains strict MFE isolation — neither remote imports the other directly.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant ProductCard as <ProductCard> (Listing MFE)
    participant ListingApp as ListingApp (Listing MFE)
    participant Window as window (Browser Event Bus)
    participant CartApp as CartApp (Cart MFE)
    participant CartState as cartState (useState in CartApp)
    participant CartIcon as <CartIcon> (Container Nav)
    participant LocalStorage as localStorage

    User->>ProductCard: clicks "Add to Cart"
    ProductCard->>ListingApp: onAddToCart(product) callback fires
    activate ListingApp

    ListingApp->>Window: window.dispatchEvent(new CustomEvent("cart:add", { detail: { product } }))
    Note over ListingApp,Window: ListingApp does NOT import CartApp — zero coupling, event is the contract
    deactivate ListingApp

    Window-->>CartApp: addEventListener("cart:add") handler fires
    activate CartApp

    CartApp->>CartState: setCartItems(prev => [...prev, product])
    activate CartState
    CartState-->>CartApp: new cart items array
    CartState-->>CartIcon: cart count badge re-renders (via shared context or event)
    deactivate CartState

    CartApp->>LocalStorage: localStorage.setItem("cart", JSON.stringify(items))
    LocalStorage-->>CartApp: persisted ✅

    CartApp-->>User: cart count badge increments in nav
    deactivate CartApp

    Note over ProductCard, LocalStorage: Cart MFE does NOT import Listing MFE — unidirectional via window events
```

### Flow 4 — Layer Call Chain

```
User clicks "Add to Cart" in ProductCard (Listing MFE)
    │
    ▼
onAddToCart(product) — prop callback fires
    │
    ▼
window.dispatchEvent(new CustomEvent("cart:add", { detail: { product } }))
    │ ListingApp does NOT reference CartApp — zero import
    ▼
window event bubbles — CartApp's listener catches it
    │ registered at CartApp mount: window.addEventListener("cart:add", handler)
    ▼
CartApp: setCartItems([...prev, product])
    │ localStorage.setItem("cart", serialized items)
    ▼
CartIcon in Container nav re-renders with updated count
    │
    ▼
User sees cart badge increment immediately
```

### Flow 4 — Cross-MFE Communication Pattern Comparison

| Pattern | Coupling | Example in Flow 4 | Breaks If |
|---|---|---|---|
| Custom Event (used here) | Zero | `window.dispatchEvent / addEventListener` | Event name typo — fails silently at runtime |
| Shared Zustand store | Low (shared module) | `useCartStore()` in both MFEs | Shared module version mismatch |
| Props drilling via Container | Medium | Container passes `onAddToCart` as prop to both remotes | Container must know about both remotes |
| Direct MFE import | High | `import CartStore from "cart/store"` | Cart MFE deploy breaks Listing at runtime |
| Backend API as source of truth | None (async) | POST /api/cart/:userId | Network latency on every add-to-cart |

---

## Flow 5: Checkout Multi-Step Form Submission

> **§4.3 Checkout — Order Checkout Micro-Frontend**  
> **Feature:** Multi-step form progression with per-step validation, cart data handoff from Cart MFE via `localStorage`, and order submission.  
> The user navigates to `/checkout`. `CheckoutApp` reads the cart from `localStorage` (the Cart MFE's persistence layer). The form progresses through four sequential steps, each validated before advancing. On final submission, the order is posted to the Order API and the user is shown a confirmation screen. Navigation back to `/listing` is handled by `useNavigate()` — which resolves against the Container's router.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Router as BrowserRouter (Container)
    participant CheckoutApp as CheckoutApp (Checkout MFE)
    participant LocalStorage as localStorage
    participant OrderReview as Step 1: <OrderReview>
    participant ShippingForm as Step 2: <ShippingForm>
    participant PaymentForm as Step 3: <PaymentForm>
    participant Confirmation as Step 4: <OrderConfirmation>
    participant OrderAPI as Order API (Backend)
    participant Navigate as useNavigate() → Container Router

    User->>Router: navigate("/checkout")
    Router->>CheckoutApp: mount <CheckoutApp>
    activate CheckoutApp

    CheckoutApp->>LocalStorage: localStorage.getItem("cart")
    LocalStorage-->>CheckoutApp: cart JSON { items: [...], total: 159.98 }

    CheckoutApp->>OrderReview: render Step 1 — cart items + totals
    OrderReview-->>User: order summary visible

    User->>OrderReview: clicks "Continue to Shipping"
    OrderReview->>CheckoutApp: stepState → step 2

    CheckoutApp->>ShippingForm: render Step 2 — address fields
    ShippingForm-->>User: shipping form visible

    User->>ShippingForm: fills in address fields
    User->>ShippingForm: clicks "Continue to Payment"
    activate ShippingForm
    ShippingForm->>ShippingForm: validate(fields) — email, address, postal regex
    alt Validation fails
        ShippingForm-->>User: inline error messages on invalid fields
        Note over ShippingForm,User: Submit stays disabled until all fields pass
    else Validation passes
        ShippingForm->>CheckoutApp: stepState → step 3 + save shippingData
        deactivate ShippingForm
    end

    CheckoutApp->>PaymentForm: render Step 3 — payment fields
    PaymentForm-->>User: payment form visible

    User->>PaymentForm: fills card details
    User->>PaymentForm: clicks "Place Order"
    activate PaymentForm
    PaymentForm->>PaymentForm: validate(cardFields) — Luhn check, expiry
    alt Payment validation fails
        PaymentForm-->>User: card error message
    else Payment validation passes
        PaymentForm->>OrderAPI: POST /api/orders { cart, shipping, payment }
        activate OrderAPI
        OrderAPI-->>PaymentForm: 201 Created { orderId: "ORD-4821" }
        deactivate OrderAPI
        deactivate PaymentForm

        PaymentForm->>CheckoutApp: stepState → step 4 + save orderId

        CheckoutApp->>LocalStorage: localStorage.removeItem("cart")
        Note over CheckoutApp,LocalStorage: Cart cleared on successful order

        CheckoutApp->>Confirmation: render Step 4 — confirmation + orderId
        Confirmation-->>User: "Order ORD-4821 confirmed! ✅"

        User->>Confirmation: clicks "Continue Shopping"
        Confirmation->>Navigate: useNavigate()("/listing")
        Navigate->>Router: push "/listing" → Container re-routes
        Router-->>User: back on /listing (Listing MFE mounts again)
    end

    deactivate CheckoutApp
```

### Flow 5 — Layer Call Chain

```
User navigates to /checkout
    │
    ▼
BrowserRouter (Container) matches route
    │ React.lazy() fetches checkout remoteEntry.js if not yet loaded
    ▼
<CheckoutApp> mounts
    │ reads localStorage.getItem("cart") — Cart MFE's persisted state
    ▼
Step 1 — <OrderReview>:  display items + total  →  user confirms
    │
    ▼
Step 2 — <ShippingForm>: address fields + client-side validation
    │ validate(): regex per field → inline errors on blur
    │ all fields valid → advance
    ▼
Step 3 — <PaymentForm>:  card fields + Luhn check
    │ POST /api/orders → 201 Created { orderId }
    ▼
Step 4 — <OrderConfirmation>:  orderId displayed
    │ localStorage.removeItem("cart")
    │ useNavigate()("/listing")  → Container BrowserRouter routes back
    ▼
User returns to Listing MFE
```

### Flow 5 — Form Validation Sequence (Step 2 Detail)

```
User submits ShippingForm
    │
    ▼
validate(fields):
    ├── email: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)   → ✅ or inline error
    ├── address: length >= 5                               → ✅ or inline error
    ├── city: non-empty                                    → ✅ or inline error
    └── postal: /^\d{5}(-\d{4})?$/.test(postal)           → ✅ or inline error
         │
         │ ALL pass?
         ▼
    setCurrentStep(3)
    setFormData(prev => ({ ...prev, shipping: fields }))
         │
         │ ANY fail?
         ▼
    setErrors({ email: "Invalid email format", ... })
    → inline messages rendered below each field
    → submit button stays disabled
```

---

## Flow 6: Shared Singleton Version Negotiation

> **§4.4 Shared Module Design**  
> **Feature:** Module Federation runtime shared scope negotiation — `singleton: true` enforcement across Container + all three remotes.  
> This flow shows the sequence the MF runtime executes when a second remote (e.g., `cart`) loads AFTER the Container has already sealed the shared scope with `react@18.2.0`. The runtime detects the already-registered singleton and reuses it — the Cart bundle ships without React/ReactDOM, saving ~1.1 MB of duplicate code.

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Browser JS Runtime
    participant ContainerMF as Container MF Plugin
    participant SharedScope as Shared Scope (global singleton registry)
    participant CartMF as Cart MF Plugin (remoteEntry.js)
    participant ReactSingleton as react@18.2.0 (one instance in heap)

    Note over Browser,ReactSingleton: Container bootstrap already executed (Flow 1)

    Browser->>CartMF: import("cart/App") triggered by React.lazy()
    activate CartMF

    CartMF->>SharedScope: initContainerAsync() — check shared scope
    activate SharedScope

    SharedScope->>SharedScope: is "react" already in scope?
    SharedScope-->>CartMF: YES — react@18.2.0 registered by Container ✅

    CartMF->>SharedScope: requested version ^18.0.0 — is 18.2.0 compatible?
    SharedScope-->>CartMF: YES — semver compatible ✅

    CartMF->>ReactSingleton: reuse Container's react@18.2.0 instance (DO NOT bundle own copy)
    ReactSingleton-->>CartMF: reference to singleton handed over

    deactivate SharedScope
    deactivate CartMF

    CartMF-->>Browser: CartApp component available
    Note over Browser,ReactSingleton: Cart MFE React hooks use the SAME instance — no "Invalid hook call" error

    Note over Browser, ReactSingleton: Two React instances = hooks throw "Invalid hook call. Hooks can only be called inside..."
```

### Flow 6 — Singleton Resolution Outcomes

```
Scenario A — Happy path (both at 18.2.0, singleton: true):
┌─────────────────────────────────────────────────────────────────┐
│  Container seals scope: react@18.2.0                           │
│  Cart loads: requires react ^18.0.0                            │
│  → 18.2.0 satisfies ^18.0.0 → REUSE Container's instance      │
│  → Cart bundle: react NOT included → -130KB                    │
└─────────────────────────────────────────────────────────────────┘

Scenario B — Version mismatch (18.2.0 vs 18.3.0, singleton: true):
┌─────────────────────────────────────────────────────────────────┐
│  Container seals scope: react@18.2.0                           │
│  Cart loads: requires react@18.3.0 exactly                     │
│  → ^18.3.0 does NOT satisfy @18.2.0 (already locked)          │
│  → Warning logged: "Shared module is not available"            │
│  → MF falls back to Cart's own bundled react@18.3.0            │
│  → TWO React instances → hooks CRASH at runtime               │
└─────────────────────────────────────────────────────────────────┘

FIX: Use ^18.0.0 requiredVersion in all four MFEs' shared{} configs
     and align all package.json to the same minor version.
```

### Flow 6 — Shared Scope State Transitions

| Phase | Owner | Shared Scope State | react Instances in Heap |
|---|---|---|---|
| Before Container loads | — | Empty | 0 |
| Container `initContainerAsync()` | Container | `react@18.2.0` sealed | 1 |
| Listing loads (first remote) | Listing MF | Queries scope → reuses | 1 |
| Cart loads (second remote) | Cart MF | Queries scope → reuses | 1 |
| Checkout loads (third remote) | Checkout MF | Queries scope → reuses | 1 |
| All four MFEs active | All | Sealed at 18.2.0 | **1 (correct)** |
| Cart has `singleton: false` (broken) | Cart MF | Ignores scope — loads own | **2 (broken)** |

---

## Flow 7: Independent MFE Deploy + Container Auto-Discovery

> **§4.5 Infrastructure and Deployment**  
> **Feature:** Path-filtered CI/CD pipeline — a change to `listing/` triggers only the Listing pipeline. The Container never redeploys; it auto-discovers the new `remoteEntry.js` on the next user page load.  
> This is the core operational value of Micro-Frontend Architecture: **Teams ship independently.**

```mermaid
sequenceDiagram
    autonumber
    actor ListingTeam as Listing Team
    participant GitHub as GitHub (main branch)
    participant CI as GitHub Actions (listing-deploy.yml)
    participant Webpack as webpack (listing build)
    participant CDN as CDN / S3 Bucket (listing assets)
    actor ContainerTeam as Container Team (no action needed)
    actor UserA as Existing User (tab open)
    actor UserB as New User (fresh page load)

    ListingTeam->>GitHub: git push origin main (changes in listing/)
    GitHub->>CI: trigger: paths: ["listing/**"] matched
    activate CI

    CI->>CI: npm ci (install deps in /listing)
    CI->>CI: npm test --watchAll=false (unit + integration)
    CI-->>CI: ✅ all tests pass

    CI->>Webpack: npm run build (webpack production)
    activate Webpack
    Webpack->>Webpack: bundle listing domain code (NO React — singleton excluded)
    Webpack->>Webpack: emit dist/remoteEntry.js (manifest, ~2KB)
    Webpack->>Webpack: emit dist/listing.[NEW_HASH].js (~200KB, content-hashed)
    Webpack-->>CI: build complete
    deactivate Webpack

    CI->>CDN: upload dist/listing.[NEW_HASH].js (Cache-Control: max-age=31536000,immutable)
    CI->>CDN: upload dist/remoteEntry.js (Cache-Control: no-cache)
    Note over CI,CDN: remoteEntry.js must be no-cache (version manifest), listing.[HASH].js is immutable (hash changes on rebuild)

    CDN-->>CI: assets deployed ✅
    deactivate CI

    Note over ContainerTeam: Container was NOT redeployed — no release coordination needed

    UserA->>CDN: current tab still has OLD remoteEntry.js cached
    Note over UserA,CDN: Old user sees old Listing until they hard-refresh or reopen tab

    UserB->>CDN: GET /remoteEntry.js (no-cache → always fresh)
    CDN-->>UserB: NEW remoteEntry.js → points to listing.[NEW_HASH].js
    UserB->>CDN: GET listing.[NEW_HASH].js
    CDN-->>UserB: new Listing bundle served (immutably cached for future)
    UserB-->>UserB: sees updated Listing MFE immediately ✅
```

### Flow 7 — Layer Call Chain

```
Listing team git push → listing/ path filter matches CI
    │
    ▼
GitHub Actions: npm ci → npm test → npm run build
    │ Webpack output:
    │   dist/remoteEntry.js           (~2 KB manifest — NO-CACHE)
    │   dist/listing.[h4sh].js        (~200 KB domain bundle — IMMUTABLE)
    │   dist/vendors.[h4sh].js        (any non-shared 3rd party — IMMUTABLE)
    ▼
S3/CDN upload:
    │ remoteEntry.js      → Cache-Control: no-cache
    │ *.js (hashed)       → Cache-Control: max-age=31536000, immutable
    ▼
Container DOES NOT redeploy — no action
    ▼
Next user page load:
    │ Browser fetches remoteEntry.js (always fresh)
    │ remoteEntry.js points to new listing.[h4sh].js
    │ Browser fetches new bundle (or hits cache if hash unchanged)
    ▼
User sees new Listing features ✅
```

### Flow 7 — CDN Cache Strategy Reference

| Asset | Cache-Control | Reason |
|---|---|---|
| `remoteEntry.js` | `no-cache` | Version manifest — must always be current |
| `listing.[hash].js` | `max-age=31536000, immutable` | Content-hashed — hash changes on rebuild; safe to cache forever |
| `vendors.[hash].js` | `max-age=31536000, immutable` | Same — hash is deterministic from content |
| `index.html` (container) | `no-cache` | Entry point — must always serve latest container shell |

---

## Flow 8: Test Execution — Unit → Contract → E2E

> **§4.6 Testing Strategy**  
> **Feature:** Full test pyramid execution — from sub-second unit tests through Module Federation contract validation to full cross-MFE Playwright E2E.  
> Each layer of the test pyramid provides a distinct quality signal. The layers run in increasing cost and decreasing frequency: unit on every file save, contract on every PR, full E2E nightly.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer / CI
    participant Jest as Jest + React Testing Library (jsdom)
    participant MockMF as moduleNameMapper (MF mock)
    participant RTL as RTL render()
    participant Component as React Component (unit under test)
    participant ContractTest as Module Contract Test
    participant RemoteEntry as remoteEntry.js (built artifact)
    participant Playwright as Playwright (real browser)
    participant Container as Container devServer (:3000)
    participant ListingDev as Listing devServer (:3001)
    participant CartDev as Cart devServer (:3002)
    participant CheckoutDev as Checkout devServer (:3003)

    rect rgb(230, 245, 255)
        Note over Dev,Component: LAYER 1 — Unit Tests (< 30s, every commit)

        Dev->>Jest: npm test (in /listing, /cart, or /checkout)
        Jest->>MockMF: intercept import("listing/App") via moduleNameMapper
        MockMF-->>Jest: resolve to __mocks__/listingApp.jsx stub

        Jest->>RTL: render(<ProductCard product={mockProduct} onAddToCart={spy} />)
        RTL->>Component: mount in jsdom (no real browser, no network)
        Component-->>RTL: DOM tree

        RTL-->>Jest: assertions: getByRole("button", { name: /add to cart/i })
        Jest-->>Dev: ✅ unit tests pass (~500ms)
    end

    rect rgb(230, 255, 230)
        Note over Dev,RemoteEntry: LAYER 2 — Module Federation Contract Test (< 1 min, every PR)

        Dev->>ContractTest: npm run test:contract (verifies MF exposes contract)

        ContractTest->>RemoteEntry: inspect built dist/remoteEntry.js
        RemoteEntry-->>ContractTest: exposes map: { "./App": "./src/App" }

        ContractTest->>ContractTest: assert "listing/App" entry exists in manifest
        ContractTest->>ContractTest: assert shared[] includes react, react-dom, react-router-dom

        ContractTest-->>Dev: ✅ contract intact — rename of exposes key would fail here
    end

    rect rgb(255, 245, 230)
        Note over Dev,CheckoutDev: LAYER 3 — E2E Cross-MFE Tests (2-5 min smoke, nightly full)

        Dev->>Playwright: npx playwright test e2e/shopping-journey.spec.ts
        Playwright->>Container: launch real Chromium → GET http://localhost:3000
        Container-->>Playwright: container shell loaded

        Playwright->>Container: navigate to /listing
        Container->>ListingDev: fetch remoteEntry.js (real Module Federation)
        ListingDev-->>Container: remoteEntry.js
        Container->>ListingDev: fetch listing bundle
        ListingDev-->>Container: listing JS bundle
        Container-->>Playwright: /listing rendered (real MFE, no mocks)

        Playwright->>Container: click "Add to Cart" on first product
        Container->>CartDev: cart:add event triggers Cart MFE update
        CartDev-->>Container: cart count badge updated
        Container-->>Playwright: badge shows "1"

        Playwright->>Container: navigate to /checkout
        Container->>CheckoutDev: fetch checkout remoteEntry + bundle
        CheckoutDev-->>Container: checkout MFE loaded
        Container-->>Playwright: /checkout rendered with cart items

        Playwright->>Container: fill shipping form + submit
        Container-->>Playwright: order confirmation visible

        Playwright-->>Dev: ✅ full journey passed — real MF wiring verified
    end
```

### Flow 8 — Layer Call Chain

```
UNIT LAYER (every commit):
Jest + RTL in jsdom
    │ moduleNameMapper mocks all Module Federation imports
    │ render(<ProductCard>) → assert DOM output, event handlers
    │ render(<CartApp>) with mock items → assert totals
    │ render(<CheckoutApp>) with mock cart → assert step transitions
    ▼ Pass: < 30 seconds

CONTRACT LAYER (every PR):
Contract test reads built remoteEntry.js
    │ assert exposes["./App"] exists
    │ assert shared[] singletons declared
    ▼ Pass: < 60 seconds — catches exposes key renames before they reach production

E2E LAYER (smoke every PR, full nightly):
Playwright with real devServers on :3000/:3001/:3002/:3003
    │ Real Module Federation wiring (no mocks)
    │ browse → add-to-cart → checkout → confirm
    ▼ Pass: 2–5 min smoke | 10–20 min full
```

### Flow 8 — Test Isolation Matrix

| What is tested | Tool | Module Federation | Network calls | Speed |
|---|---|---|---|---|
| `<ProductCard>` renders correctly | Jest + RTL | Mocked (`moduleNameMapper`) | Mocked (MSW) | ~50ms |
| Listing standalone app flow | Jest + RTL | Not needed (direct import) | Mocked (MSW) | ~500ms |
| `remoteEntry.js` exposes correct keys | Contract test | Real built artifact | None | ~5s |
| Container loads all 3 remotes at routes | Playwright | REAL (devServer) | Real / MSW | ~30s |
| Full shopping journey (browse→cart→checkout) | Playwright | REAL (devServer) | Real / stub API | ~60s |
| Cart count persists across navigation | Playwright | REAL (devServer) | Real / localStorage | ~20s |

### Flow 8 — Quality Engineer's Test Pyramid Ratios

```
                        /\
                       /  \    10% — E2E (Playwright)
                      / E2E\   Full journey, real MF wiring
                     /──────\
                    /        \  20% — Integration / Contract
                   / Contract \  remoteEntry contract + standalone MFE
                  /────────────\
                 /              \
                /   Unit Tests   \  70% — Unit (Jest + RTL)
               /  (per component) \  Pure component, no MF, no network
              /────────────────────\

Principal QE Rule:
  An inverted pyramid (mostly E2E) → brittle, slow CI pipeline.
  A missing contract layer → exposes key renames silently break production.
  A missing unit base → component regressions caught too late = expensive.
```

---

*Generated 2026-03-06 · Principal architect and quality engineer analysis of `Micro-Frontend Architecture with React` (React 18 · Webpack Module Federation · Tailwind CSS · React Router DOM · Playwright · Jest · React Testing Library)*
