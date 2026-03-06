# Micro-Frontend Architecture with React — Architecture Reference

> **Module:** E-Commerce Platform · Webpack Module Federation · React 18 · Tailwind CSS  
> **Principal architect view:** MFE topology, Module Federation wiring, shared dependency strategy, routing, state, infrastructure, and testing.

---

## 1. Overall System Architecture

Five independent applications collaborate through Webpack Module Federation. The `container` is the **host** — it owns the shell, global navigation, and routing. The three **remotes** (`listing`, `cart`, `checkout`) are deployed independently and expose their components at runtime.

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
