# SPIKE: Bundle Re-engineering — Code Splitting & Dynamic Loading

**Project:** Enterprise Platform  
**Stack:** React 19 + Vite 7 (Rollup) + Redux Toolkit + React Router DOM v7  
**Date:** February 2026  
**Author:** Lead Engineer 
**Version:** 1.0

---

## Table of Contents

1. [Executive Summary for Clients](#1-executive-summary-for-clients)
2. [Research Methodological Framework](#2-research-methodological-framework)
3. [Current State Diagnosis](#3-current-state-diagnosis)
4. [Visual Analysis Tool (Bundle Analyzer)](#4-visual-analysis-tool-bundle-analyzer)
5. [Step-by-Step Implementation Guide](#5-step-by-step-implementation-guide)
6. [Results Report Template (Before/After)](#6-results-report-template-beforeafter)

---

## 1. Executive Summary for Clients

### Problem Detected

The Enterprise Platform application loads **its entire source code** on the user's first visit, regardless of the requested page. This means a user who only needs to sign in also downloads the code for reports, document generation, charts, entity management, and the other 50+ application screens.

### Measurable Impact

| Metric | Current State (Estimated) | Post-Optimization Target |
|---------|--------------------------|----------------------------|
| **Main bundle** | >2 MB (uncompressed) | <400 KB initial load |
| **First Contentful Paint** | >3.5s (3G) | <1.8s (3G) |
| **Time to Interactive** | >5.0s (3G) | <3.0s (3G) |
| **Lighthouse Performance** | 40–60 | 80–95 |

### Expected ROI

- **-60% to -75%** in initial load size (main bundle).
- **+30 to +50 points** in Lighthouse Performance Score.
- **Direct SEO improvement:** Google penalizes sites with FCP > 2.5s and TTI > 3.8s.
- **Bounce rate reduction:** Every 100ms improvement in load time increases conversion by ~1% (source: Deloitte, "Milliseconds Make Millions", 2020).

### Risk of Not Acting

A growing monolithic bundle implies progressive degradation. Each new feature increases load time for **all** pages, including the login screen.

---

## 2. Research Methodological Framework

### 2.1 Research Type

**Quasi-Experimental Research with Pre-Test / Post-Test Single Group Design.**

| Aspect | Description |
|---------|-------------|
| **Type** | Quasi-Experimental (Applied Research) |
| **Design** | Pre-Test / Post-Test — Single Group |
| **Independent Variables** | Code splitting techniques, lazy loading, vendor splitting, tree shaking |
| **Dependent Variables** | Bundle size (KB), FCP (ms), TTI (ms), Lighthouse Score |
| **Control Group** | Baseline measurements of current state (pre-intervention) |
| **Experimental Group** | Same application post-optimization |
| **Internal Validity** | Environment controlled (same hardware, same simulated network with Lighthouse throttling) |
| **External Validity** | Results generalizable to SPAs with similar architecture |

### 2.2 Problem Statement

> *The Enterprise Platform web application presents a monolithic bundle that synchronously loads all 55 scenes, 30+ shared components, and all third-party dependencies (~15 libraries) on initial load, resulting in First Contentful Paint (FCP) and Time to Interactive (TTI) times that exceed Google Web Vitals recommended thresholds (FCP < 1.8s, TTI < 3.8s), negatively impacting SEO ranking, user experience, and business conversion metrics.*

### 2.3 Hypotheses

**H₁:** The implementation of route-level code splitting, vendor splitting, and tree shaking optimization will reduce initial bundle size by at least 60% and improve FCP by at least 40%.

**H₀ (Null):** Bundle optimization techniques do not produce a statistically significant improvement in performance metrics.

### 2.4 Technical Justification (for Stakeholders)

1. **Core Web Vitals as ranking factor (Google, 2021–present):** LCP, FID/INP, and CLS are ranking signals. Excessive bundle directly degrades LCP and FID.
2. **Cost of JavaScript (Addy Osmani, Google Chrome Team, 2023):** Every KB of JavaScript has a parsing + compilation + execution cost. JS is byte-by-byte more expensive than an image of the same size.
3. **HTTP Archive Web Almanac 2023:** Median JS transferred on mobile pages is ~500KB. Exceeding this threshold places the application in the lower performance percentile.
4. **Deloitte, "Milliseconds Make Millions" (2020):** 100ms improvements in load speed generate measurable increases in engagement and conversion.

### 2.5 Measurement Methodology

```
┌─────────────────────────────────────────────────────────────┐
│                    MEASUREMENT PROTOCOL                     │
├─────────────────────────────────────────────────────────────┤
│ 1. Production build: `npm run build`                       │
│ 2. Size registration: `dist/assets/*.js` (raw + gzip)      │
│ 3. Lighthouse CI: 5 runs, median, mobile mode               │
│    - Throttling: Simulated Slow 4G                          │
│    - CPU: 4x slowdown                                       │
│ 4. Bundle Analyzer: visual treemap pre and post             │
│ 5. Tabulated comparison with percentage deltas              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Current State Diagnosis

### 3.1 Critical Anti-Patterns Detected

#### 🔴 AP-1: Zero Code Splitting — Monolithic Bundle

**File:** `@/router/index.tsx`

All **55 views** are statically imported through the barrel export `src/views/index.jsx`:

```javascript
// @/router/index.tsx (lines 1-58)
import {
  AccountSettings, OrganizationSetup, AccessControl, TeamManagement,
  Partners, PartnerDirectory, StakeholderRegistry, IdentifierManager, EntityCreation,
  ClientOnboarding, ApprovalWorkflows, GlobalContractTerms,
  ProfileEditor, ProviderRegistry, RoleDefinitions, UserDirectory, CustomerIdentifiers,
  AnalyticsDashboard, AdminAuditLog, BillingTermsEditor, WorkflowReview,
  ClientGroups, VendorDirectory, DocumentDetailView, BillingDashboard, MainLayout,
  EntityRegistry, AssetManagement, SystemLogs, DisputeResolution, Documentation,
  EmptyState, ActivityTracker, ComplianceHistory, ContractLibrary,
  ServiceProviders, SecurityRecovery, PaymentMethodRegistry, UserGraph, BusinessAnalytics,
  DataRequest, PermissionsManager, AuthPortal, SuccessFeedback, ComplianceTerms, UserProfileView, ValidationPortal,
  DocumentBundles, TransactionRetry, SSOGateway, ProfileSelection, PolicyDirectory,
  ConsentAgreement, OperationsManual,
} from "../views";
```

**Consequence:** A user visiting `/login` downloads code for all 55 screens.

#### 🔴 AP-2: Namespace Imports that Destroy Tree Shaking

**File:** `src/util/index.js`

```javascript
import * as FaIcons from "react-icons/fa";   // ~1,500 FA icons included
import * as TbIcons from "react-icons/tb";   // ~4,500 Tabler icons included
import * as BiIcons from "react-icons/bi";   // ~800 BoxIcons included
```

**Consequence:** `import *` imports **ALL** icons from each package. If only 20 icons are used, ~6,800 unnecessary icons are being loaded. Estimated impact: **+500KB to +1MB** of dead JavaScript.

**Additional files with `import *`:**
- `src/index.jsx` → `import * as Sentry from "@sentry/react"`
- `src/App.jsx` → `import * as Sentry from "@sentry/react"`
- `src/components/layout/SideNavigation/components/Menu/index.jsx` → react-icons
- `src/views/layout/components/Dropdown/index.jsx` → react-icons
- `src/components/ToggleButton/index.jsx` → Sentry
- `src/components/ErrorElement/index.jsx` → Sentry

#### 🟡 AP-3: Heavy Dependencies Without Lazy Loading

| Dependency | Estimated Size (min) | Used in | Usage Frequency |
|-------------|----------------------|----------|-------------------|
| `recharts` | ~500 KB | `analytics/components/Charts/` (1 file) | Only AnalyticsDashboard |
| `jspdf` + `jspdf-autotable` | ~300 KB | `documentDetail/components/DocumentPDF/` (1 file) | Only when generating PDF |
| `react-icons` (with `import *`) | ~500-1000 KB | 3 files | Sidebar menu + Utils |
| `@sentry/react` | ~250 KB | Entry point | Always (but namespace import) |
| `react-datepicker` | ~150 KB | Date forms | Some screens |
| `react-select` | ~100 KB | Advanced dropdowns | Some screens |
| `react-table-legacy` | ~80 KB | Legacy tables | Multiple screens |
| `core-js` (4 polyfills) | ~80 KB | `index.jsx` | Unnecessary in React 19 |
| `bulma` | ~200 KB (CSS) | Global | Always |
| `redux-logger` | ~15 KB | Store (dev only) | Included in production |

#### 🟡 AP-4: Barrel Exports Without Lazy Boundaries

**Files:** `src/views/index.jsx` (55 re-exports) and `src/components/index.jsx` (30 re-exports)

Barrel exports consolidate imports but, combined with static imports in `@/router/index.tsx`, force Rollup to include **everything** in a single chunk.

#### 🟡 AP-5: No Vendor Splitting

**File:** `vite.config.mjs` — The `rollupOptions.output` section doesn't configure `manualChunks`, so all `node_modules` dependencies are packaged together with application code.

### 3.2 Impact Diagram

```
                    index.html
                        │
                   ┌────▼────┐
                   │ main.js │ ← MONOLITHIC (~2-3 MB estimated)
                   └────┬────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐    ┌─────▼─────┐   ┌────▼────┐
   │ 55      │    │ 15+       │   │ 30      │
   │ Views   │    │ Vendor    │   │ Shared  │
   │ (ALL)   │    │ Libs      │   │ Comps   │
   └─────────┘    └───────────┘   └─────────┘
        │               │
   Partners        react-icons/*
   jspdf           sentry/*
   react-table     core-js
   react-select    redux-logger
   react-datepicker bulma (CSS)
```

---

## 4. Visual Analysis Tool (Bundle Analyzer)

### 4.1 Recommended Tool: `rollup-plugin-visualizer`

Since Vite uses Rollup internally for production builds, the native tool is **rollup-plugin-visualizer**.

#### Installation

```bash
npm install --save-dev rollup-plugin-visualizer
```

#### Configuration in `vite.config.mjs`

```javascript
import { visualizer } from "rollup-plugin-visualizer";

// Add inside the plugins array:
plugins: [
  // ...existing plugins,
  visualizer({
    filename: "bundle-analysis.html",  // Output file
    open: true,                         // Auto-opens in browser
    gzipSize: true,                     // Shows gzip size
    brotliSize: true,                   // Shows brotli size
    template: "treemap",                // Options: treemap | sunburst | network
  }),
],
```

#### Execution

```bash
npm run build
# bundle-analysis.html generated at project root
```

### 4.2 How to Interpret the Treemap

```
┌─────────────────────────────────────────────────────────────────┐
│                        READING GUIDE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rectangle Size = Module size in bundle                        │
│                                                                 │
│  ┌──────────────────────┬────────────┬──────┐                  │
│  │                      │            │      │                  │
│  │   node_modules/      │  recharts  │ jspdf│                  │
│  │   react-icons/       │  (~500KB)  │      │                  │
│  │   (~800KB)           │            │      │                  │
│  │                      ├────────────┤      │                  │
│  │   ← QUICK WIN #1    │  sentry    │      │                  │
│  │                      │  (~250KB)  │      │                  │
│  ├──────────────────────┼────────────┴──────┤                  │
│  │   src/scenes/        │  core-js          │                  │
│  │   (all app code)     │  (polyfills)      │                  │
│  │                      │  ← QUICK WIN #3   │                  │
│  └──────────────────────┴───────────────────┘                  │
│                                                                 │
│  LOOK FOR:                                                     │
│  1. LARGE rectangles in node_modules → split candidates       │
│  2. Modules appearing DUPLICATED → deduplication              │
│  3. All app code together → missing code splitting             │
│  4. Libraries used in 1 route → dynamic import candidates    │
│                                                                 │
│  QUICK WINS (by impact order):                                │
│  #1: react-icons import * → named imports (−500KB to −1MB)    │
│  #2: recharts + jspdf → dynamic import (−800KB initial)       │
│  #3: core-js polyfills → remove (React 19 doesn't need them)  │
│  #4: 55 scenes → lazy loading by route (−60% initial bundle)  │
│  #5: vendor splitting → granular third-party caching           │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Add to `.gitignore`

```
bundle-analysis.html
```

---

## 5. Step-by-Step Implementation Guide

### Phase 1: Quick Wins — Fix Tree Shaking (Immediate Impact)

#### Step 1.1: Remove `import *` from react-icons — Icon Registry Pattern

##### Problem Context

The `customIcon(name)` function in `src/util/index.js` is a **dynamic resolver** that receives a string (e.g., `"FaChartLine"`) and searches for the component in namespace imports. It's used **182 times across 48 files**. The same pattern is replicated in `Menu/index.jsx` (with `Icons` and `IconsGo`).

Icon names come as strings from `@/config/navigationOptions.ts` and hardcoded JSX in scenes (e.g., `customIcon("FaRegEdit")`).

**Why can't we just change to named imports?** Because the `customIcon` function does dynamic string lookup: `FaIcons[name]`. We need to maintain the string→component map, but **only with actually used icons**.

##### Solution: Icon Registry (Explicit Map)

**Step A:** Audit all icon names used in the project:

```bash
# Extract all icon names passed to customIcon
grep -roh "customIcon(['\"][A-Za-z]*['\"])" src/ | sort -u
# Extract icon props in navigationOptions.ts
grep -oh 'icon: "[A-Za-z]*"' @/config/navigationOptions.ts | sort -u
```

**Icons identified in project audit:**

```
# From navigationOptions.ts (Menu icons):
FaAngleDown, FaBook, FaChartLine, FaClipboardList, FaFileAlt,
FaFileInvoiceDollar, FaRegCreditCard, FaRegIdBadge, FaRegListAlt,
FaSyncAlt, FaThList, FaUniversity, FaUserFriends, FaUserPlus,
FaUsers, FaUserTag, GoLaw

# From customIcon() calls in scenes/components:
FaBan, FaCheck, FaCheckCircle, FaDownload, FaEdit,
FaExclamationCircle, FaEye, FaFileInvoice, FaHandHoldingUsd,
FaMinusCircle, FaPlus, FaPlusCircle, FaRegCheckCircle, FaRegEdit,
FaRegEye, FaRegFileAlt, FaStore, FaStoreAltSlash, FaTimes,
FaUsersSlash

# Total: ~37 icons (vs ~6,800 with import *)
# NOTE: Run audit with grep before implementing
# to capture any additional icons.
```

**Step B:** Create `src/util/iconRegistry.js`:

```javascript
// ── Icon Registry ──
// We only import icons actually used in the application.
// To add a new icon: 1) import it here, 2) add it to the map.
// NEVER use import * from react-icons.

import {
  FaAngleDown,
  FaBan,
  FaBook,
  FaChartLine,
  FaCheck,
  FaCheckCircle,
  FaClipboardList,
  FaDownload,
  FaEdit,
  FaExclamationCircle,
  FaEye,
  FaFileAlt,
  FaFileInvoice,
  FaFileInvoiceDollar,
  FaHandHoldingUsd,
  FaMinusCircle,
  FaPlus,
  FaPlusCircle,
  FaRegCheckCircle,
  FaRegCreditCard,
  FaRegEdit,
  FaRegEye,
  FaRegFileAlt,
  FaRegIdBadge,
  FaRegListAlt,
  FaStore,
  FaStoreAltSlash,
  FaSyncAlt,
  FaTimes,
  FaThList,
  FaUniversity,
  FaUserFriends,
  FaUserPlus,
  FaUsers,
  FaUsersSlash,
  FaUserTag,
} from "react-icons/fa";

import { GoLaw } from "react-icons/go";

// Add Tabler and BoxIcons icons that are actually used here:
// import { TbXxx } from "react-icons/tb";
// import { BiXxx } from "react-icons/bi";

const ICON_MAP = {
  // FontAwesome
  FaAngleDown,
  FaBan,
  FaBook,
  FaChartLine,
  FaCheck,
  FaCheckCircle,
  FaClipboardList,
  FaDownload,
  FaEdit,
  FaExclamationCircle,
  FaEye,
  FaFileAlt,
  FaFileInvoice,
  FaFileInvoiceDollar,
  FaHandHoldingUsd,
  FaMinusCircle,
  FaPlus,
  FaPlusCircle,
  FaRegCheckCircle,
  FaRegCreditCard,
  FaRegEdit,
  FaRegEye,
  FaRegFileAlt,
  FaRegIdBadge,
  FaRegListAlt,
  FaStore,
  FaStoreAltSlash,
  FaSyncAlt,
  FaTimes,
  FaThList,
  FaUniversity,
  FaUserFriends,
  FaUserPlus,
  FaUsers,
  FaUsersSlash,
  FaUserTag,
  // GitHub Octicons
  GoLaw,
  // Tabler Icons (add per audit)
  // BoxIcons (add per audit)
};

export default ICON_MAP;
```

**Step C:** Refactor `customIcon` in `src/util/index.js`:

```javascript
// BEFORE:
import * as FaIcons from "react-icons/fa";
import * as TbIcons from "react-icons/tb";
import * as BiIcons from "react-icons/bi";

export const customIcon = (name) => {
  const Icon = FaIcons[name] || TbIcons[name] || BiIcons[name];
  return Icon ? <Icon /> : "";
};

// AFTER:
import ICON_MAP from "./iconRegistry";

export const customIcon = (name) => {
  const Icon = ICON_MAP[name];
  if (!Icon) {
    if (process.env.NODE_ENV !== "production") {
      console.warn(`[customIcon] Icon "${name}" not found in registry. Add it to src/util/iconRegistry.js`);
    }
    return "";
  }
  return <Icon />;
};
```

**Step D:** Refactor `Menu/index.jsx`:

```javascript
// BEFORE:
import * as Icons from "react-icons/fa";
import * as IconsGo from "react-icons/go";

customIcon = ({ name }) => {
  const FaIcon = Icons[name];
  const GoIcon = IconsGo[name];
  // ...
};

// AFTER:
import ICON_MAP from "@/util/iconRegistry";

customIcon = ({ name }) => {
  const Icon = ICON_MAP[name];
  if (!Icon) return "";
  return (
    <span className="icon" style={{ marginRight: "5px" }}>
      <Icon />
    </span>
  );
};
```

##### Benefit

From ~6,800 imported icons (FA: ~1,500 + Tabler: ~4,500 + BoxIcons: ~800) to **~30-40 explicit icons**.

**Estimated impact: −500 KB to −1 MB.**

#### Step 1.2: Remove unnecessary core-js polyfills

**Before** (`src/index.jsx`):
```javascript
import "core-js/features/object";
import "core-js/features/array";
import "core-js/features/string";
import "core-js/features/promise";
```

**After:** Delete these lines. React 19 requires modern browsers that already support these APIs natively. Additionally, the project's `browserslist` excludes IE11 and Opera Mini.

**Estimated impact: −60 KB to −80 KB.**

#### Step 1.3: Exclude redux-logger from production

**Before** (`src/store/store.js`):
```javascript
import { logger } from "redux-logger";
```

**After** — conditional dynamic import:
```javascript
// Remove static import from above and use dynamic import:
const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) => {
    const middleware = getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    });

    return middleware;
  },
  devTools: process.env.NODE_ENV === "production" ? false : { trace: true },
});

// Logger only in development, loaded asynchronously
if (process.env.NODE_ENV !== "production") {
  import("redux-logger").then(({ createLogger }) => {
    const logger = createLogger();
    store.dispatch = ((next) => (action) => {
      // This approach is not ideal with RTK; alternative:
      // use Redux DevTools Extension which is already enabled
    })(store.dispatch);
  });
}
```

> **Simpler alternative:** Since `devTools: { trace: true }` is already enabled, consider **removing `redux-logger` completely** and use Redux DevTools Extension from the browser.

**Estimated impact: −15 KB + dependency removal.**

---

### Phase 2: Route-Level Code Splitting (Major Impact)

#### Step 2.1: Create lazy imports file

Create `@/router/lazyRoutes.tsx`:

```javascript
import { lazy } from "react";

// ── Auth Routes (immediate load — first screen) ──
// SignIn, RecoverPassword, ValidateCode, Saml remain static
// because they are the first screens the user sees.

// ── Private Routes (dynamic load) ──
export const AnalyticsDashboard = lazy(() => import("@/views/analyticsDashboard"));
export const ServiceProviders = lazy(() => import("@/views/serviceProviders"));
export const BillingDashboard = lazy(() => import("@/views/billingDashboard"));
export const DocumentDetailView = lazy(() => import("@/views/documentDetailView"));
export const DocumentBundles = lazy(() => import("@/views/documentBundles"));
export const ProviderRegistry = lazy(() => import("@/views/providerRegistry"));
export const ClientOnboarding = lazy(() => import("@/views/clientOnboarding"));
export const ProfileEditor = lazy(() => import("@/views/profileEditor"));
export const UserGraph = lazy(() => import("@/views/userGraph"));
export const EntityCreation = lazy(() => import("@/views/entityCreation"));
export const Partners = lazy(() => import("@/views/partners"));
export const RoleDefinitions = lazy(() => import("@/views/roleDefinitions"));
export const PermissionsManager = lazy(() => import("@/views/permissionsManager"));
export const AccessControl = lazy(() => import("@/views/accessControl"));
export const BusinessAnalytics = lazy(() => import("@/views/businessAnalytics"));
export const UserProfileView = lazy(() => import("@/views/userProfileView"));
export const ApprovalWorkflows = lazy(() => import("@/views/approvalWorkflows"));
export const AccountSettings = lazy(() => import("@/views/accountSettings"));
export const GlobalContractTerms = lazy(() => import("@/views/globalContractTerms"));
export const WorkflowReview = lazy(() => import("@/views/workflowReview"));
export const SystemLogs = lazy(() => import("@/views/systemLogs"));
export const PartnerDirectory = lazy(() => import("@/views/partnerDirectory"));
export const UserDirectory = lazy(() => import("@/views/userDirectory"));
export const AdminAuditLog = lazy(() => import("@/views/adminAuditLog"));
export const ActivityTracker = lazy(() => import("@/views/activityTracker"));
export const DisputeResolution = lazy(() => import("@/views/disputeResolution"));
export const SecurityRecovery = lazy(() => import("@/views/securityRecovery"));
export const SuccessFeedback = lazy(() => import("@/views/successFeedback"));
export const DataRequest = lazy(() => import("@/views/dataRequest"));
export const AssetManagement = lazy(() => import("@/views/assetManagement"));
export const TeamManagement = lazy(() => import("@/views/teamManagement"));
export const Documentation = lazy(() => import("@/views/documentation"));
export const CustomerIdentifiers = lazy(() => import("@/views/customerIdentifiers"));
export const IdentifierManager = lazy(() => import("@/views/identifierManager"));
export const BillingTermsEditor = lazy(() => import("@/views/billingTermsEditor"));
export const WorkflowReview = lazy(() => import("@/views/workflowReview"));
export const ClientGroups = lazy(() => import("@/views/clientGroups"));
export const VendorDirectory = lazy(() => import("@/views/vendorDirectory"));
export const EntityRegistry = lazy(() => import("@/views/entityRegistry"));
export const ComplianceHistory = lazy(() => import("@/views/complianceHistory"));
export const ContractLibrary = lazy(() => import("@/views/contractLibrary"));
export const TransactionRetry = lazy(() => import("@/views/transactionRetry"));
export const SSOGateway = lazy(() => import("@/views/ssoGateway"));
export const ProfileSelection = lazy(() => import("@/views/profileSelection"));
export const EmptyState = lazy(() => import("@/views/emptyState"));
export const PolicyDirectory = lazy(() => import("@/views/policyDirectory"));
export const ConsentAgreement = lazy(() => import("@/views/consentAgreement"));
export const ComplianceTerms = lazy(() => import("@/views/complianceTerms"));
export const OperationsManual = lazy(() => import("@/views/operationsManual"));
```

#### Step 2.2: Refactor `@/router/index.tsx`

```javascript
import { Suspense } from "react";
import { Navigate } from "react-router-dom";
import { useSelector } from "react-redux";

// ── Static load: only necessary for first render ──
import { AuthPortal, SecurityRecovery, ValidationPortal, SSOGateway } from "../views/auth";
// Or import directly:
// import AuthPortal from "@/views/auth/authPortal";
// import SecurityRecovery from "@/views/auth/securityRecovery";
// import ValidationPortal from "@/views/validationPortal";
// import SSOGateway from "@/views/ssoGateway";

import { ErrorElement, NotFound } from "../components";
import AuthLayout from "@/views/auth/authLayout";
import { Loading } from "@/components";

// ── Dynamic load: all private routes ──
import {
  AnalyticsDashboard, ServiceProviders, BillingDashboard, DocumentDetailView, DocumentBundles,
  ProviderRegistry, ClientOnboarding, ProfileEditor, UserGraph,
  EntityCreation, Partners, RoleDefinitions, PermissionsManager, AccessControl, BusinessAnalytics,
  UserProfileView, ApprovalWorkflows, AccountSettings, GlobalContractTerms,
  WorkflowReview, SystemLogs, PartnerDirectory, UserDirectory,
  AdminAuditLog, ActivityTracker, DisputeResolution, SecurityRecovery, SuccessFeedback, DataRequest,
  AssetManagement, TeamManagement, Documentation, CustomerIdentifiers, IdentifierManager,
  BillingTermsEditor, ClientGroups, VendorDirectory, EntityRegistry, ComplianceHistory,
  ContractLibrary, TransactionRetry, SSOGateway, ProfileSelection, EmptyState, PolicyDirectory,
  ConsentAgreement, ComplianceTerms, OperationsManual,
} from "./lazyRoutes.tsx";

// ── Suspense Wrapper ──
const Lazy = ({ children }) => (
  <Suspense fallback={<Loading />}>
    {children}
  </Suspense>
);

const PrivateRoute = ({ children }) => {
  const auth = useSelector((state) => state.auth);
  if (!auth?.logged) {
    return <Navigate to="/login" replace />;
  }
  return children;
};

const routes = (featureFlags) => [
  {
    path: "/",
    element: <AuthLayout key="/" />,
    errorElement: <ErrorElement />,
    children: [
      { index: true, element: <Navigate to="/login" replace /> },
      { path: "login", element: <AuthPortal key="login" /> },
      { path: "saml", element: <SSOGateway key="saml" /> },
      { path: "recover-password", element: <SecurityRecovery key="recover-password" /> },
      { path: "validate-code", element: <ValidationPortal key="validate-code" /> },
    ],
  },
  {
    path: "/",
    element: (
      <PrivateRoute>
        <Layout key="/" />
      </PrivateRoute>
    ),
    errorElement: <ErrorElement />,
    children: [
      {
        path: "/resp",
        element: <Lazy><SuccessFeedback key="/resp" /></Lazy>,
      },
      {
        path: "dashboard",
        element: <Lazy><AnalyticsDashboard key="analyticsDashboard" /></Lazy>,
      },
      // ... (same pattern for all private routes)
    ],
  },
];
```

> **Result:** Each route becomes a separate chunk. Vite/Rollup automatically generates files like `AnalyticsDashboard.abc123.js`, `BusinessAnalytics.def456.js`, etc.

**Estimated impact: −60% to −75% of initial bundle.**

---

### Phase 3: Dynamic Import for Heavy Dependencies

#### Step 3.1: Lazy load recharts (only AnalyticsDashboard)

**Before** (`src/views/analyticsDashboard/components/Charts/index.jsx`):
```javascript
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip } from "recharts";
```

**After** — route-level `React.lazy` already resolves this. If AnalyticsDashboard is lazy-loaded, recharts is automatically included in the AnalyticsDashboard chunk.

However, if Dashboard has multiple sub-views and Charts are only shown in one of them, add more granular lazy loading:

```javascript
import { lazy, Suspense } from "react";
const Charts = lazy(() => import("./components/Charts"));

// In render:
{showCharts && (
  <Suspense fallback={<div>Loading charts...</div>}>
    <Charts data={data} />
  </Suspense>
)}
```

#### Step 3.2: Lazy load jsPDF (only when generating PDF)

**Before** (`src/views/documentDetailView/components/DocumentPDF/index.jsx`):
```javascript
import jsPDF from "jspdf";
import "jspdf-autotable";
```

**After:**
```javascript
const generatePDF = async (documentData) => {
  const { default: jsPDF } = await import("jspdf");
  await import("jspdf-autotable");

  const doc = new jsPDF();
  // ... PDF generation
  doc.save("document.pdf");
};
```

**Estimated impact: −300 KB from DocumentDetailView chunk.**

---

### Phase 4: Vendor Splitting (Cache Optimization)

#### Step 4.1: Configure `manualChunks` in `vite.config.mjs`

```javascript
// In rollupOptions.output:
output: {
  entryFileNames: "[name].[hash].js",
  chunkFileNames: "[name].[hash].js",
  assetFileNames: "[name].[hash].[ext]",
  manualChunks(id) {
    // ── Framework Core (rarely changes) ──
    if (id.includes("node_modules/react/") ||
        id.includes("node_modules/react-dom/") ||
        id.includes("node_modules/react-router")) {
      return "vendor-react";
    }

    // ── State Management (rarely changes) ──
    if (id.includes("node_modules/@reduxjs/") ||
        id.includes("node_modules/react-redux") ||
        id.includes("node_modules/redux") ||
        id.includes("node_modules/redux-persist")) {
      return "vendor-redux";
    }

    // ── Monitoring & Analytics ──
    if (id.includes("node_modules/@sentry/")) {
      return "vendor-sentry";
    }

    // ── Charting (only loaded with Dashboard) ──
    if (id.includes("node_modules/recharts") ||
        id.includes("node_modules/d3-")) {
      return "vendor-charts";
    }

    // ── PDF Generation (only loaded on-demand) ──
    if (id.includes("node_modules/jspdf")) {
      return "vendor-pdf";
    }

    // ── UI Libraries ──
    if (id.includes("node_modules/react-select") ||
        id.includes("node_modules/react-datepicker") ||
        id.includes("node_modules/react-toastify")) {
      return "vendor-ui";
    }

    // ── i18n ──
    if (id.includes("node_modules/i18next") ||
        id.includes("node_modules/react-i18next")) {
      return "vendor-i18n";
    }

    // ── Remaining node_modules ──
    if (id.includes("node_modules/")) {
      return "vendor-misc";
    }
  },
},
```

#### Vendor Splitting Benefits

```
Before:
  main.abc123.js  →  2.5 MB  (EVERYTHING together, cache invalidated on every deploy)

After:
  main.abc123.js          →  ~150 KB  (app code, changes frequently)
  vendor-react.def456.js  →  ~180 KB  (changes only when updating React)
  vendor-redux.ghi789.js  →  ~60 KB   (changes only when updating Redux)
  vendor-sentry.jkl012.js →  ~250 KB  (changes only when updating Sentry)
  vendor-charts.mno345.js →  ~500 KB  (ONLY loaded on /dashboard)
  vendor-pdf.pqr678.js    →  ~300 KB  (ONLY loaded when generating PDF)
  vendor-ui.stu901.js     →  ~250 KB  (loaded with first form)
  vendor-i18n.vwx234.js   →  ~40 KB   (changes only when updating i18n)
  vendor-misc.yz5678.js   →  ~80 KB   (remaining dependencies)

  + 55 route chunks of ~5-30 KB each
```

**Result:** Vendor chunks are cached in browser and **not re-downloaded** on successive deploys (unless library versions change).

---

### Phase 5: Complete `vite.config.mjs` Optimization

Below is the optimized configuration file with all improvements integrated:

```javascript
import { defineConfig, transformWithEsbuild } from "vite";
import react from "@vitejs/plugin-react";
import { sentryVitePlugin } from "@sentry/vite-plugin";
import { visualizer } from "rollup-plugin-visualizer";
import { MONITORING_CONFIG, ENV } from "@/config/parameters";

export default defineConfig(() => {
  const isAnalyze = process.env.ANALYZE === "true";

  const BASE_CONFIG = {
    assetsInclude: ["**/*.xlsx"],
    resolve: {
      alias: {
        "@": "/src",
      },
    },
    plugins: [
      {
        name: "treat-js-files-as-jsx",
        async transform(code, id) {
          if (!id.match(/src\/.*\.js$/)) return null;
          return transformWithEsbuild(code, id, {
            loader: "jsx",
            jsx: "automatic",
          });
        },
      },
      react(),
      sentryVitePlugin({
        org: MONITORING_CONFIG.ORG.SLUG,
        project: MONITORING_CONFIG.PROJECT.NAME,
        release: MONITORING_CONFIG.PROJECT.RELEASE,
        authToken: MONITORING_CONFIG.ORG.AUTH_TOKEN,
        url: MONITORING_CONFIG.URL,
        include: "./src",
        ignore: ["node_modules", "dist"],
        configFile: "./monitoring.properties",
        urlPrefix: "~/",
      }),
      // Bundle Analyzer — only when run with ANALYZE=true
      isAnalyze && visualizer({
        filename: "bundle-analysis.html",
        open: true,
        gzipSize: true,
        brotliSize: true,
        template: "treemap",
      }),
    ].filter(Boolean),
    optimizeDeps: {
      force: true,
      esbuildOptions: {
        loader: {
          ".js": "jsx",
        },
      },
    },
    server: {
      port: 3000,
    },
    build: {
      outDir: "build",
      sourcemap: "hidden",
      assetsInlineLimit: 0,
      emptyOutDir: true,
      rollupOptions: {
        cache: false,
        input: {
          main: "./index.html",
        },
        output: {
          entryFileNames: "[name].[hash].js",
          chunkFileNames: "[name].[hash].js",
          assetFileNames: "[name].[hash].[ext]",
          manualChunks(id) {
            if (id.includes("node_modules/react/") ||
                id.includes("node_modules/react-dom/") ||
                id.includes("node_modules/react-router")) {
              return "vendor-react";
            }
            if (id.includes("node_modules/@reduxjs/") ||
                id.includes("node_modules/react-redux") ||
                id.includes("node_modules/redux") ||
                id.includes("node_modules/redux-persist")) {
              return "vendor-redux";
            }
            if (id.includes("node_modules/@sentry/")) {
              return "vendor-sentry";
            }
            if (id.includes("node_modules/recharts") ||
                id.includes("node_modules/d3-")) {
              return "vendor-charts";
            }
            if (id.includes("node_modules/jspdf")) {
              return "vendor-pdf";
            }
            if (id.includes("node_modules/react-select") ||
                id.includes("node_modules/react-datepicker") ||
                id.includes("node_modules/react-toastify")) {
              return "vendor-ui";
            }
            if (id.includes("node_modules/i18next") ||
                id.includes("node_modules/react-i18next")) {
              return "vendor-i18n";
            }
            if (id.includes("node_modules/")) {
              return "vendor-misc";
            }
          },
        },
      },
    },
    test: {
      globals: true,
      include: ["**/test.{ts,tsx}", "**/__tests__/**/*.{js,jsx,ts,tsx}"],
    },
  };

  return BASE_CONFIG;
});
```

#### Analysis Script in `package.json`

```json
{
  "scripts": {
    "analyze": "cross-env ANALYZE=true vite build"
  }
}
```

> If not using `cross-env`, in PowerShell: `$env:ANALYZE='true'; vite build`

---

## 6. Results Report Template (Before/After)

### 6.1 Bundle Size Metrics

| Chunk | Before (raw) | Before (gzip) | After (raw) | After (gzip) | Delta |
|-------|-------------|---------------|-------------|--------------|-------|
| `main.js` (entry) | ___ KB | ___ KB | ___ KB | ___ KB | __% |
| `vendor-react.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-redux.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-sentry.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-charts.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-pdf.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-ui.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-i18n.js` | — | — | ___ KB | ___ KB | N/A |
| `vendor-misc.js` | — | — | ___ KB | ___ KB | N/A |
| Route chunks (sum) | — | — | ___ KB | ___ KB | N/A |
| **TOTAL** | ___ KB | ___ KB | ___ KB | ___ KB | __% |
| **Initial Load** | ___ KB | ___ KB | ___ KB | ___ KB | **__%** |

> **Initial Load** = `main.js` + `vendor-react` + `vendor-redux` + `vendor-i18n` + `vendor-misc` + active route.

### 6.2 Core Web Vitals (Lighthouse Mobile, Simulated Throttling)

| Metric | Before | After | Delta | Google Threshold |
|---------|--------|-------|-------|-----------------|
| **FCP** (First Contentful Paint) | ___ ms | ___ ms | __% | < 1,800 ms ✅ |
| **LCP** (Largest Contentful Paint) | ___ ms | ___ ms | __% | < 2,500 ms ✅ |
| **TTI** (Time to Interactive) | ___ ms | ___ ms | __% | < 3,800 ms ✅ |
| **TBT** (Total Blocking Time) | ___ ms | ___ ms | __% | < 200 ms ✅ |
| **CLS** (Cumulative Layout Shift) | ___ | ___ | __% | < 0.1 ✅ |
| **Performance Score** | ___/100 | ___/100 | +___ pts | > 90 ✅ |

### 6.3 Route Breakdown (Top 5 Most Visited)

| Route | JS Loaded (Before) | JS Loaded (After) | Reduction |
|-------|---------------------|-------------------|-----------|
| `/login` | ___ KB | ___ KB | __% |
| `/dashboard` | ___ KB | ___ KB | __% |
| `/invoices` | ___ KB | ___ KB | __% |
| `/reports` | ___ KB | ___ KB | __% |
| `/invoice-detail` | ___ KB | ___ KB | __% |

### 6.4 Measurement Protocol

```bash
# 1. Baseline — before changes
npm run build
# Record build/assets/*.js sizes

# 2. Lighthouse — 5 runs, median
# Use Chrome DevTools > Lighthouse > Mobile > Performance
# Or CLI:
npx lighthouse http://localhost:3000/login --output=json --output-path=./lighthouse-before.json

# 3. Apply optimization changes

# 4. Post-optimization
npm run build
# Record new sizes

# 5. Lighthouse post
npx lighthouse http://localhost:3000/login --output=json --output-path=./lighthouse-after.json

# 6. Visual bundle analysis
npm run analyze
# Compare treemaps before/after
```

### 6.5 Expected Impact Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                 CONSOLIDATED IMPACT PROJECTION                │
├──────────────────────────────────┬───────────────────────────────┤
│ Optimization                     │ Estimated Reduction           │
├──────────────────────────────────┼───────────────────────────────┤
│ react-icons named imports        │ −500 KB to −1 MB               │
│ Code splitting (55 lazy routes)  │ −60% to −75% initial bundle    │
│ Vendor splitting (9 chunks)      │ Cache hit rate: ~90% on redep │
│ Remove core-js polyfills         │ −60 KB to −80 KB               │
│ jsPDF dynamic import             │ −300 KB from invoice chunk     │
│ Remove redux-logger (prod)       │ −15 KB                         │
├──────────────────────────────────┼───────────────────────────────┤
│ TOTAL Initial Load               │ From ~2-3 MB → ~300-500 KB     │
│ FCP Improvement                  │ −40% to −60%                   │
│ Lighthouse Score                 │ +30 to +50 points              │
└──────────────────────────────────┴───────────────────────────────┘
```

---

## Appendix A: Recommended Execution Order

| Priority | Task | Effort | Impact | Risk |
|----------|------|--------|--------|------|
| 🔴 P0 | Fix `import *` from react-icons | 2-4h | Very High | Low |
| 🔴 P0 | Lazy loading of 55 routes | 4-6h | Very High | Medium |
| 🟡 P1 | Vendor splitting in vite.config.mjs | 1-2h | High | Low |
| 🟡 P1 | Remove core-js polyfills | 30 min | Medium | Low |
| 🟡 P1 | Dynamic import of jsPDF | 1h | High | Low |
| 🟢 P2 | Remove/conditional redux-logger | 30 min | Low | Low |
| 🟢 P2 | Integrate bundle analyzer | 30 min | Diagnostic | None |
| 🟢 P2 | Optimize Monitoring imports | 1-2h | Medium | Medium |

## Appendix B: Bibliographic References

1. Osmani, A. (2023). *The Cost of JavaScript in 2023*. Google Chrome Team. https://v8.dev/blog/cost-of-javascript-2019
2. Google. (2024). *Web Vitals*. https://web.dev/vitals/
3. Deloitte. (2020). *Milliseconds Make Millions*. https://www2.deloitte.com/ie/en/pages/consulting/articles/milliseconds-make-millions.html
4. HTTP Archive. (2023). *Web Almanac — JavaScript Chapter*. https://almanac.httparchive.org/en/2022/javascript
5. Vite Documentation. (2024). *Build Optimizations*. https://vitejs.dev/guide/build.html
6. Rollup Documentation. (2024). *Code Splitting*. https://rollupjs.org/tutorial/#code-splitting


🔒 Confidentiality Notice

All project names, component identifiers, database schemas, and business logic terminology in this technical vault have been fully anonymized and abstracted. The cases presented here reflect my technical methodology, research, and engineering impact, not the proprietary intellectual property of past clients. All metrics presented (e.g., bundle reduction, performance scores) are accurate results of my engineering work.