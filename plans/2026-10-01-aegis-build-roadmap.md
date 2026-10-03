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
| 1.10 | Shared UI kit: `DataTable`, `Field`, loading/empty/error states, `Pagination`; move every page onto it and onto `lib/api` |
| 1.11 | 401 redirects to login with return URL; role-aware actions (hide what the user can't do) |
| 1.12 | Readable evidence panel: feature / actual value / threshold / pass-fail table (PRD §26), raw JSON behind a toggle |
| 1.13 | Customer 360 page; transactions list page |
| 1.14 | Case timeline, escalate, link alerts, assign to another user |
| 1.15 | Audit viewer filters and paging; CSV import screen with per-row results |
| 1.16 | Admin users page |

**Exit:** every item in PRD §65 except risk and screening works from a clean environment, covered by the Playwright demo flow.

### Phase 2 — Risk engine and dashboard (M6) · ~2 weeks

| PR | Scope |
|---|---|
| 2.1 | Risk module: configurable, versioned risk model (factors and weights per tenant, PRD §30) |
| 2.2 | Customer risk score calculation on customer change, alert, and nightly batch; score history with factor breakdown |
| 2.3 | `GET /customers/{id}/risk` and risk in customer 360 |
| 2.4 | Dashboard metrics API: open alerts by severity and age, SLA breaches, cases by status, alerts per rule, dismissal rate per rule (false-positive proxy) |
| 2.5 | Console dashboard page (home route) |
| 2.6 | Console risk panel with factor breakdown and history |

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
