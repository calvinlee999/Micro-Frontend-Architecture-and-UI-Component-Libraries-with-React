# FinTech Enterprise Micro-Frontend Architecture — Architecture Reference

> **Platform:** Digital Banking & Wealth Platform · Webpack Module Federation · React 18 · Design System (Storybook 8) · TypeScript · Tailwind CSS  
> **Perspective:** Principal Front-End Solution Architect · Principal Front-End Quality Engineer  
> **Regulatory scope:** PCI-DSS Level 1 · SOC 2 Type II · PSD2 / Open Banking · WCAG 2.1 AA  
> **Principal architect view:** MFE topology, Design System library architecture, Module Federation wiring, OAuth2/OIDC auth layer, shared dependency strategy, PCI-DSS boundary, feature flags, audit trail, accessibility gates, multi-region deployment, and testing pyramid.

---

## 1. Overall System Architecture

Six independently deployable applications form the **Digital Banking & Wealth Platform**, integrated at runtime via Webpack Module Federation. A shared **Design System Library** (`@fintechbank/design-system`) is published to a private npm registry and consumed as a build-time dependency by all MFEs. The `shell` is the **host** — it owns the authentication guard, global navigation, routing, the feature-flag client, and the audit trail dispatcher. The five **remotes** each map to a distinct regulated domain.

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                                    │
│                                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Shell — Auth-Gated Host (port 3000)                                                │  │
│  │  OAuth2 PKCE · React Router · Global Nav · Feature Flags · Audit Dispatcher        │  │
│  │                                                                                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐  │  │
│  │  │ /dashboard  │  │ /payments   │  │ /trading    │  │ /compliance              │  │  │
│  │  │ ↓ lazy load │  │ ↓ lazy load │  │ ↓ lazy load │  │ ↓ lazy load            │  │  │
│  │  │ Dashboard   │  │ Payments    │  │ Trading     │  │ Compliance MFE          │  │  │
│  │  │ MFE :3001   │  │ MFE :3002   │  │ MFE :3003   │  │ (KYC/AML) :3004        │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └──────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                             │
│  Shared singletons (Module Federation):  react · react-dom · react-router-dom             │
│  Shared build-time (npm package):        @fintechbank/design-system (Storybook 8)          │
│  Auth context (shared singleton):        @fintechbank/auth-context                         │
│  Feature flags (shared singleton):       @fintechbank/feature-flags (LaunchDarkly SDK)     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Platform runtime topology:**

```
User authenticates via OAuth2 PKCE (Identity Provider)
    │
    ▼
Shell (port 3000) — validates id_token · stores access_token in memory (NOT localStorage)
    │
    ▼
React Router matches route → AuthGuard checks token validity
    │
    ▼
React.lazy() + Suspense triggers dynamic import of domain MFE
    │
    ▼
Browser fetches remoteEntry.js from MFE CDN origin
    │
    ▼
Module Federation runtime resolves shared deps (react singleton, auth-context singleton)
    │
    ▼
Domain MFE component mounts — inherits auth context, feature flags, audit dispatcher
    │
    ▼
MFE makes API call with Authorization: Bearer <access_token> (from auth context)
```

---

## 2. Module Federation Topology

```
┌───────────────── HOST ───────────────────────────────────────────────────────────────────────┐
│  shell (port 3000)                                                                           │
│                                                                                              │
│  ModuleFederationPlugin {                                                                    │
│    name: "shell",                                                                            │
│    remotes: {                                                                                │
│      dashboard:  "dashboard@<dashboard-cdn>/remoteEntry.js",                               │
│      payments:   "payments@<payments-cdn>/remoteEntry.js",                                 │
│      trading:    "trading@<trading-cdn>/remoteEntry.js",                                   │
│      compliance: "compliance@<compliance-cdn>/remoteEntry.js",                             │
│    },                                                                                        │
│    shared: {                                                                                 │
│      react:                     { singleton: true, requiredVersion: "^18.3.0" },           │
│      "react-dom":               { singleton: true, requiredVersion: "^18.3.0" },           │
│      "react-router-dom":        { singleton: true, requiredVersion: "^6.22.0" },           │
│      "@fintechbank/auth-context":   { singleton: true },                                   │
│      "@fintechbank/feature-flags":  { singleton: true },                                   │
│      "@fintechbank/audit-client":   { singleton: true },                                   │
│    }                                                                                         │
│  }                                                                                           │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
        │                │                   │                      │
        ▼                ▼                   ▼                      ▼
┌───────────────┐ ┌──────────────┐ ┌────────────────┐ ┌─────────────────────┐
│ dashboard     │ │ payments     │ │ trading        │ │ compliance          │
│ (port 3001)   │ │ (port 3002)  │ │ (port 3003)    │ │ (port 3004)         │
│               │ │              │ │                │ │                     │
│ exposes:      │ │ exposes:     │ │ exposes:       │ │ exposes:            │
│  "./App"      │ │  "./App"     │ │  "./App"       │ │  "./App"            │
│ (accounts,    │ │ (send money, │ │ (market data,  │ │ (KYC forms,         │
│  portfolio    │ │  history,    │ │  order book,   │ │  AML checks,        │
│  summary)     │ │  open bank)  │ │  portfolio)    │ │  doc upload)        │
└───────────────┘ └──────────────┘ └────────────────┘ └─────────────────────┘

All four remotes consume: @fintechbank/design-system (npm — build-time dependency)
```

**Why `@fintechbank/auth-context` is a shared singleton:**  
The access token lives exclusively in memory inside the auth context — never in `localStorage` or `sessionStorage` (XSS mitigation). Every MFE that needs to attach `Authorization: Bearer` headers reads from the same auth context instance. If it were not a singleton, each MFE would hold a separate token copy that desynchronises on silent refresh, causing 401s mid-session.

---

## 3. Design System Library Architecture (Storybook 8)

> **Source:** Inspired by the LinkedIn Learning course *Building Scalable React UI Component Libraries with Storybook*.  
> **Package:** `@fintechbank/design-system` — published to a private npm registry (GitHub Packages / JFrog Artifactory).  
> **Principal view:** The Design System is the single source of truth for all UI primitives. Every MFE consumes it as a versioned dependency. No MFE builds its own buttons, inputs, or colour palette.

### 3.1 Atomic Design Hierarchy

```
@fintechbank/design-system
    │
    ├── tokens/              ← Design tokens (CSS custom properties + TS constants)
    │   ├── colours.ts       ← Brand palette, semantic aliases (--color-danger, etc.)
    │   ├── typography.ts    ← Font scale, weights, line-heights
    │   ├── spacing.ts       ← 4px grid — 4, 8, 12, 16, 24, 32, 48, 64
    │   └── motion.ts        ← Easing curves, duration scale (regulatory: no animation > 200ms)
    │
    ├── atoms/               ← Indivisible primitives — no domain knowledge
    │   ├── Button/          ← primary | secondary | ghost | destructive variants
    │   ├── Input/           ← text | number | currency | masked (sort code, IBAN)
    │   ├── Badge/           ← status badges: pending | settled | rejected | flagged
    │   ├── Icon/            ← SVG sprite wrapper — accessible title + aria-hidden
    │   ├── Avatar/          ← User / account avatar
    │   └── Spinner/         ← Accessible loading indicator (aria-busy, aria-label)
    │
    ├── molecules/           ← Compositions of atoms — still domain-agnostic
    │   ├── FormField/       ← Input + Label + InlineError (WCAG 2.1 SC 3.3.1)
    │   ├── CurrencyInput/   ← Input + currency symbol + locale formatting
    │   ├── DataTable/       ← Sortable, paginated table with aria-sort
    │   ├── StatusCard/      ← Icon + Badge + amount — transaction summary unit
    │   └── Alert/           ← info | warning | error | success — role="alert"
    │
    ├── organisms/           ← Domain-aware composed layouts
    │   ├── AccountHeader/   ← Account name + balance + masked account number
    │   ├── TransactionRow/  ← Merchant logo + description + amount + status badge
    │   ├── PaymentForm/     ← Multi-field payment entry — uses FormField molecules
    │   └── KYCDocumentCard/ ← Document type + upload zone + verification status
    │
    └── storybook/           ← Storybook 8 configuration
        ├── .storybook/      ← main.ts | preview.ts | manager.ts
        ├── chromatic.config.ts ← Visual regression config (Chromatic)
        └── a11y.config.ts   ← axe-core accessibility addon config
```

### 3.2 Design Token Strategy

```ts
// tokens/colours.ts — semantic alias layer over brand palette
export const tokens = {
  // Brand primitives (never used directly in components)
  palette: {
    cobalt50:  "#EBF2FF",
    cobalt500: "#1A56DB",
    cobalt900: "#0C2461",
    red500:    "#E02424",
    green500:  "#057A55",
    amber500:  "#C27803",
    neutral50: "#F9FAFB",
  },
  // Semantic aliases — what components reference
  color: {
    primary:          "var(--color-cobalt500)",
    danger:           "var(--color-red500)",
    success:          "var(--color-green500)",
    warning:          "var(--color-amber500)",
    surfaceDefault:   "var(--color-neutral50)",
    textPrimary:      "var(--color-neutral900)",
    textSecondary:    "var(--color-neutral600)",
  },
} as const;
```

**Principal rule:** Components must only reference semantic tokens (`color.danger`), never palette primitives (`palette.red500`). This allows white-labelling (swapping the token values at the CSS custom property layer) without touching component code.

### 3.3 Storybook as Living Specification

```
Storybook 8 serves three audiences simultaneously:
┌───────────────────────────────────────────────────────────────┐
│  DESIGNERS  — Visual preview, token showcase, variant matrix  │
│  ENGINEERS  — Copy-paste usage, props API, do/don't examples  │
│  QA / COMPLIANCE — Accessibility audit, WCAG status per story │
└───────────────────────────────────────────────────────────────┘

Each component story covers:
  1. Default state
  2. All named variants
  3. Error / disabled states
  4. Accessibility panel (axe-core violations = CI fail)
  5. Interaction tests (play() function — user events in story)
  6. Chromatic snapshot (visual regression baseline)
```

```ts
// atoms/Button/Button.stories.tsx — Production-grade story format
import type { Meta, StoryObj } from "@storybook/react";
import { userEvent, within, expect } from "@storybook/test";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "Atoms/Button",
  component: Button,
  tags: ["autodocs"],          // auto-generate Props docs page
  parameters: {
    a11y: { element: "#storybook-root" },  // axe-core on every story
    chromatic: { delay: 300 },             // wait for animations
  },
};
export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { variant: "primary", children: "Confirm Payment" },
};

export const DestructiveWithInteraction: Story = {
  args: { variant: "destructive", children: "Cancel Transfer" },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const btn = canvas.getByRole("button", { name: /cancel transfer/i });
    await userEvent.click(btn);
    await expect(btn).toHaveFocus();  // verify focus management after click
  },
};
```

### 3.4 Component Library CI Pipeline

```
PR opened to @fintechbank/design-system repo
    │
    ├── TypeScript typecheck (tsc --noEmit)
    ├── ESLint + Prettier
    ├── Jest unit tests (component logic, hook behaviour)
    ├── Storybook interaction tests (play() functions)
    ├── axe-core accessibility scan (0 violations = pass)
    ├── Chromatic visual snapshot diff  ← blocks merge if unexpected diff
    │     └── Requires designer sign-off on UI changes
    └── npm publish (semver bump) → GitHub Packages
              │
              ▼
    MFEs upgrade via Renovate bot PR (weekly minor/patch auto-merge)
```

---

## 4. Authentication and Authorization Layer

### 4.1 OAuth2 PKCE Flow

```
User opens platform
    │
    ▼
Shell bootstrap — checks memory for valid access_token
    │
    ├── Token present + not expiring < 60s? → Continue (silent path)
    │
    └── No token / near-expiry?
            │
            ▼
        Generate PKCE code_verifier + code_challenge (SHA-256)
        Redirect to Identity Provider (IdP) /authorize endpoint
            │
            ▼
        User authenticates (password + TOTP / biometric)
            │
            ▼
        IdP redirects back with authorization_code
            │
            ▼
        Shell POST /token { code, code_verifier, client_id }
            │
            ▼
        IdP returns: access_token (15 min TTL) + id_token + refresh_token (httpOnly cookie)
            │
            ▼
        Shell stores access_token in React auth context (memory only — NOT localStorage)
        Shell decodes id_token claims → user profile (name, roles, permitted_accounts)
            │
            ▼
        Shell initialises: audit client (userId), feature flag client (userId, roles)
            │
            ▼
        React tree renders — AuthContext.Provider wraps entire app
```

### 4.2 Route-Level Authorization Guard

```tsx
// shell/src/components/AuthGuard.tsx
interface AuthGuardProps {
  requiredRole: "customer" | "relationship_manager" | "compliance_officer";
  children: React.ReactNode;
}

export const AuthGuard: React.FC<AuthGuardProps> = ({ requiredRole, children }) => {
  const { user, isAuthenticated } = useAuthContext();

  if (!isAuthenticated) return <Navigate to="/login" replace />;
  if (!user.roles.includes(requiredRole)) return <ForbiddenPage />;
  return <>{children}</>;
};

// Usage in shell routing:
<Route
  path="/compliance/*"
  element={
    <AuthGuard requiredRole="compliance_officer">
      <Suspense fallback={<PageLoader />}>
        <ComplianceMFE />
      </Suspense>
    </AuthGuard>
  }
/>
```

### 4.3 Token Security Rules

| Concern | Rule | Rationale |
|---|---|---|
| Token storage | Memory only (React context) | localStorage/sessionStorage is readable by XSS; memory is not |
| Refresh token | httpOnly cookie ONLY | JS cannot read httpOnly cookies — XSS cannot steal refresh token |
| Token transmission | `Authorization: Bearer` header ONLY | Never in URL query params (logged in access logs) |
| Token validation | Shell validates on every route change | Expired token caught before domain MFE mounts |
| Silent refresh | 60s before expiry trigger | Prevents mid-flow 401s on slow Payments forms |
| Logout | Revoke refresh token + clear auth context + navigate to /login | Clears all in-memory tokens |

---

## 5. Layer Summary Tables

> End-to-end breakdown of each application.  
> Ordered: Shell (Host) → Design System → Dashboard → Payments → Trading → Compliance → Shared Infrastructure → Test Layer.

---

### 5.0 Shell — Auth-Gated Host Application

> **Role:** Platform entry point — owns the OAuth2 PKCE auth flow, global navigation, React Router, Suspense boundaries, feature flag client initialisation, and audit trail dispatcher. Provides shared context (auth, flags, audit) to all remotes.  
> **Pattern:** Thin shell — zero business logic. All domain logic lives in remotes.  
> **Key Features:** Module Federation host · OAuth2 PKCE · React Router DOM · Tailwind CSS · React.lazy + Suspense · Auth/FeatureFlag/Audit singletons.

#### 5.0a — Shell Responsibility Boundary

| Concern | Shell Owns | Remote Owns |
|---|---|---|
| OAuth2 PKCE auth flow | ✅ | ❌ |
| access_token in memory | ✅ (AuthContext) | ❌ reads via useAuthContext() |
| Browser URL / history | ✅ BrowserRouter | ❌ Must not define BrowserRouter |
| Global navigation bar | ✅ | ❌ |
| Route definitions | ✅ | ❌ |
| AuthGuard per route | ✅ | ❌ |
| Suspense fallback + Error Boundary | ✅ | ❌ |
| Feature flag client initialisation | ✅ | ❌ reads via useFeatureFlag() |
| Audit event dispatcher | ✅ initialises | ✅ dispatches domain events |
| Domain UI (account balances, payment form) | ❌ | ✅ |
| Domain API calls | ❌ | ✅ |
| Internal sub-routes | ❌ | ✅ (via Routes inside remote) |

#### 5.0b — webpack.config.js — Module Federation Host Configuration

```ts
// shell/webpack.config.ts
new ModuleFederationPlugin({
  name: "shell",
  remotes: {
    dashboard:  "dashboard@<dashboard-cdn-url>/remoteEntry.js",
    payments:   "payments@<payments-cdn-url>/remoteEntry.js",
    trading:    "trading@<trading-cdn-url>/remoteEntry.js",
    compliance: "compliance@<compliance-cdn-url>/remoteEntry.js",
  },
  shared: {
    react:                        { singleton: true, requiredVersion: "^18.3.0" },
    "react-dom":                  { singleton: true, requiredVersion: "^18.3.0" },
    "react-router-dom":           { singleton: true, requiredVersion: "^6.22.0" },
    "@fintechbank/auth-context":  { singleton: true, requiredVersion: "^3.0.0" },
    "@fintechbank/feature-flags": { singleton: true, requiredVersion: "^2.0.0" },
    "@fintechbank/audit-client":  { singleton: true, requiredVersion: "^1.0.0" },
  },
})
```

#### 5.0c — Shell Bootstrap Pattern (Bootstrap Indirection + Auth Init)

```ts
// shell/src/index.ts — async boundary for Module Federation negotiation
import("./bootstrap");

// shell/src/bootstrap.tsx — auth + provider initialisation
import React from "react";
import ReactDOM from "react-dom/client";
import { AuthProvider } from "@fintechbank/auth-context";
import { FeatureFlagProvider } from "@fintechbank/feature-flags";
import { AuditProvider } from "@fintechbank/audit-client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <AuthProvider config={authConfig}>       {/* OAuth2 PKCE initialisation */}
      <FeatureFlagProvider>                  {/* LaunchDarkly SDK — user context set after auth */}
        <AuditProvider>                      {/* Compliance audit trail */}
          <App />
        </AuditProvider>
      </FeatureFlagProvider>
    </AuthProvider>
  </React.StrictMode>
);
```

---

### 5.1 Dashboard — Account Overview Micro-Frontend

> **Role:** Displays account balances, portfolio summary, recent transactions, and personalised financial insights. The first screen users see after authentication.  
> **Pattern:** Remote MFE. Read-heavy — primarily data visualisation. Consumes Account API and Analytics API.  
> **Key Features:** Module Federation remote · React Query (server state) · Recharts (accessible charts) · Design System components.

#### 5.1a — Component Responsibility Matrix

| Component | Responsibility | Design System Atom |
|---|---|---|
| `<AccountSummaryCard>` | Balance + account number (masked) | `AccountHeader` organism |
| `<PortfolioChart>` | Holdings breakdown (Recharts area chart) | Custom — accessible SVG |
| `<TransactionFeed>` | Last 10 transactions, paginated | `TransactionRow` organism |
| `<InsightBanner>` | AI-driven spending insight | `Alert` molecule |
| `<QuickActions>` | Send / Receive / Statement buttons | `Button` atom |

#### 5.1b — Data Fetching Pattern (React Query)

```ts
// dashboard/src/hooks/useAccountSummary.ts
import { useQuery } from "@tanstack/react-query";
import { useAuthContext } from "@fintechbank/auth-context";

export const useAccountSummary = (accountId: string) => {
  const { getAccessToken } = useAuthContext();

  return useQuery({
    queryKey: ["account-summary", accountId],
    queryFn: async () => {
      const token = await getAccessToken();   // handles silent refresh
      const res = await fetch(`/api/accounts/${accountId}/summary`, {
        headers: { Authorization: `Bearer ${token}` },
      });
      if (!res.ok) throw new ApiError(res.status, await res.json());
      return res.json() as Promise<AccountSummary>;
    },
    staleTime: 30_000,          // 30s — balance data is semi-real-time
    retry: (count, error) =>
      error instanceof ApiError && error.status !== 401 && count < 2,
  });
};
```

---

### 5.2 Payments — Regulated Payment Micro-Frontend

> **Role:** Send money (domestic + international), view payment history, manage standing orders, and Open Banking payment initiation. Highest regulatory exposure domain.  
> **Pattern:** Remote MFE. Payments data and submission are PCI-DSS controlled — card data entry uses a sandboxed iframe hosted on an isolated PCI scope subdomain.  
> **Key Features:** Module Federation remote · PCI-DSS iframe for card data · PSD2 SCA (Strong Customer Authentication) step-up · Audit event dispatch on every mutation.

#### 5.2a — PCI-DSS Boundary Architecture

```
Payments MFE (/payments) — NOT in PCI scope
    │
    │  User clicks "Pay with Card"
    │
    ▼
Shell navigates to PCI-scope route: /payments/card-capture
    │
    ▼
<PCIFrame> renders — isolated iframe origin:
  https://pci.fintechbank.com  (separate subdomain, separate infra)
    │
    │  Parent shell and Payments MFE can NOT read the iframe DOM
    │  Card number, CVV, expiry live ONLY in the PCI iframe
    │
    ▼
User enters card data in iframe
    │
    ▼
PCI iframe POSTs tokenisation request to PCI vault
  → returns opaque payment_method_token (no raw card data crosses)
    │
    ▼
PCI iframe postMessage({ type: "PAYMENT_TOKEN", token }) to parent
    │ postMessage target origin MUST be "https://app.fintechbank.com" (strict)
    ▼
Payments MFE receives token via window.addEventListener("message", handler)
    │ validates event.origin strictly
    ▼
Payments MFE submits payment_method_token to Payment API
  POST /api/payments { amount, recipient, payment_method_token }
    │
    ▼
Payment API processes via payment processor — returns transaction_id
```

**PCI-DSS principle:** Raw card data (PAN, CVV, expiry) must never touch the React application, the browser's JS heap, or any CDN-served asset. The iframe boundary enforces this — the parent window JavaScript cannot inspect or intercept iframe DOM nodes when the iframe is cross-origin.

#### 5.2b — Payment Audit Hook

```ts
// payments/src/hooks/usePaymentAudit.ts
import { useAuditClient } from "@fintechbank/audit-client";

export const usePaymentAudit = () => {
  const audit = useAuditClient();

  return {
    onPaymentInitiated: (amount: number, recipient: string) =>
      audit.dispatch({
        event:    "PAYMENT_INITIATED",
        domain:   "payments",
        severity: "INFO",
        payload:  { amount, recipientMasked: maskAccountNumber(recipient) },
      }),
    onPaymentSubmitted: (transactionId: string) =>
      audit.dispatch({
        event:    "PAYMENT_SUBMITTED",
        domain:   "payments",
        severity: "INFO",
        payload:  { transactionId },
      }),
    onPaymentFailed: (error: ApiError) =>
      audit.dispatch({
        event:    "PAYMENT_FAILED",
        domain:   "payments",
        severity: "WARN",
        payload:  { errorCode: error.code },
      }),
  };
};
```

**Principal rule:** Every PSD2-regulated user action (initiate payment, authorise payment, cancel standing order) must produce an immutable audit event. Audit events are append-only, signed, and stored independently of the application database.

---

### 5.3 Trading — Market Data and Order Management Micro-Frontend

> **Role:** Real-time market data display (WebSocket feed), portfolio performance, order placement (equities, ETFs). Subject to MiFID II best-execution and suitability disclosure requirements.  
> **Pattern:** Remote MFE. Highest performance demands — WebSocket streaming, virtualised data tables (tens of thousands of rows), 60 fps chart updates.  
> **Key Features:** Module Federation remote · WebSocket (market feed) · TanStack Virtual (row virtualisation) · Recharts / D3 (candlestick charts) · Feature flag gating for new order types.

#### 5.3a — Feature Flag Gate (Order Types)

```ts
// trading/src/components/OrderPanel.tsx
import { useFeatureFlag } from "@fintechbank/feature-flags";

export const OrderPanel: React.FC<{ instrument: Instrument }> = ({ instrument }) => {
  const optionsEnabled = useFeatureFlag("trading.options_chain.enabled");
  const cryptoEnabled  = useFeatureFlag("trading.crypto_pairs.enabled");

  return (
    <section aria-labelledby="order-panel-heading">
      <h2 id="order-panel-heading">Place Order</h2>
      <EquityOrderForm instrument={instrument} />
      {optionsEnabled && <OptionsChain instrument={instrument} />}
      {cryptoEnabled  && <CryptoOrderForm instrument={instrument} />}
    </section>
  );
};
```

**Principal rule:** New financial instrument types (options, crypto, structured products) must be gated behind feature flags. This allows:
1. Regulatory approval to precede technical deployment.
2. Staged rollout — 1% of users, then 10%, then 100%.
3. Instant kill switch without a code deployment.

---

### 5.4 Compliance — KYC and AML Micro-Frontend

> **Role:** Know Your Customer (KYC) identity verification, Anti-Money Laundering (AML) check status display, document upload (passport, proof of address), and adverse media screening results. Accessed by customers and by compliance officers (role-gated).  
> **Pattern:** Remote MFE. Separate deployment cadence from other MFEs — compliance workflow changes are driven by regulatory timeline, not product timeline.

#### 5.4a — Role-Gated Views

| Route | Customer view | Compliance Officer view (role: compliance_officer) |
|---|---|---|
| `/compliance/status` | Own KYC status + outstanding documents | All customers with pending KYC |
| `/compliance/documents` | Upload own documents | Review + approve / reject submissions |
| `/compliance/aml` | Own screening status | Full adverse media report + override controls |
| `/compliance/audit-log` | Own activity log | Platform-wide audit log with filters |

---

### 5.5 Shared Module Design

#### 5.5a — Shared Dependency Classification

```
RUNTIME SINGLETONS (Module Federation shared{} — one instance in JS heap)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  react, react-dom             → Hooks + Context require single React root
  react-router-dom             → One BrowserRouter = one history stack
  @fintechbank/auth-context    → Single access token in memory
  @fintechbank/feature-flags   → Single LaunchDarkly client connection
  @fintechbank/audit-client    → Single audit queue, batched flush

BUILD-TIME DEPENDENCY (npm package — consumed at compile time)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  @fintechbank/design-system   → UI components + design tokens
  @fintechbank/api-client      → Typed API fetch wrapper (OpenAPI-generated)

NOT SHARED — EACH MFE OWNS ITS OWN COPY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  @tanstack/react-query        → Can duplicate — stateless per MFE cache
  recharts / d3                → Can duplicate — rendering only
  Internal components          → Domain-specific, not re-used
  Internal hooks               → useAccountSummary, useOrderBook
```

#### 5.5b — Design System Version Governance

| Situation | SLA | Process |
|---|---|---|
| Patch (bug fix, accessibility fix) | 24 hrs | Auto-merge via Renovate across all MFEs |
| Minor (new component, token addition) | 1 sprint | Renovate PR — team lead review |
| Major (breaking API, token rename) | 1 quarter | RFC process, migration guide, MFE team coordination |
| Security patch (XSS fix in component) | 2 hrs (P0) | Emergency all-team deployment |

**Principal rule:** The Design System version is a platform contract. Major version bumps require a coordinated migration. The Design System team publishes a codemod for breaking changes (e.g., renamed props, removed variants).

---

### 5.6 Infrastructure and Deployment

> **Pattern:** Multi-region active-active (us-east-1 primary, eu-west-1 secondary — GDPR data residency) · Blue-green deployment per MFE · Canary traffic splitting · Independent CI/CD pipeline per domain team.

#### 5.6a — Port Allocation (Local Development)

| Application | Port | Entry URL | remoteEntry URL |
|---|---|---|---|
| `shell` | 3000 | `http://localhost:3000` | N/A (host) |
| `dashboard` | 3001 | `http://localhost:3001` | `http://localhost:3001/remoteEntry.js` |
| `payments` | 3002 | `http://localhost:3002` | `http://localhost:3002/remoteEntry.js` |
| `trading` | 3003 | `http://localhost:3003` | `http://localhost:3003/remoteEntry.js` |
| `compliance` | 3004 | `http://localhost:3004` | `http://localhost:3004/remoteEntry.js` |
| `design-system` (Storybook) | 6006 | `http://localhost:6006` | N/A (npm, not MF) |

#### 5.6b — Canary Deployment Flow

```
Payments team merges to main → triggers payments CI pipeline
    │
    ▼
CI: typecheck → lint → unit test → Storybook interaction test → accessibility scan
    │
    ▼
CI: Playwright smoke E2E against staging environment
    │
    ▼
Build: webpack production bundle
  dist/remoteEntry.js               (~2 KB manifest)
  dist/payments.[NEW_HASH].js       (~300 KB domain bundle)
    │
    ▼
Deploy dist/ to CDN — CANARY origin:
  https://payments-cdn.fintechbank.com/canary/remoteEntry.js
    │
    ▼
CDN edge rule: route 5% of traffic to canary/remoteEntry.js
  Check: error rates, p99 latency, Core Web Vitals
    │
    ├── Metrics nominal for 30 min? → promote to 100% (swap remoteEntry.js path)
    │
    └── Error rate spike? → rollback: point remoteEntry.js back to previous hash
```

#### 5.6c — CDN Cache Strategy (FinTech Rules)

| Asset | Cache-Control | Reason |
|---|---|---|
| `remoteEntry.js` | `no-cache, no-store` | Version manifest — zero TTL in FinTech (regulatory audit trail) |
| `payments.[hash].js` | `max-age=31536000, immutable` | Content-hashed — hash changes on rebuild |
| `shell/index.html` | `no-cache` | Entry point must always be current |
| API responses | `no-store` | Financial data must never be cached on CDN |

**FinTech tightening vs generic e-commerce:** `no-store` on `remoteEntry.js` (not just `no-cache`) prevents any intermediate proxy or service worker from storing the manifest. This is required to ensure that every page load fetches the definitive current version — important for audit trail accuracy.

#### 5.6d — Build Pipeline per Domain Team (GitHub Actions)

```yaml
# .github/workflows/payments-deploy.yml
name: Deploy Payments MFE
on:
  push:
    paths: ["packages/payments/**"]
    branches: [main]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
        working-directory: packages/payments

      - name: Type Check
        run: npx tsc --noEmit
        working-directory: packages/payments

      - name: Unit & Integration Tests
        run: npm test -- --watchAll=false --coverage --coverageThreshold='{"global":{"lines":80}}'
        working-directory: packages/payments

      - name: Accessibility Scan (axe-core)
        run: npm run test:a11y
        working-directory: packages/payments

      - name: Build
        run: npm run build
        working-directory: packages/payments

      - name: Canary Deploy
        run: |
          aws s3 sync packages/payments/dist s3://payments-cdn-bucket/canary \
            --cache-control "max-age=31536000,immutable"
          aws s3 cp packages/payments/dist/remoteEntry.js \
            s3://payments-cdn-bucket/canary/remoteEntry.js \
            --cache-control "no-cache,no-store"

      - name: Playwright Smoke E2E (Staging)
        run: npx playwright test e2e/payments.smoke.spec.ts
        env:
          PLAYWRIGHT_BASE_URL: https://staging.fintechbank.com
```

---

### 5.7 Testing Strategy

> **Role:** Multi-level quality gate with FinTech-specific additions: accessibility auditing (WCAG 2.1 AA), visual regression (Chromatic), security-focused mutation testing, Storybook interaction tests, and compliance boundary contract tests.  
> **Pattern:** Storybook interaction → Vitest/Jest unit → Visual regression (Chromatic) → Accessibility (axe) → Contract → E2E (Playwright).

#### 5.7a — Test Layer Overview

| Layer | Scope | Tool | Speed | Trigger |
|---|---|---|---|---|
| Storybook Interaction | Component user events in isolation | Storybook play() + Testing Library | < 1 min | Every commit (Storybook CI) |
| Unit | Single component / hook in isolation | Vitest + React Testing Library | < 30 s | Every commit |
| Visual Regression | Pixel-diff vs Chromatic baseline | Chromatic (Storybook snapshots) | 2–5 min | Every PR |
| Accessibility | WCAG 2.1 AA violations | axe-core (jest-axe + Storybook addon) | < 1 min | Every PR |
| Integration | MFE standalone with mocked APIs | Vitest (JSDOM) + RTL + MSW | < 2 min | Every PR |
| Contract | Does remoteEntry.js expose expected keys? | Custom manifest assertion | < 30 s | Every PR |
| E2E Smoke | Critical payment journeys | Playwright (real browser) | 5 min | Every PR |
| E2E Regression | Full cross-MFE journeys + edge cases | Playwright | 15–20 min | Nightly |
| Performance Budget | Core Web Vitals (LCP, CLS, INP) | Lighthouse CI | 5 min | Every PR |

#### 5.7b — Accessibility Testing Pattern

```tsx
// design-system/src/atoms/Button/__tests__/Button.a11y.test.tsx
import { render } from "@testing-library/react";
import { axe, toHaveNoViolations } from "jest-axe";
import { Button } from "../Button";

expect.extend(toHaveNoViolations);

describe("Button — WCAG 2.1 AA", () => {
  it("has no accessibility violations in default state", async () => {
    const { container } = render(<Button variant="primary">Confirm Payment</Button>);
    expect(await axe(container)).toHaveNoViolations();
  });

  it("has no accessibility violations in disabled state", async () => {
    const { container } = render(
      <Button variant="primary" disabled aria-describedby="hint">
        Submit
      </Button>
    );
    expect(await axe(container)).toHaveNoViolations();
  });

  it("loading spinner is announced to screen readers", async () => {
    const { container, getByRole } = render(<Button loading>Processing...</Button>);
    expect(getByRole("status")).toBeInTheDocument();  // aria-live region
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

#### 5.7c — Performance Budget (Lighthouse CI)

```js
// lighthouserc.js
module.exports = {
  ci: {
    collect: { url: ["http://localhost:3000/", "http://localhost:3000/dashboard"] },
    assert: {
      assertions: {
        "categories:performance":    ["error", { minScore: 0.9  }],   // 90+ Lighthouse score
        "categories:accessibility":  ["error", { minScore: 1.0  }],   // 100 — zero violations
        "first-contentful-paint":    ["error", { maxNumericValue: 1500 }],  // < 1.5s
        "largest-contentful-paint":  ["error", { maxNumericValue: 2500 }],  // < 2.5s (Core Web Vitals)
        "cumulative-layout-shift":   ["error", { maxNumericValue: 0.1  }],  // < 0.1 (CWV)
        "total-blocking-time":       ["error", { maxNumericValue: 300  }],  // < 300ms
      },
    },
  },
};
```

**FinTech additional rule:** Accessibility score must be 100 on every PR — financial services applications are subject to the European Accessibility Act (EAA) effective June 2025. A 0.9 accessibility threshold is not acceptable.

#### 5.7d — Module Federation Contract Test

```ts
// shell/tests/mf-contract.test.ts — verifies remote exposes contract
import path from "path";
import fs from "fs";

const EXPECTED_CONTRACTS = {
  dashboard:  ["./App"],
  payments:   ["./App"],
  trading:    ["./App"],
  compliance: ["./App"],
};

describe("Module Federation Contract Verification", () => {
  Object.entries(EXPECTED_CONTRACTS).forEach(([remote, exposedKeys]) => {
    it(`${remote} remoteEntry.js exposes required keys`, () => {
      const remoteEntry = fs.readFileSync(
        path.resolve(__dirname, `../../packages/${remote}/dist/remoteEntry.js`),
        "utf-8"
      );
      exposedKeys.forEach(key => {
        expect(remoteEntry).toContain(key);
      });
    });
  });
});
```

#### 5.7e — Quality Engineer Principal Rules

1. **Accessibility is not optional.** Every component has an axe-core test. Zero violations is a hard CI gate, not a warning. A payment form with an inaccessible submit button is a regulatory defect.

2. **Visual regression protects the design system contract.** Chromatic blocks merge when any component changes visually — requires designer sign-off. This prevents silent Design System drift across MFEs.

3. **Storybook interaction tests replace many integration tests.** The `play()` function runs real user events against the component in a real browser (Storybook uses Playwright internally). This is faster and more isolated than spinning up the full MFE.

4. **Performance budgets are hard gates.** An LCP regression from a new chart library in Trading MFE should fail CI — not be noticed by users. Lighthouse CI runs against a localhost build on every PR.

5. **Contract tests guard the MFE API boundary.** The shell declares it will import `payments/App`. This contract is verified against the built `remoteEntry.js` artifact before any deployment proceeds. Renaming an expose key without updating the shell contract must fail CI — not production.

6. **The test pyramid applies to the Design System too.** `atoms/Button` has more unit tests than `organisms/PaymentForm`. Story tests cover the component API surface. E2E tests verify the organism in context within a real MFE.

---

## 6. Cross-Cutting Concerns

### 6.1 Audit Trail Architecture

```
Every significant user action dispatches an audit event:
    │
    ▼
useAuditClient().dispatch({ event, domain, severity, payload })
    │
    ▼
AuditClient (shared singleton):
    │ buffers events in a local queue (max 50 events / flush every 5s)
    │ flushes to Audit API via POST /api/audit/events (batched)
    │ retries on network failure (up to 3 attempts, exponential backoff)
    ▼
Audit API:
    │ appends to append-only event store (Apache Kafka topic / AWS Kinesis)
    │ events are immutable — no UPDATE or DELETE operations permitted
    │ signs each event with HMAC using audit service key
    ▼
Audit store consumed by:
    ├── Compliance MFE (officer audit log view)
    ├── SIEM (security alerting — anomaly detection)
    └── Regulatory reporting (FCA / GDPR data export)
```

### 6.2 Error Boundary Strategy (FinTech)

```tsx
// shell/src/components/RemoteErrorBoundary.tsx
interface RemoteErrorBoundaryProps {
  remote: string;
  fallback?: React.ReactNode;
  children: React.ReactNode;
}

class RemoteErrorBoundary extends React.Component<RemoteErrorBoundaryProps> {
  state = { hasError: false, errorId: "" };

  static getDerivedStateFromError() {
    return { hasError: true, errorId: crypto.randomUUID() };
  }

  componentDidCatch(error: Error) {
    // Report to observability platform (Datadog / OpenTelemetry)
    telemetry.captureException(error, {
      remote:  this.props.remote,
      errorId: this.state.errorId,
      userId:  authContext.user?.id,   // PII — ensure GDPR-compliant telemetry config
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div role="alert" aria-live="assertive" className="p-8 text-center">
          <p>The {this.props.remote} section is temporarily unavailable.</p>
          <p>Reference: {this.state.errorId}</p>  {/* for support correlation */}
          <Button onClick={() => this.setState({ hasError: false, errorId: "" })}>
            Retry
          </Button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

**FinTech addition vs generic:** The `errorId` is a UUID logged to the telemetry platform and displayed to the user. A support agent can correlate the user-reported reference ID to the exact error event. This is critical for regulated incident investigation.

### 6.3 Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://*.fintechbank.com 'nonce-{nonce}';
  connect-src 'self' https://api.fintechbank.com https://identity.fintechbank.com
              wss://market-data.fintechbank.com;
  frame-src https://pci.fintechbank.com;  — only PCI capture iframe allowed
  img-src 'self' https://cdn.fintechbank.com data:;
  style-src 'self' 'unsafe-inline';       — required for Tailwind CSS in production
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
```

**Principal rule:** CSP is the last line of defence against XSS. `frame-src` explicitly whitelists only the PCI capture subdomain for iframes — no arbitrary third-party iframe injection is possible. `object-src 'none'` disables Flash and plugin execution entirely.

### 6.4 Bundle Size Philosophy (FinTech)

```
Shell initial load (authenticated user):
  react + react-dom:           ~130 KB + ~1 MB (shared — loaded once)
  react-router-dom (shared):   ~50 KB
  auth-context (shared):       ~30 KB (OAuth2 PKCE logic)
  feature-flags (shared):      ~20 KB (LaunchDarkly client)
  design-system (tree-shaken):  ~80 KB (only used atoms)
  shell bundle (nav + routing): ~40 KB
  ─────────────────────────────────────
  Total initial:               ~1.35 MB (TTI target < 2.5s on 4G)

Per-page additions:
  dashboard bundle:            ~250 KB (Recharts tree-shaken)
  payments bundle:             ~300 KB (form logic, validation)
  trading bundle:              ~500 KB (D3 + virtualisation)  ← lazy-loaded only on /trading
  compliance bundle:           ~200 KB (document upload)

Performance budget rule:
  Trading MFE's D3 dependency MUST be lazy-loaded within the MFE.
  No single MFE bundle > 600 KB gzipped.
  LCP must remain < 2.5s even on cached return visits.
```

---

## 7. Architecture Decision Records (ADRs)

### ADR-001: Module Federation over Iframes for MFE Integration (with PCI-DSS exception)

**Decision:** Use Module Federation for all MFE integration except the PCI card capture flow, which uses a cross-origin iframe.

**Rationale:** Module Federation provides shared React singleton (no duplicate hook errors), unified routing (single BrowserRouter), and shared auth context (single access token instance). Iframes are reserved exclusively for the PCI card capture boundary where JS isolation is a hard regulatory requirement — Module Federation components share the same JS heap and cannot provide the PAN data isolation required by PCI-DSS.

---

### ADR-002: Design System as npm Package, Not Module Federation Remote

**Decision:** `@fintechbank/design-system` is published to a private npm registry and consumed as a build-time dependency. It is NOT exposed as a Module Federation remote.

**Alternatives considered:**
- **Module Federation remote** — components would be fetched at runtime and shared as singletons. Rejected: runtime visual regressions are harder to detect; design system bugs surface in production without prior visual regression gate.
- **npm package (chosen)** — each MFE bundles the design system components it uses (tree-shaken). Storybook interaction tests and Chromatic visual regression run before publish. Design system bugs never reach production without a PR approval.

**Consequence:** Design system updates require each MFE to bump its npm dependency and redeploy. Renovate bot automates this for patch/minor releases.

---

### ADR-003: access_token in React Context Memory, refresh_token in httpOnly Cookie

**Decision:** Store the OAuth2 access_token exclusively in React context memory. Store the refresh_token in an httpOnly, SameSite=Strict, Secure cookie.

**Consequence:** Refreshing the page triggers silent token refresh via the httpOnly refresh cookie (the browser sends it automatically to the auth server). The access_token is never persisted beyond the page session.

**Rationale:** localStorage is accessible via any JavaScript — a single XSS payload can exfiltrate tokens and hijack sessions. httpOnly cookies are invisible to JavaScript. Memory storage for the access_token means it's gone on refresh (acceptable — silent refresh recovers it within ~500ms).

---

### ADR-004: Canary Deployment over Big-Bang for Regulated Changes

**Decision:** All Payments and Trading MFE deployments use a canary traffic split (5% → 25% → 100%) with automated metric gates before promotion.

**Rationale:** A broken payment form in a big-bang deployment affects all users simultaneously — a P0 incident in a regulated environment with mandatory CASS/FCA incident reporting obligations. A canary limits blast radius to 5% of users and can be rolled back within seconds (remoteEntry.js pointer swap).

---

### ADR-005: Accessibility Score 100 as Hard CI Gate

**Decision:** Lighthouse CI accessibility score must be 100/100 on every PR. axe-core violations in any component story are a hard CI failure.

**Consequence:** Features with inaccessible UI cannot be merged. Teams must implement accessible components from the first iteration.

**Rationale:** The European Accessibility Act (EAA) applied to financial services from June 2025. A post-hoc accessibility remediation programme across a multi-team MFE platform would cost significantly more than enforcing correctness at the component level from day zero.

---

*Generated 2026-03-06 · Principal Front-End Solution Architect · Principal Front-End Quality Engineer · Digital Banking & Wealth Platform (React 18 · Webpack Module Federation · Storybook 8 · TypeScript · Tailwind CSS · OAuth2 PKCE · PCI-DSS · WCAG 2.1 AA)*

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                    │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Container — Shell / Host (port 3000)                               │  │
│  │  React Router · Global Nav · TailwindCSS                            │  │
│  │                                                                      │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │  │
│  │  │  /listing    │  │  /cart       │  │  /checkout               │  │  │
│  │  │  ↓ lazy load │  │  ↓ lazy load │  │  ↓ lazy load            │  │  │
│  │  │  Listing MFE │  │  Cart MFE    │  │  Checkout MFE           │  │  │
│  │  │  (port 3001) │  │  (port 3002) │  │  (port 3003)            │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Shared singletons: react · react-dom · react-router-dom                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Request flow (runtime):**

```
User navigates to /listing
    │
    ▼
Container (port 3000) — React Router matches route
    │
    ▼
React.lazy() + Suspense triggers dynamic import
    │
    ▼
Browser fetches remoteEntry.js from listing (port 3001)
    │
    ▼
Module Federation runtime resolves shared deps (react singleton)
    │
    ▼
ListingApp component mounts inside Container's React tree
```

---

## 2. Module Federation Topology

```
┌─────────── HOST ─────────────────────────────────────────────────────────────┐
│  container (port 3000)                                                       │
│                                                                              │
│  ModuleFederationPlugin {                                                    │
│    name: "container",                                                        │
│    remotes: {                                                                │
│      listing:  "listing@<listing-url>/remoteEntry.js",                      │
│      cart:     "cart@<cart-url>/remoteEntry.js",                            │
│      checkout: "checkout@<checkout-url>/remoteEntry.js"                     │
│    },                                                                        │
│    shared: { react: singleton, react-dom: singleton,                        │
│              react-router-dom: singleton }                                   │
│  }                                                                           │
└──────────────────────────────────────────────────────────────────────────────┘
        │ fetches at runtime          │                          │
        ▼                             ▼                          ▼
┌────────────────────┐  ┌────────────────────┐  ┌──────────────────────────┐
│  REMOTE: listing   │  │  REMOTE: cart      │  │  REMOTE: checkout        │
│  (port 3001)       │  │  (port 3002)       │  │  (port 3003)             │
│                    │  │                    │  │                          │
│  exposes: {        │  │  exposes: {        │  │  exposes: {              │
│    "./App":        │  │    "./App":        │  │    "./App":              │
│    ListingApp      │  │    CartApp         │  │    CheckoutApp           │
│  }                 │  │  }                 │  │  }                       │
│                    │  │                    │  │                          │
│  shared:           │  │  shared:           │  │  shared:                 │
│   react singleton  │  │   react singleton  │  │   react singleton        │
└────────────────────┘  └────────────────────┘  └──────────────────────────┘
```

**Mental model — Module Federation is a runtime plugin system:**  
Think of it like a browser extension marketplace. The `container` is the browser — it defines the runtime environment and security boundary. Each remote is an extension that the browser can download and run at any time, without knowing the extension's internal code ahead of time.

---

## 3. Routing Strategy Decision Tree

```
User Request
     │
     ▼
Is React Router initialized in Container?
     │
   YES ──────────────────────────────────────────────────────────────┐
     │                         │                        │            │
     ▼                         ▼                        ▼            ▼
Route: /              Route: /listing           Route: /cart   Route: /checkout
     │                         │                        │            │
Home / Nav only        Lazy load Listing MFE   Lazy load Cart   Lazy load Checkout
                       remoteEntry.js          remoteEntry.js   remoteEntry.js
                               │                        │            │
                       Mount <ListingApp>       Mount <CartApp>  Mount <CheckoutApp>
                       inside Suspense           inside Suspense  inside Suspense
                       boundary                 boundary         boundary
```

**Routing Rule — Only the host owns the router:**  
Remotes must NOT define their own top-level `<BrowserRouter>`. If a remote wraps its root in a router, it creates a nested router conflict (two history stacks). Only the `container` defines `<BrowserRouter>`. Remotes use `<Routes>` / `<Route>` only for their own internal sub-navigation, wrapped inside the host's router context.

---

## 4. Layer Summary Tables

> End-to-end breakdown of each application.  
> Each section follows: Role & Design Decisions → ASCII architecture diagram → Detailed tables → Configuration reference.  
> Ordered: Container (Host) → Listing → Cart → Checkout → Shared Infrastructure → Test Layer.

---

### 4.0 Container — Shell Host Application

> **Role:** Application entry point — owns the browser URL, global navigation, React Router, and Suspense boundaries. Fetches remote `remoteEntry.js` files at runtime and mounts remote components into designated route slots.  
> **Pattern:** Shell application (dumb host — zero business logic). All domain logic lives in the remotes.  
> **Key Features:** Webpack Module Federation host · React Router DOM · Tailwind CSS · React.lazy + Suspense · Shared singleton enforcement.

#### 4.0a — Container Responsibility Boundary

| Concern | Container Owns | Remote Owns |
|---|---|---|
| Browser URL / history | ✅ BrowserRouter | ❌ Must not define BrowserRouter |
| Top-level navigation bar | ✅ | ❌ |
| Route definitions (`/listing`, `/cart`, `/checkout`) | ✅ | ❌ |
| Suspense fallback (loading spinner) | ✅ | ❌ |
| Error boundary for remote load failure | ✅ | ❌ |
| Domain UI (product grid, cart items, form) | ❌ | ✅ |
| Internal sub-routes (`/listing/detail/:id`) | ❌ | ✅ (using Routes inside remote) |
| Domain state (cart items, selected product) | ❌ | ✅ or shared via event bus |

#### 4.0b — webpack.config.js — Module Federation Host Configuration

```js
// container/webpack.config.js
new ModuleFederationPlugin({
  name: "container",                       // host identity
  remotes: {
    // Key   = import alias used in code:  import("listing/App")
    // Value = <remoteName>@<remoteEntry.js URL>
    listing:  "listing@<listing-cdn-url>/remoteEntry.js",
    cart:     "cart@<cart-cdn-url>/remoteEntry.js",
    checkout: "checkout@<checkout-cdn-url>/remoteEntry.js",
  },
  shared: {
    react:            { singleton: true },  // ← CRITICAL: one React instance
    "react-dom":      { singleton: true },  // prevents "hooks called in different trees"
    "react-router-dom": { singleton: true } // one router, one history stack
  }
})
```

**Why `singleton: true` is non-negotiable:**  
React's Context API and Hooks require a single React instance in the JS heap. If `container` loads React 18.2.0 and a remote also bundles React 18.2.0 separately, you get TWO instances — hooks from the remote throw `Invalid hook call` because they look for a React instance context that doesn't exist in their copy's runtime. `singleton: true` forces Module Federation to reuse the host's copy for all remotes.

#### 4.0c — Container App Entry (Bootstrap Pattern)

```js
// container/src/index.js  — MUST use bootstrap indirection
// Direct import would prevent Module Federation from negotiating shared modules
import("./bootstrap");

// container/src/bootstrap.js  — actual app initialisation
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

**Why the bootstrap indirection exists:**  
Without the async `import("./bootstrap")`, webpack executes the entry synchronously before Module Federation has had time to negotiate which version of shared modules (react, react-dom) to use. The dynamic import gives webpack an event loop tick to perform the negotiation — without it, remotes may load their own bundled React instead of the shared singleton.

---

### 4.1 Listing — Product Listing Micro-Frontend

> **Role:** Displays the product catalogue. Entirely owns the product browsing experience — grid layout, search/filter UI, product card,  detail navigation.  
> **Pattern:** Remote MFE. Exposes one entry point (`./App`). Can run standalone on port 3001 or be consumed by the container.  
> **Key Features:** webpack Module Federation remote · Tailwind CSS · Standalone dev mode · `./App` exposed entry.

#### 4.1a — Expose Configuration

```js
// listing/webpack.config.js
new ModuleFederationPlugin({
  name: "listing",             // matches the remote key in container config
  filename: "remoteEntry.js",  // manifest file fetched by the host at runtime
  exposes: {
    "./App": "./src/App",      // host imports as: import("listing/App")
  },
  shared: {
    react:              { singleton: true },
    "react-dom":        { singleton: true },
    "react-router-dom": { singleton: true },
  }
})
```

#### 4.1b — Standalone Dev Mode Pattern

```js
// listing/src/index.js — bootstrap for standalone mode
import("./bootstrap");

// listing/src/bootstrap.js
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

// When running standalone (npm start in /listing), renders to its own root.
// When consumed by container, this file is never executed — container 
// imports ./App directly via Module Federation.
ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

**Principal design rule:** Every remote must be runnable as a standalone application. This enables:
1. Independent team development without needing the container running.
2. Independent CI/CD — the remote's pipeline deploys without touching other remotes.
3. Independent testing — unit and integration tests run in isolation.

#### 4.1c — Component Responsibility Matrix

| Component | Responsibility | Notes |
|---|---|---|
| `<App>` | Root entry — exported via Module Federation | Accepts React Router context from host |
| `<ProductGrid>` | Renders the product card collection | Fetches product data internally |
| `<ProductCard>` | Displays single product: image, name, price, CTA | Stateless, driven by props |
| `<SearchBar>` | Text input + filters | Drives local filter state |
| `<CategoryFilter>` | Category selector | Drives local filter state |

---

### 4.2 Cart — Shopping Cart Micro-Frontend

> **Role:** Manages the shopping cart user experience — displays cart items, item quantities, totals, and a checkout entry point.  
> **Pattern:** Remote MFE. Cart state is the most cross-cutting concern in the system — it must be accessible to both Listing (add-to-cart action) and Checkout (order review). See §4.5 for cross-MFE state patterns.  
> **Key Features:** webpack Module Federation remote · Cart state management · Event-bus integration point · Tailwind CSS.

#### 4.2a — Expose Configuration

```js
// cart/webpack.config.js
new ModuleFederationPlugin({
  name: "cart",
  filename: "remoteEntry.js",
  exposes: {
    "./App": "./src/App",
  },
  shared: {
    react:              { singleton: true },
    "react-dom":        { singleton: true },
    "react-router-dom": { singleton: true },
  }
})
```

#### 4.2b — Cross-MFE State Communication Patterns

The cart is the canonical example of shared state in a micro-frontend system. Three strategies, ordered from simplest to most scalable:

```
STRATEGY 1 — Custom Event Bus (used in this project)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Listing MFE                Container              Cart MFE
      │                          │                     │
  "Add to Cart"                  │                     │
  button clicked                 │                     │
      │                          │                     │
      └──── window.dispatchEvent(new CustomEvent(      │
              "cart:add", { detail: { product } }))    │
                                 │                     │
                                 │         window.addEventListener(
                                 │           "cart:add", handler)
                                 │                     │
                                 │         updates cart state
                                 │                     ▼
                                 │         re-renders CartApp

STRATEGY 2 — Shared State Library
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Expose a shared cart store via Module Federation "shared":
    zustand store or Redux Toolkit slice in its own package
    listed in shared{} config → singleton instance across all MFEs

STRATEGY 3 — Backend-as-Source-of-Truth (production grade)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  All cart mutations persist to a Cart Service REST API / GraphQL.
  All MFEs read from the same API or subscribe via WebSocket.
  Removes all need for cross-MFE runtime communication.
```

#### 4.2c — Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Cart state location | Local to Cart MFE | Keeps domain isolation; Listing communicates via events |
| Cross-MFE communication | `window` custom events | Zero coupling — no shared module needed between Listing and Cart |
| Cart persistence | `localStorage` (dev) → Cart API (prod) | Survives page refresh; production delegates to Cart microservice |
| Checkout handoff | URL navigation (`/checkout`) | Decoupled — Checkout MFE reads cart state independently |

---

### 4.3 Checkout — Order Checkout Micro-Frontend

> **Role:** Guides the user through the final purchase flow — order review, shipping address, payment capture (or stub), and order confirmation.  
> **Pattern:** Remote MFE. Terminal flow — user enters on `/checkout` and exits to an order confirmation or back to `/listing`. No other MFE navigates the user away from checkout.  
> **Key Features:** webpack Module Federation remote · Multi-step form · Cart data consumer · Tailwind CSS.

#### 4.3a — Expose Configuration

```js
// checkout/webpack.config.js
new ModuleFederationPlugin({
  name: "checkout",
  filename: "remoteEntry.js",
  exposes: {
    "./App": "./src/App",
  },
  shared: {
    react:              { singleton: true },
    "react-dom":        { singleton: true },
    "react-router-dom": { singleton: true },
  }
})
```

#### 4.3b — Checkout Flow

```
User arrives at /checkout
        │
        ▼
CheckoutApp reads cart (localStorage / Cart API / event bus)
        │
        ▼
Step 1: Order Review ──────── display items + totals
        │
        ▼
Step 2: Shipping Address ─── address form + validation
        │
        ▼
Step 3: Payment ──────────── payment stub / form
        │
        ▼
Step 4: Confirmation ─────── order ID displayed
                             navigate to /listing (container router)
```

#### 4.3c — Form Validation Strategy

| Field | Validation | Error Presentation |
|---|---|---|
| Email | HTML5 `type="email"` + regex | Inline below field |
| Address line 1 | Required, min 5 chars | Inline below field |
| Postal code | Format regex per locale | Inline below field |
| Card number (stub) | Luhn algorithm or length check | Inline below field |
| Submit | All fields valid → enable button | Disabled state on CTA |

**Principal rule:** Never submit a checkout form until all fields pass client-side validation. Inline error messages must appear on `onBlur` (not on `onChange` — typing halfway through a field should not show errors immediately).

---

### 4.4 Shared Module Design

#### 4.4a — Shared Dependency Classification

```
SHARED SINGLETONS (must be the same instance across all MFEs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  react              → Hooks + Context require a single React root
  react-dom          → ReactDOM.render / createRoot must be called once
  react-router-dom   → One BrowserRouter = one history object

SHARED WITH VERSION NEGOTIATION (duplicates allowed but wasteful)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  tailwindcss        → CSS utility classes; no singleton requirement
  axios / fetch      → HTTP client; stateless; can duplicate safely

NOT SHARED / EACH MFE OWNS ITS OWN COPY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Internal components     → ProductCard, CartItem, CheckoutForm
  Internal hooks          → useProductFilter, useCartTotal
  Domain-specific utils   → price formatting, address validation
  Internal state stores   → cart slice, checkout form state
```

#### 4.4b — Shared Singleton Version Conflict Rules

| Situation | Outcome | Resolution |
|---|---|---|
| Container: React 18.2.0, Remote: React 18.2.0 | ✅ Single instance shared | No action needed |
| Container: React 18.2.0, Remote: React 18.3.0 | ⚠️ Version mismatch — Module Federation picks one | Pin all to same minor version in all `package.json` files |
| Container has `singleton: true`, Remote omits `singleton` | ❌ Remote loads its own React | Add `singleton: true` to ALL remotes |
| Two remotes with conflicting required ranges | ❌ Module Federation throws at runtime | Use `requiredVersion: "^18.0.0"` consistently |

**Principal rule:** Treat shared singleton versions as a contract enforced at the monorepo or dependency-management level. One team updating their React version must coordinate with all other teams. This is the primary dependency governance challenge in micro-frontend systems.

---

### 4.5 Infrastructure and Deployment

> **Role:** Runtime platform — provides the hosting, build, dev, and deployment pipeline for four independently deployable applications.  
> **Pattern:** Five progressive tiers: Local Dev (individual `npm start`) → Dev Server (webpack-dev-server per MFE) → CI/CD (independent pipeline per MFE) → CDN (static assets per MFE) → Container/K8s (optional).

#### 4.5a — Port Allocation (Local Development)

| Application | Port | Entry URL | remoteEntry URL |
|---|---|---|---|
| `container` | 3000 | `http://localhost:3000` | N/A (host) |
| `listing` | 3001 | `http://localhost:3001` | `http://localhost:3001/remoteEntry.js` |
| `cart` | 3002 | `http://localhost:3002` | `http://localhost:3002/remoteEntry.js` |
| `checkout` | 3003 | `http://localhost:3003` | `http://localhost:3003/remoteEntry.js` |

#### 4.5b — Independent Deployment Flow

```
Team A merges to main in /listing
    │
    ▼
CI pipeline: install → test → build (webpack production)
    │
    ▼
Outputs: dist/remoteEntry.js  +  dist/[contenthash].js
    │
    ▼
Deploy dist/ to CDN (S3 + CloudFront / Azure Blob + CDN / GCS + Cloud CDN)
    │
    ▼
remoteEntry.js URL is now live at:
  https://listing.cdn.example.com/remoteEntry.js
    │
    ▼
Container DOES NOT redeploy — it fetches the new remoteEntry.js 
at the NEXT page load automatically
    │
    ▼
✅  Listing updated in production without touching Container, Cart, or Checkout
```

**This is the core value proposition of Micro-Frontend Architecture:** Each team deploys independently. No cross-team release coordination required for routine feature releases.

#### 4.5c — webpack-dev-server Configuration (All MFEs)

```js
// webpack.config.js (shared pattern)
devServer: {
  port: <3000 | 3001 | 3002 | 3003>,
  historyApiFallback: true,  // React Router needs this — returns index.html for all 404s
  headers: {
    "Access-Control-Allow-Origin": "*",  // required: host fetches remoteEntry.js cross-origin
  },
},
output: {
  publicPath: "auto",  // resolves asset URLs relative to where remoteEntry.js was served from
},
```

**Why `publicPath: "auto"`:** In codespace and cloud environments, the URL where the remote is served is not known at build time. `"auto"` tells webpack to derive the public URL at runtime from wherever `remoteEntry.js` was fetched — this is what enables the GitHub Codespaces URLs to work without hardcoding.

#### 4.5d — Multi-Cloud CDN Deployment Options

| Cloud Provider | Static Hosting | CDN | Notes |
|---|---|---|---|
| AWS | S3 Bucket | CloudFront | Set `Cache-Control: no-cache` on `remoteEntry.js` only; long-cache chunked JS |
| Azure | Azure Blob Storage | Azure CDN / Front Door | Same `remoteEntry.js` cache invalidation strategy |
| GCP | Google Cloud Storage | Cloud CDN | Cache rules via `gcs` bucket metadata |
| Vercel | Vercel deployment per MFE | Vercel Edge Network | Zero-config, per-app deployments |
| Netlify | Netlify per-site | Netlify CDN | `netlify.toml` per MFE |

**Critical CDN Rule:** `remoteEntry.js` must have `Cache-Control: no-cache` or a very short TTL (≤ 60s). It is the manifest that points to the versioned chunk filenames. If it is cached for 24 hours, users will not receive new deployments for up to 24 hours. Versioned chunk files (e.g., `listing.8f3a1b.js`) can be cached forever (`max-age=31536000, immutable`).

#### 4.5e — Build Pipeline per MFE (GitHub Actions Pattern)

```yaml
# .github/workflows/listing-deploy.yml
name: Deploy Listing MFE
on:
  push:
    paths: ["listing/**"]   # only trigger when listing changes
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
        working-directory: listing
      - run: npm test -- --watchAll=false
        working-directory: listing
      - run: npm run build
        working-directory: listing
      # Deploy listing/dist/ to CDN
      - name: Deploy to S3
        run: |
          aws s3 sync listing/dist s3://my-listing-bucket \
            --cache-control "max-age=31536000,immutable"
          aws s3 cp listing/dist/remoteEntry.js s3://my-listing-bucket/remoteEntry.js \
            --cache-control "no-cache"   # ← remoteEntry must be fresh
```

---

### 4.6 Testing Strategy

> **Role:** Multi-level quality gate — MFE isolation unit tests, integration tests per remote in standalone mode, cross-MFE contract tests, and E2E tests through the container.  
> **Pattern:** Vitest / Jest per MFE → React Testing Library → Mock Module Federation → Playwright / Cypress for cross-MFE E2E.

#### 4.6a — Test Layer Overview

| Layer | Scope | Tool | Speed | When to Run |
|---|---|---|---|---|
| Unit | Single component in isolation | Jest + React Testing Library | < 30s | Every commit |
| Integration | MFE running standalone | Jest (JSDOM) + RTL | < 2 min | Every PR |
| Contract | Does the remote expose what the host expects? | Module Federation contract test | < 1 min | Every PR |
| E2E (smoke) | Full user journey through Container | Playwright / Cypress | 2–5 min | Every PR (smoke) |
| E2E (full) | All user journeys including edge cases | Playwright / Cypress | 10–20 min | Nightly |

#### 4.6b — Unit Testing Pattern (React Testing Library)

```jsx
// listing/src/components/__tests__/ProductCard.test.jsx
import { render, screen, fireEvent } from "@testing-library/react";
import ProductCard from "../ProductCard";

const mockProduct = {
  id: 1,
  name: "Wireless Headphones",
  price: 79.99,
  image: "/headphones.jpg",
};

describe("ProductCard", () => {
  it("renders product name and price", () => {
    render(<ProductCard product={mockProduct} onAddToCart={() => {}} />);

    expect(screen.getByText("Wireless Headphones")).toBeInTheDocument();
    expect(screen.getByText("$79.99")).toBeInTheDocument();
  });

  it("calls onAddToCart with product when button clicked", () => {
    const handleAdd = jest.fn();
    render(<ProductCard product={mockProduct} onAddToCart={handleAdd} />);

    fireEvent.click(screen.getByRole("button", { name: /add to cart/i }));

    expect(handleAdd).toHaveBeenCalledWith(mockProduct);
    expect(handleAdd).toHaveBeenCalledTimes(1);
  });
});
```

#### 4.6c — Mocking Remote Modules in Container Tests

```js
// container/jest.config.js
module.exports = {
  moduleNameMapper: {
    // Intercept Module Federation remote imports and substitute test doubles
    "^listing/App$":   "<rootDir>/__mocks__/listingApp.jsx",
    "^cart/App$":      "<rootDir>/__mocks__/cartApp.jsx",
    "^checkout/App$":  "<rootDir>/__mocks__/checkoutApp.jsx",
  },
};

// container/__mocks__/listingApp.jsx
const ListingApp = () => <div data-testid="listing-mock">Listing MFE</div>;
export default ListingApp;

// container/src/__tests__/App.test.jsx
import { render, screen } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";
import App from "../App";

it("renders listing mock on /listing route", () => {
  render(
    <MemoryRouter initialEntries={["/listing"]}>
      <App />
    </MemoryRouter>
  );
  expect(screen.getByTestId("listing-mock")).toBeInTheDocument();
});
```

#### 4.6d — Cross-MFE Contract Testing

```
CONTRACT TEST CONCEPT:
The container declares it will import "listing/App" and render it.
The listing remote declares it exposes "./App".
A contract test verifies this agreement holds on every build.

  Container test:           Remote test:
  ─────────────────         ──────────────────
  Can I import              Does my remoteEntry.js 
  "listing/App"?            expose "./App"?
       │                           │
       ▼                           ▼
  import("listing/App")     Inspect built remoteEntry.js
  resolves without error    for exposes map entry

Tooling: @module-federation/contract (or custom test reading remoteEntry manifest)
```

#### 4.6e — E2E Test Pattern (Playwright)

```ts
// e2e/shopping-journey.spec.ts
import { test, expect } from "@playwright/test";

test("full shopping journey: browse → add to cart → checkout", async ({ page }) => {
  // Step 1: Land on Listing
  await page.goto("http://localhost:3000/listing");
  await expect(page.getByTestId("product-grid")).toBeVisible();

  // Step 2: Add first product to cart
  const firstAddButton = page.getByRole("button", { name: /add to cart/i }).first();
  await firstAddButton.click();
  await expect(page.getByTestId("cart-count")).toHaveText("1");

  // Step 3: Navigate to checkout
  await page.goto("http://localhost:3000/checkout");
  await expect(page.getByTestId("order-summary")).toBeVisible();
  await expect(page.getByTestId("cart-items")).toContainText("1 item");

  // Step 4: Complete checkout form
  await page.fill('[name="email"]', "test@example.com");
  await page.fill('[name="address"]', "123 Main Street");
  await page.fill('[name="city"]', "New York");
  await page.fill('[name="postal"]', "10001");
  await page.click('[data-testid="submit-order"]');

  // Step 5: Confirm order success
  await expect(page.getByTestId("order-confirmation")).toBeVisible();
});
```

#### 4.6f — Test Data Management

| Strategy | Use Case | Implementation |
|---|---|---|
| Mock data objects | Unit tests | `const mockProduct = { id: 1, name: "...", price: 9.99 }` |
| MSW (Mock Service Worker) | Integration — stub APIs | `rest.get("/api/products", (req, res, ctx) => res(ctx.json(mockData)))` |
| Playwright fixtures | E2E — consistent product catalogue | `page.route("**/api/products", route => route.fulfill({ json: seedData }))` |
| Shared test seed file | All layers | `src/__fixtures__/products.ts` committed alongside tests |

#### 4.6g — Principal Testing Best Practices

1. **Test the component as the user sees it, not its internals.** Use `getByRole`, `getByText`, `getByLabelText` — not `getByTestId` or component class names. If the markup changes but the UX is identical, the test should still pass.

2. **Do not test Module Federation wiring with Jest.** Jest cannot execute real `remoteEntry.js` files. Use Playwright for cross-MFE integration. Jest tests should mock all remote imports (§4.6c).

3. **Each MFE's CI must include its standalone E2E.** The Listing team should be able to verify their MFE works end-to-end without starting the container. A `docker compose` with a mock container is acceptable.

4. **`remoteEntry.js` exposure is a breaking API.** If `listing` renames `./App` to `./ListingApp` in its exposes config, the container breaks silently at runtime. Contract tests (§4.6d) are the only automated guard against this.

5. **Test the Suspense fallback.** Verify the container shows a loading state while the remote loads — test this by delaying the dynamic import with a mock that returns a delayed promise.

6. **Smoke test on every PR, full E2E nightly.** Tag critical journeys `@smoke` and run them on every PR (< 5 minutes). Run the full suite — including cross-MFE navigation flows — in a nightly scheduled CI run.

---

## 5. Cross-Cutting Concerns

### 5.1 Styling Strategy

```
TAILWIND CSS — Utility-first CSS (all MFEs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Each MFE configures its own tailwind.config.js.
The container provides the baseline design tokens.

Risk: class name collisions are impossible with Tailwind
  (classes are global but deterministic — "text-red-500" means
  the same thing everywhere).

Risk: design inconsistency across teams.

Solution: Shared Tailwind config package (@myorg/tailwind-config)
  published to npm — all MFEs extend it:

  // tailwind.config.js in every MFE
  const sharedConfig = require("@myorg/tailwind-config");
  module.exports = { ...sharedConfig, content: ["./src/**/*.{js,jsx}"] };
```

### 5.2 Error Boundary Strategy

```jsx
// container/src/components/RemoteErrorBoundary.jsx
class RemoteErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error) {
    // Log to monitoring (Datadog, Sentry, Azure Application Insights)
    console.error("Remote MFE failed to load:", error);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert" className="p-8 text-center">
          <h2>This section is temporarily unavailable.</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage in container routing:
<RemoteErrorBoundary>
  <Suspense fallback={<LoadingSpinner />}>
    <ListingApp />
  </Suspense>
</RemoteErrorBoundary>
```

**Principal Rule:** Every remote import must be wrapped in an error boundary. If `listing` fails to load (CDN outage, network error, JS syntax error in remoteEntry), the error boundary prevents the entire container from unmounting. The `cart` and `checkout` continue to function. This is resilience through isolation — one remote outage must never take down the whole application.

### 5.3 Performance — Bundle Splitting Philosophy

```
WITHOUT Module Federation:
  One 2.5 MB bundle → user downloads everything on first load
  React: 130 KB · react-dom: 1 MB · all domain code: 1.4 MB

WITH Module Federation + shared singletons:
  Container initial load:  ~80 KB (shell + router only)
  React (shared):         ~130 KB (loaded once, shared across all remotes)
  react-dom (shared):     ~1 MB  (loaded once, shared across all remotes)
  Listing remote:         ~200 KB (loaded on /listing navigation)
  Cart remote:            ~150 KB (loaded on /cart navigation)
  Checkout remote:        ~180 KB (loaded on /checkout navigation)

  Initial page load saves 330 KB (listing + cart + checkout not loaded upfront)
  React/react-dom NOT duplicated across remotes → saves ~3.3 MB vs naive duplication
```

### 5.4 Principal Recommendation Matrix

| Concern | Recommended Approach | Rationale |
|---|---|---|
| Cross-MFE state (simple) | `window` Custom Events | Zero coupling; no shared module; works for low-frequency events |
| Cross-MFE state (complex) | Shared Zustand/Redux store via `shared{}` singleton | Type-safe; subscribable; devtools support |
| Cross-MFE state (production) | Backend microservice as source of truth | Eliminates all client-side sync issues; works for server-rendered MFEs |
| Routing | Host-only BrowserRouter; remotes use sub-Routes | Only one history instance; no router conflicts |
| Styling | Shared Tailwind config package | Design system consistency; zero CSS specificity wars |
| Shared deps | `singleton: true` for React/React-DOM/Router | Prevents dual-instance hook errors |
| Remote load failure | Error Boundary per remote slot | Isolated failure; rest of app works |
| CDN caching | `no-cache` on remoteEntry.js, `immutable` on chunks | Fresh manifests; optimal chunk caching |
| CI/CD | Independent pipeline per MFE (path filter trigger) | True independent deployability |
| Testing remotes in host | Jest `moduleNameMapper` mocks | Fast; no real webpack federation in unit tests |
| E2E cross-MFE tests | Playwright with real devServer | Tests real Module Federation wiring |

---

## 6. Architecture Decision Records (ADRs)

### ADR-001: Webpack Module Federation over import maps or iframes

**Decision:** Use Webpack Module Federation for runtime integration.

**Considered:**
- `<iframe>` embedding — full isolation but no shared React, no CSS bleed protection via scope, no unified routing
- Import maps — native browser feature but no shared singleton negotiation, no fallback loading, limited tooling
- Webpack Module Federation — mature, battle-tested, React-first, handles shared singleton negotiation automatically

**Rationale:** Module Federation provides the best balance of team independence (independent deployments), runtime integration (shared React instance), and developer experience (standard webpack workflow).

---

### ADR-002: Single React Router instance in Container only

**Decision:** Only the `container` defines `<BrowserRouter>`. All remotes are React sub-trees inside the host's router.

**Consequence:** Remotes cannot define their own top-level navigation. They use `<Link>`, `<Routes>`, `<Route>` — but they consume the router context provided by the container.

**Rationale:** Two `<BrowserRouter>` instances create two independent history stacks. Navigation in one does not update the other. URL bar state becomes inconsistent. One router is the only correct pattern for a multi-page SPA.

---

### ADR-003: `singleton: true` on all shared React packages

**Decision:** All four applications (`container`, `listing`, `cart`, `checkout`) declare `react`, `react-dom`, and `react-router-dom` as `{ singleton: true }` in their shared configuration.

**Consequence:** All remotes must be compatible with the same React major version. A breaking React upgrade must be coordinated across all teams simultaneously.

**Rationale:** Non-singleton React causes `Invalid hook call` at runtime — a hard failure with a confusing error message. The coordination cost of version alignment is lower than the operational cost of runtime hook failures in production.

---

*Generated 2026-03-06 · Principal architect analysis of `Micro-Frontend Architecture with React` (React 18 · Webpack Module Federation · Tailwind CSS · React Router DOM · JavaScript · GitHub Codespaces)*
