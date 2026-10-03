# Aegis Build Roadmap (October 2026)

**Status:** PROPOSED. Needs owner sign-off on phase order before Phase 1 starts.

**Goal:** take Aegis from the current vertical slice to the PRD's MVP Definition of Done (§65), then into the
post-MVP items raised by the [KYC in 2026 research](../research/2026-10-01-sumsub-kyc-in-2026.md) (§67.1).

**How to use this file:** each phase is a list of small PRs. Each PR gets its own detailed plan in
`plans/` (same format as [Phase 0](./2026-10-01-phase-0-hardening.md)) just before it starts, so plans
reflect the code as it is then, not as it was guessed now.

---

## 1. Where we are (audit, 2026-10-01)

### Backend (`aegis`, .NET 9, PostgreSQL 16, modular monolith)

| PRD milestone | State | What exists | What's missing |
|---|---|---|---|
| M1 Platform core | Partial | JWT login, tenants, repository pattern, audit foundation | Real RBAC (roles/permissions tables unused; only hardcoded Admin/Analyst), user and tenant management APIs |
| M2 Customers and transactions | Partial | Individual/business customers, accounts, single ingest, idempotency | CSV / batch import, normalization pipeline |
| M3 Rule engine | Mostly done | Definitions, validator, versions, draft/activate, 3 scenarios | New rule creation, retire/suspend, simulation; feature windows hardcoded (ignore `Schedule.Lookback`) |
| M4 Alerts | Partial | Creation with evidence and rule version, severity, dedupe, assign/dismiss/resolve, filters | Escalate, dismissal reason, assignee validation, search |
| M5 Cases | Thin | Create from alert, assign, notes, close with disposition | Link more alerts, timeline, customer 360, escalation |
| M6 Risk and dashboard | Missing | — | Risk engine, factors, scores, dashboard and rule metrics |
| M7 Screening | Missing | Empty module | Everything |
| M8 Hardening | Mostly missing | Serilog, DB health check, CI, Dockerfile, some isolation tests | Rate limiting, OpenTelemetry, perf tests, backups, dependency scanning |

**Confirmed defects** (fixed in Phase 0):

- Audit `AfterState` is built by string interpolation from user input into a `jsonb` column (`AlertsController.cs:138`, `CasesController.cs:112`, and six more). A quote in `assignedTo` breaks the insert, and crafted input can inject keys into audit rows.
- Alert assign/dismiss/resolve/create-case and case assign/notes have no role check.
- Login accepts users of suspended or terminated tenants.
- CORS allows any origin; no rate limiting on login.
- `Database.Migrate()` runs on every startup in every environment.
- `Dockerfile` doesn't copy the `Application`, `Identity`, and `Customers` project files before restore.

**Structural debt:** no EF global query filters for tenant isolation (every repository must remember the
predicate); feature calculator sums amounts across currencies; `IngestAndEvaluateStructuring` evaluates all
rules and re-seeds on every ingest; Redis and OpenTelemetry are referenced but unused.

### Console (`aegis-console`, Next 16, React 19, Tailwind 4)

Real API, no mocks. Pages: login, alerts queue and detail (evidence, actions), cases list and detail (notes,
close), customers search and profile, transaction detail, rules list and detail (activate), audit list.

Missing: dashboard, customer 360, case timeline, escalation, rule authoring and simulation, risk, screening,
user admin, role-aware UI, pagination, transaction list, readable evidence (raw JSON today). Every page is a
client component with duplicated fetch/loading/error and table code. A 401 clears the token without redirecting.
The [console plan](./2026-09-23-aegis-compliance-console.md) is out of date.

### Marketing site (`aegis-web`, Next 16, static export on Firebase Hosting)

The "Request access" form discards submissions. No OpenGraph, sitemap, robots, or canonical URLs. Privacy,
Terms, and Security links go to `/#contact`. Testimonials and "100%" stats are unattributed. Autoplay carousel
has no pause or reduced-motion handling. Four unused components. README describes a different stack. No tests.

**Update 2026-10-03:** redesigned on branch `marketing-redesign` (see
[design](../designs/2026-10-03-marketing-site.md)). Homepage cut from 14 sections to 7, built features shown
with real console screenshots, screening and simulation marked "Planned", testimonials and invented stats
removed, dead Privacy/Terms/Security links removed, the form replaced by a prefilled mailto link, and the
visual system matched to the console. `lib/` logic has unit tests. Still open: OpenGraph, sitemap,
robots, canonical URLs, README, and swapping the placeholder `access@aegis.example` address.

### Environment

`dotnet` isn't installed on the dev machine and neither frontend has `node_modules`, so nothing has been
built or tested locally during this audit. Phase 0 starts by fixing that.

---

## 2. Principles for every PR

- One concern per PR, reviewable in under 30 minutes. Backend and console changes for the same feature ship as two PRs, backend first.
- TDD: failing test first. Every bugfix gets a regression test.
- Tenant isolation test for every new read or write endpoint.
- Every state change writes an audit event built with `AuditPayload.Json` (Phase 0), never string interpolation.
- No new package younger than 14 days. Prefer built-in .NET and Next.js features (rate limiting, OpenTelemetry exporters already referenced).
- API contract changes update the console types in the same sprint.

---

## 3. Phases

### Phase 0 — Toolchain and security hardening (backend) · ~1 week

Detailed plan: [2026-10-01-phase-0-hardening.md](./2026-10-01-phase-0-hardening.md).

| PR | Scope |
|---|---|
| 0.1 | Install .NET 9 SDK, run baseline build and tests, record results |
| 0.2 | `AuditPayload.Json` helper; replace all interpolated audit JSON |
| 0.3 | `CanWorkAlerts` authorization on alert and case work actions |
| 0.4 | Reject login for non-active tenants |
| 0.5 | CORS allowlist from config |
| 0.6 | Login rate limiting (built-in ASP.NET Core limiter) |
| 0.7 | Migrations on startup behind config; Dockerfile restore fix |

**Exit:** all tests green; none of the confirmed defects reproduce.

**Status: DONE (2026-10-01)** on `aegis` branch `phase-0-hardening`, awaiting review and merge. See the plan's exit checklist for deviations.

### Phase 1 — Close MVP gaps (M1, M2, M4, M5) · ~3 weeks

Backend:

| PR | Scope |
|---|---|
| 1.1 | ✅ EF global query filters for `TenantId` plus an isolation test per aggregate (alerts and audit currently untested) |
| 1.2 | ✅ RBAC: PRD §10 permission catalog and built-in `Admin`, `Reviewer`, `Analyst`, `Viewer` grants in `Aegis.Shared/Security/Permissions.cs`, enforced with `[RequirePermission]` on every endpoint. The grants are code, not the `identity.roles` table; tenant-configurable roles are deferred until a pilot needs them. Analysts no longer read the full audit log (Reviewer and Admin do) |
| 1.3 | ✅ User management API: list, create, change roles, deactivate (`user.manage`, audited). Admins cannot drop their own Admin role or deactivate themselves. A deactivated user's existing JWT stays valid until expiry; revocation lands with refresh tokens in Phase 5 |
| 1.4 | ✅ Alert escalate endpoint (`alert.escalate`, Analyst and up) and required dismissal reason (stored on the alert, migration `AlertDismissalReason`); assignee must be the id of an active tenant user; resolved or dismissed alerts return 409. Console has the reason form and escalate button |
| 1.5 | ✅ `GET /cases/{id}/timeline` (audit events on the case and its linked alerts, plus note text; `case.read`), `POST /cases/{id}/alerts` (idempotent link), `POST /cases/{id}/escalate`. Case assignee is validated like alerts; closed cases return 409. Notes now audit as `CASE_NOTE_ADDED`. New `(tenant_id, EntityId)` index on audit events |
| 1.6 | ✅ `GET /customers/{id}/overview`: profile, accounts, 50 most recent transactions, alerts and cases, plus summary counts. Each section is only returned when the caller holds its read permission. Open alert and case counts cover the 50 most recent items |
| 1.7 | ✅ Synchronous `POST /transactions/batch` (JSON) and `POST /transactions/import` (multipart CSV, 5 MB): up to 1,000 rows, processed in order, each row in its own scope and unit of work, per-row `CREATED`/`DUPLICATE`/`FAILED` results with row or line numbers. Same external-reference idempotency as single ingest. An async job queue is deferred until pilots send files larger than 1,000 rows |
| 1.8 | ✅ Each rule version is evaluated over its `Schedule.Lookback` (up to 90 days; features calculated once per distinct window). Generic features `transaction_count`, `transaction_sum`, `max_single_amount`, `credit_sum`, `debit_sum`, `pass_through_ratio` cover that window, and the `_24h` and `_1h` names keep their fixed windows. Aggregates only use the triggering transaction's currency, and there is a new `currency` feature. The KES-only restriction is lifted: accounts take any ISO 4217 code and a transaction must match its account's currency. Seeded rules now include `currency IN [KES]`, but rules seeded for existing tenants don't have it, so add a new version if those tenants take other currencies. Renamed to `IngestAndEvaluateRules`; rules are seeded at tenant bootstrap only |
| 1.9 | ✅ Audit list: paging (newest first, up to 200 per page) and filters (event type, entity type, entity id, actor, date range). Responses use the same `{ items, page, pageSize, totalCount }` shape as other lists and include actor role, before/after state and correlation id. The console audit page has event-type and entity filters and paging |

Console:

| PR | Scope |
|---|---|
| 1.10 | ✅ Shared UI kit (`components/ui`: `DataTable`, `Pagination`, `Field`/`Button`, loading/empty/error states, `PageHeader`, `Stat`) and a `useApiQuery` loading hook. List pages use server-side paging and `lib/api` modules. Console lint is down from 5 errors to 0 |
| 1.11 | ✅ When the session disappears (an API 401 clears it), the console redirects to `/login?returnTo=…`; only same-app paths are accepted. An explicit logout carries no return URL. The nav and actions are filtered by a client mirror of the role grants (the API still enforces everything) |
| 1.12 | ✅ Evidence table: feature / actual / rule / threshold / met, built from `conditionFacts`. All evaluated features and the raw JSON sit behind toggles |
| 1.13 | ✅ Customer 360 on `GET /customers/{id}/overview` (accounts, alerts, cases, recent transactions, counts; sections the user can't read are hidden). Transactions list with customer and date filters |
| 1.14 | ✅ Case page: timeline (audit events plus notes, including escalation reasons), escalate, link alert, assign to a named user. Adds `GET /api/v1/users/assignees`: active users who can work cases, id/name/email only, requires `case.update` |
| 1.15 | ✅ Audit filters and paging (done in 1.9). CSV import screen: client-side checks (.csv, ≤ 5 MB), template download, created/duplicate/failed counts, per-row results linking to transactions and alerts |
| 1.16 | ✅ Admin users page: create, change roles, deactivate (not offered on your own row; the API also blocks self-lockout) |

Open from the exit criterion: the Playwright demo flow is not automated yet. Phase 1 was checked by hand in the browser against a seeded tenant (alert → case → note → link → assign → escalate → customer 360 → users → analyst role gating).

**Exit:** every item in PRD §65 except risk and screening works from a clean environment, covered by the Playwright demo flow.

### Phase 2 — Risk engine and dashboard (M6) · ~2 weeks

| PR | Scope |
|---|---|
| 2.1 | ✅ Risk module: a tenant risk model with weighted factors (high-risk geography, customer type, open alerts, high/critical alerts in a window, cases closed as suspicious or reported, transaction volume in a window) and LOW/MEDIUM/HIGH/CRITICAL band thresholds. Scores are the sum of factor points, capped at 100. Saving a change creates a new immutable version (`risk.manage`, Admin only, audited as `RISK_MODEL_UPDATED`); a default model is created the first time a tenant needs one. PEP and screening factors wait for Phase 3 |
| 2.2 | ✅ Customers are rescored when they are created, when an ingested transaction raises a new alert, nightly (02:00 UTC, `Risk:NightlyBatch:Enabled`/`HourUtc`), manually, and through `POST /risk/recalculate-all` (`risk.manage`). A scoring failure is logged and never fails the customer create or ingest that triggered it. Every score is kept with its model version, trigger and per-factor points; a band change is audited as `RISK_SCORE_CHANGED` |
| 2.3 | ✅ `GET /customers/{id}/risk` (current score plus the last 20; customers created before scoring existed are scored on first view) and `POST /customers/{id}/risk/recalculate` (`customer.write`). `GET`/`PUT /risk/model` |
| 2.4 | ✅ `GET /dashboard`: open alerts by severity and age, alerts today, high-risk open, alert SLA breaches, a 14-day alert trend, top rules over 30 days with dismissal rate, cases by status and overdue, transactions ingested today and over 7 days, and the customer risk distribution. Sections the caller can't read are null. SLAs default to 3 days for alerts and 14 for cases (`Dashboard:AlertSlaDays`/`CaseSlaDays`) because PRD §67 leaves them open |
| 2.5 | ✅ Console dashboard is the home route and the post-login default: stat tiles, CSS bar charts (no chart library), severity bars linking to the filtered alert queue (the alerts page now reads `?status=`/`?severity=`), top-rules table |
| 2.6 | ✅ Customer page risk panel: score, band, change since the last score, per-factor points with explanations, history. Admin "Risk model" page to edit factors and bands, save a new version, and rescore all customers. API error messages no longer show JSON quotes |

Checked by hand in the browser against a seeded tenant: dashboard figures, lazy first score (31, MEDIUM), an invalid band order rejected with the API message, a saved version adding `KE`, and a rescore to 61 (HIGH).

### Console workbench redesign (2026-10-03)

Done on branch `console-workbench` in `aegis` and `aegis-console` (design `designs/2026-10-03-console-workbench.md`, plan `plans/2026-10-03-console-workbench.md`).

- API, additive only: alert, case and transaction responses carry customer and assignee names (resolved in batches through `IDisplayNameLookup`, no per-row queries); cases list their linked alerts by rule name, severity and status; customer responses carry the latest risk band. `GET /alerts` takes `view=open|mine|unassigned|pastSla` and `customerId`; `GET /settings/sla` exposes the SLA days to any signed-in user.
- Console: sidebar shell grouped by job with open counts, a customer jump box and a loading skeleton instead of the "Checking session…" flash. The alert queue has view tabs, customer names, age against the SLA and readable status chips. Alert and case pages are two-column workspaces: a plain-language headline, a neutral check for met conditions, related transactions, and a right rail with actions, SLA and a customer card. Amounts read `KES 95,000.00` and dates use one format everywhere. Transactions filter by customer name. Login is a split layout.
- The alert queue no longer reads `?status=`; the view tabs replace it, and `?severity=` still works.
- Checked in the browser at 1024 px and 1440 px (no horizontal overflow) and at 800 px, where the sidebar becomes a menu drawer.

### Phase 3 — Screening (M7) · ~2 weeks

| PR | Scope |
|---|---|
| 3.1 | Screening domain per PRD §31: request, provider port, match results, match review, decision; all audited |
| 3.2 | Local provider adapter backed by a loaded sanctions/PEP list file (no vendor lock-in; real vendors later) |
| 3.3 | Screen on customer create/update; screening hits feed the risk engine |
| 3.4 | Console screening queue and match review |

### Phase 4 — Rule authoring, simulation, approval · ~2 weeks

| PR | Scope |
|---|---|
| 4.1 | Create new rule, suspend and retire versions |
| 4.2 | Maker-checker approval: drafts need approval by a different user before activation (PRD §48) |
| 4.3 | Rule simulation: run a draft against historical transactions, return would-be alerts and diff vs active version (PRD §23) |
| 4.4 | Console rule editor with schema validation, simulation results, approval queue |

### Phase 5 — Production hardening (M8) · ~2 weeks, can overlap Phases 3–4

| PR | Scope |
|---|---|
| 5.1 | OpenTelemetry traces and metrics wired to the existing packages; correlation ID through audit |
| 5.2 | Global rate limiting per tenant; request size limits; forwarded-headers config so the login limiter partitions by real client IP behind a proxy |
| 5.3 | Refresh tokens and logout (PRD §36) |
| 5.4 | CI for all three repos: build, test, lint, typecheck, dependency audit, container build |
| 5.5 | Load test of ingest + evaluate against the PRD §43 targets |
| 5.6 | Backup/restore runbook and tested restore |
| 5.7 | Security review pass (separate session) |

### Phase 6 — Research follow-ups (post-MVP, gated on PRD §67.1 decisions)

| PR | §67.1 item | Scope |
|---|---|---|
| 6.1 | 18 | Device/session signal ingestion; "shared attribute across N customers" rule type for rings and mule networks |
| 6.2 | 19 | Provider-neutral IDV outcome ingestion as risk-engine inputs |
| 6.3 | 20 | Re-verification request action delivered by webhook |
| 6.4 | 21 | Recommended action (approve / step-up / block) on rule results |
| 6.5 | 22 | Dormant-then-spike scenario for aged synthetic identities |
| 6.6 | 23 | Transaction initiator type and authorizing identity (Know Your Agent) |
| 6.7 | — | Console network graph view for linked customers |

### Parallel track — Marketing site · ~1 week, any time

| PR | Scope |
|---|---|
| W.1 | Wire the demo form to a real endpoint (Firebase function or form service), with spam protection and success/error states |
| W.2 | SEO: `metadataBase`, per-page OpenGraph/Twitter, canonical URLs, `sitemap.ts`, `robots.ts`, Organization JSON-LD |
| W.3 | Accessibility: carousel pause and reduced-motion, `focus-visible` styles, labels; Lighthouse ≥ 95 accessibility |
| W.4 | Real Privacy, Terms, Security pages; remove or attribute testimonials and unsourced stats |
| W.5 | Self-host the font via `next/font`, remove unused components and SVGs, rewrite README |
| W.6 | Add a KYC-maturity self-assessment page based on the research checklist (lead magnet), framed in Aegis terms |

### Parallel track — Docs

- Update the console plan to reflect what shipped; tick completed items.
- After each phase, update the README document list and this roadmap's status table.

---

## 4. Decisions needed from the owner

1. Phase order: is risk (Phase 2) before screening (Phase 3) right for the first pilot institution?
2. RBAC role set for 1.2: confirm `Admin`, `Analyst`, `Reviewer`, `Viewer`.
3. Demo form destination for W.1 (Firebase function writing to Firestore, or an external form service).
4. Screening data source for 3.2 (which public sanctions lists to load first: UN, OFAC, EU, Kenya FRC).
5. The PRD §67 open product decisions that block thresholds and SLAs (items 3–8).
