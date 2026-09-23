# Aegis Compliance Console — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a separate analyst-facing web app that lets a tenant user log in, work an alert queue, inspect explainable alert evidence / related transactions, and navigate customers—backed by the Aegis API vertical slice (plus thin operator API additions called out below).

**Architecture:** New Next.js App Router SPA/BFF-style client (`aegis-console`), **not** the marketing site (`aegis-web`). Browser talks only to Aegis REST (`/api/v1/*`) with JWT in memory (or httpOnly cookie via optional thin BFF later). Server Components for read shells; client components for interactive queues and forms. No business rules in the UI—detection remains backend-owned.

**Tech Stack:** Next.js 15+ (App Router), TypeScript, Tailwind CSS 4, React 19; fetch-based API client; no auth library lock-in initially (custom JWT session). Prefer existing patterns from `aegis-web` only for tooling familiarity—**do not** merge marketing and console into one deployable.

## Global Constraints

- Keep **`aegis-web` marketing-only**; compliance UI is a **new repo folder** `aegis-console/` (sibling to `aegis`, `aegis-web`, `aegis-docs`).
- Packages younger than 14 days: do not add without explicit override.
- Never put JWT signing secrets or connection strings in the frontend.
- Tenant identity never from UI forms—only JWT claims.
- UI must not invent audit events; display only.
- Design: avoid generic purple-dashboard AI look; brand-first where landing-like; for console, prioritize clarity and density of investigation workflows over marketing hero patterns.
- Backend prerequisites from engineering review must land **before** or **in lockstep** with console PRs that need them (see Gate 0).
- YAGNI: no cases, screening, rule editor, or bulk upload UI in v1 console.

---

## Gate 0 — Backend API prerequisites (blockers for a usable console)

These are **not** frontend work; track as `aegis` PRs and merge before Console PR2+.

| API need | Why console needs it | Suggested endpoint |
|----------|----------------------|--------------------|
| Alert filters + pagination | Queue unusable without | `GET /api/v1/alerts?status=&severity=&from=&to=&page=&pageSize=` |
| Alert assign / status | Analyst workflow | `POST /api/v1/alerts/{id}/assign`, `POST .../dismiss` or `.../resolve` |
| Transaction list | Evidence drill-down | `GET /api/v1/transactions?customerId=&from=&to=&page=` |
| Customer list/search | Navigation | `GET /api/v1/customers?q=&page=` |
| Rule read (active) | Explainability UI | `GET /api/v1/rules`, `GET /api/v1/rules/{id}` (active version + definition) |
| Auth hygiene | Safe demos | Secrets via env; `AllowDevBootstrap` off outside Development |

Existing (already usable):

```text
POST /api/v1/auth/login
POST /api/v1/customers  GET /api/v1/customers/{id}
POST /api/v1/customers/{id}/accounts  GET /api/v1/accounts/{id}
POST /api/v1/transactions  GET /api/v1/transactions/{id}
GET  /api/v1/alerts  GET /api/v1/alerts/{id}
GET  /api/v1/audit-events  GET /api/v1/audit-events/{id}
GET  /api/v1/status  GET /health
```

Dev-only: `POST /api/v1/tenants` (bootstrap)—console may call it only behind a “Local demo” panel when `NEXT_PUBLIC_ALLOW_DEV_BOOTSTRAP=true`.

---

## Target layout

```text
aegis-console/
  package.json
  next.config.ts
  .env.example                 # NEXT_PUBLIC_AEGIS_API_BASE_URL=
  src/
    app/
      layout.tsx
      page.tsx                 # redirect → /alerts or /login
      login/page.tsx
      (app)/
        layout.tsx             # shell: nav + tenant badge
        alerts/page.tsx        # queue
        alerts/[id]/page.tsx   # detail + evidence
        customers/page.tsx
        customers/[id]/page.tsx
        transactions/[id]/page.tsx
        audit/page.tsx         # read-only recent events
    components/
      auth/
      alerts/
      customers/
      ui/                      # buttons, tables, badges — minimal
    lib/
      api/client.ts            # fetch wrapper, 401 → login
      api/types.ts             # DTOs matching backend JSON
      auth/session.ts          # token storage + claims parse
      config.ts
    styles/globals.css
  tests/                       # Playwright smoke (PR5)
```

---

## PR sequence (web)

| PR | Deliverable | Depends on |
|----|-------------|------------|
| **W0** | Scaffold `aegis-console` + API client + login | Existing login API |
| **W1** | App shell + alert queue (list/detail read-only) | Existing alerts GET; better with Gate 0 filters |
| **W2** | Alert investigation UX (evidence, linked txs, customer) | Tx/customer GET; ideally list APIs |
| **W3** | Alert actions (assign / dismiss) | Gate 0 action APIs |
| **W4** | Customers + transactions browse | Gate 0 list APIs |
| **W5** | Rules read-only + audit trail viewer + Playwright smoke | Gate 0 rules GET |

---

### Task W0.1 — Scaffold console project

**Files:**
- Create: `aegis-console/package.json`, `next.config.ts`, `tsconfig.json`, `src/app/layout.tsx`, `src/styles/globals.css`, `.env.example`, `README.md`

- [ ] **Step 1:** `npx create-next-app@latest aegis-console` (TypeScript, App Router, Tailwind, ESLint; no marketing content).
- [ ] **Step 2:** Set `NEXT_PUBLIC_AEGIS_API_BASE_URL=http://localhost:5092` in `.env.local` (gitignored).
- [ ] **Step 3:** README: how to run API + console; note separation from `aegis-web`.
- [ ] **Step 4:** Commit `chore: scaffold aegis-console`.

---

### Task W0.2 — API client + session

**Files:**
- Create: `src/lib/config.ts`, `src/lib/api/client.ts`, `src/lib/api/types.ts`, `src/lib/auth/session.ts`

- [ ] **Step 1:** Define types for LoginResponse, Alert, Transaction, Customer, AuditEvent matching current JSON (camelCase).
- [ ] **Step 2:** `apiFetch(path, { method, body, token })` — attach `Authorization: Bearer`, throw typed `ApiError` on non-2xx; on 401 clear session.
- [ ] **Step 3:** Session helpers: `setAccessToken`, `getAccessToken`, `clearSession`, `getTenantIdFromJwt` (decode payload only; do not trust for authz).
- [ ] **Step 4:** Unit-test JWT claim parse with a fixed token string (no network).
- [ ] **Step 5:** Commit `feat(console): api client and session helpers`.

---

### Task W0.3 — Login page

**Files:**
- Create: `src/app/login/page.tsx`, `src/components/auth/LoginForm.tsx`
- Modify: `src/app/page.tsx` (redirect)

- [ ] **Step 1:** Form: email, password, tenantSlug → `POST /api/v1/auth/login`.
- [ ] **Step 2:** On success store token; redirect `/alerts`.
- [ ] **Step 3:** Show API error message on 401; no stack traces.
- [ ] **Step 4:** Manual test against local API.
- [ ] **Step 5:** Commit `feat(console): login against Aegis API`.

---

### Task W1.1 — Authenticated app shell

**Files:**
- Create: `src/app/(app)/layout.tsx`, `src/components/auth/RequireAuth.tsx`, `src/components/shell/AppNav.tsx`

- [ ] **Step 1:** Guard `(app)/*` — if no token, redirect `/login`.
- [ ] **Step 2:** Nav links: Alerts, Customers, Audit; show truncated tenant id / email from JWT; Logout clears session.
- [ ] **Step 3:** Optional: ping `GET /api/v1/status` for health badge.
- [ ] **Step 4:** Commit `feat(console): authenticated app shell`.

---

### Task W1.2 — Alert queue (read-only)

**Files:**
- Create: `src/app/(app)/alerts/page.tsx`, `src/components/alerts/AlertTable.tsx`, `src/components/alerts/SeverityBadge.tsx`
- Create: `src/lib/api/alerts.ts`

- [ ] **Step 1:** `listAlerts()` → `GET /api/v1/alerts`.
- [ ] **Step 2:** Table columns: severity, status, rule name/code, triggeredAt, focusEntityId, id link.
- [ ] **Step 3:** Empty / loading / error states.
- [ ] **Step 4:** When Gate 0 filters exist, add query controls (status, severity, date); until then document “unfiltered list (max 100)”.
- [ ] **Step 5:** Commit `feat(console): alert queue`.

---

### Task W1.3 — Alert detail (explainability)

**Files:**
- Create: `src/app/(app)/alerts/[id]/page.tsx`, `src/components/alerts/EvidencePanel.tsx`, `src/components/alerts/ConditionList.tsx`

- [ ] **Step 1:** `GET /api/v1/alerts/{id}`; 404 page if missing.
- [ ] **Step 2:** Show ruleVersionId, ruleVersionNumber, deduplicationKey, evaluatedValues, conditionsSatisfied, transactionIds, additionalContext.conditionFacts.
- [ ] **Step 3:** Link each transactionId → `/transactions/{id}`; focusEntityId → `/customers/{id}` when GUID.
- [ ] **Step 4:** Manual: run structuring demo via API/Swagger, open alert in console—verify evidence matches.
- [ ] **Step 5:** Commit `feat(console): alert detail explainability`.

---

### Task W2.1 — Transaction + customer detail pages

**Files:**
- Create: `src/app/(app)/transactions/[id]/page.tsx`, `src/app/(app)/customers/[id]/page.tsx`
- Create: `src/lib/api/transactions.ts`, `src/lib/api/customers.ts`

- [ ] **Step 1:** Load tx/customer by id; show amounts as decimal strings (no float formatting bugs).
- [ ] **Step 2:** Customer page: basic profile + link to create-account is **out of scope** for analysts (optional “demo tools” later).
- [ ] **Step 3:** Commit `feat(console): customer and transaction detail`.

---

### Task W3.1 — Alert actions (requires Gate 0)

**Files:**
- Modify: `src/app/(app)/alerts/[id]/page.tsx`
- Create: `src/components/alerts/AlertActions.tsx`

- [ ] **Step 1:** Confirm backend `assign` / `dismiss` (or `resolve`) merged.
- [ ] **Step 2:** Buttons call APIs; optimistic UI or refetch detail.
- [ ] **Step 3:** Disable actions when unauthenticated/error; show toast/inline error.
- [ ] **Step 4:** Commit `feat(console): alert assign and dismiss`.

---

### Task W4.1 — Browse customers & transactions (requires Gate 0 lists)

**Files:**
- Modify: `src/app/(app)/customers/page.tsx`
- Create: `src/app/(app)/transactions/page.tsx` (optional list)

- [ ] **Step 1:** Wire list endpoints with simple search box + pagination.
- [ ] **Step 2:** Commit `feat(console): customer and transaction browse`.

---

### Task W5.1 — Rules read-only + audit viewer

**Files:**
- Create: `src/app/(app)/rules/page.tsx`, `src/app/(app)/rules/[id]/page.tsx`, `src/app/(app)/audit/page.tsx`

- [ ] **Step 1:** Display active structuring rule JSON (pretty-print) for transparency.
- [ ] **Step 2:** Audit page: `GET /api/v1/audit-events` table (eventType, entity, actor, occurredAt); no create UI.
- [ ] **Step 3:** Commit `feat(console): rules and audit read views`.

---

### Task W5.2 — Playwright smoke

**Files:**
- Create: `aegis-console/playwright.config.ts`, `tests/smoke/login-alerts.spec.ts`

- [ ] **Step 1:** Smoke: login (seeded/bootstrap tenant) → alerts page loads → open first alert if any.
- [ ] **Step 2:** Document prerequisite: API up + migrate + optional seed script.
- [ ] **Step 3:** Commit `test(console): playwright login and alerts smoke`.

---

## Non-goals (console v1)

- Merging into `aegis-web`
- Rule authoring / simulation UI
- Case management UI
- KYC document upload
- Real-time websockets
- Multi-currency UX
- Mobile-native apps
- Production SSO (Azure AD / Okta)—stub “SSO later”; password login only for now

---

## Suggested demo script (after W1)

1. Start Postgres + `dotnet run` API (`:5092`).
2. `npm run dev` in `aegis-console`.
3. Bootstrap tenant (Swagger or console demo panel) → login.
4. Seed structuring scenario (API or small script) → open Alerts → open HIGH alert → verify conditions and tx links.

---

## Success criteria

- [ ] Analyst can log in and see tenant-scoped alerts only.
- [ ] Alert detail shows rule version + evidence without calling engine locally.
- [ ] No UI path to invent audit rows.
- [ ] Marketing site unchanged and separately deployable.
- [ ] Console README documents API base URL and Gate 0 dependencies.

---

## Execution handoff

Plan complete. Recommended order:

1. Finish **Gate 0** backend APIs (filter/actions/lists/rules read).  
2. Execute **W0 → W1** for a demoable console on current GETs.  
3. Then W3–W5 as APIs land.

Which approach for implementation?

1. **Subagent-Driven** — fresh agent per W-task  
2. **Inline** — execute in-session with checkpoints after each PR  
