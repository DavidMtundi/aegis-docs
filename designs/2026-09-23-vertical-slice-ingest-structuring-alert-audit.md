# Vertical Slice Design: Ingest → Structuring → Alert → Audit

**Status:** Design baseline candidate (awaiting final approval)  
**Date:** 2026-09-23  
**Product:** Aegis AML Compliance Platform  
**Repositories:** `aegis` (implementation), `aegis-docs` (this document)  
**Related:** [Product Requirements & Technical Architecture](./product-requirements-and-architecture.md)

---

## 1. Goal

Prove the core architectural assumption of Aegis:

> Can a tenant ingest financial data, evaluate a configurable AML policy, generate an explainable alert, and preserve an auditable trail end-to-end?

This is a **vertical-slice milestone**, delivered as a sequence of small PRs — not “build all of Sprint 1 in isolation.”

The first vertical slice proves the **architecture**, not merely a feature. It must demonstrate:

- tenant isolation
- canonical transaction ingestion
- idempotency
- feature abstraction
- configurable rules
- rule versioning
- explainability
- alert deduplication
- auditability
- cross-module orchestration

without prematurely building KYC, screening, cases, AI, or event infrastructure.

### Minimum demonstrable scenario

Tenant A creates a customer and account, ingests a realistic set of transactions, Aegis evaluates a **configurable** structuring rule, creates an **explainable** alert referencing the **exact rule version**, and records significant business actions in the **audit trail**.

---

## 2. Explicitly deferred (boundaries preserved)

Do **not** build in this slice:

- Full KYC / KYB
- Screening
- Sophisticated risk scoring
- All nine AML scenarios (only structuring)
- Case management UI / case workflow
- Regulatory reporting
- External banking integrations
- AI
- Advanced dashboarding
- Production-grade rule simulation
- Full rule CRUD / compliance UI for rule management
- Event bus / outbox (design for later; not implement now)
- Multi-currency / FX normalization
- Auditing every non-triggered rule evaluation

Do **not** architect these out: keep module shells, ports, and schemas so they plug in later.

The implementation plan must state **what not to implement** as explicitly as what to implement, so this slice does not quietly expand into half the platform.

---

## 3. Delivery: five independently reviewable PRs

| PR | Scope | Exit criteria |
|----|--------|----------------|
| **PR 1 — Platform foundation** | PostgreSQL, EF `AegisDbContext`, schemas, Tenant, User/RBAC (minimal), local JWT, tenant-aware repositories, `IAuditWriter` | Authenticated tenant-scoped call; audit event can be written |
| **PR 2 — Customer / account / transaction** | `Customers` module, Account inside Customers, canonical transaction + REST ingest + idempotency | Tenant can create customer/account and ingest/query transactions |
| **PR 3 — Structuring detection** | `IFeatureCalculator`, transaction read port, seeded generic JSON structuring rule, evaluator | Deterministic trigger + evidence for sample data |
| **PR 4 — Alerting** | Map `RuleEvaluationResult` → explainable `Alert` + dedupe + lifecycle shell (`OPEN`) | Alert readable with rule version + evidence |
| **PR 5 — E2E vertical slice** | Integration tests for ingest → evaluate → alert → audit (+ negative/boundary/duplicate/cross-tenant) | Green proof of the assumption |

---

## 4. Architecture decisions (locked)

### 4.1 Modular monolith, existing solution layout

Implement in `aegis` using existing `Aegis.Modules.*` projects. Add `Aegis.Modules.Customers` and a thin `Aegis.Application` for cross-module workflows.

### 4.2 Single DbContext, per-module PostgreSQL schemas

One `AegisDbContext` with schemas:

```text
identity.*
customers.*
transactions.*
features.*
aml.*
alerts.*
audit.*
```

Example tables:

```text
customers.customers
customers.accounts
transactions.transactions
aml.rules
aml.rule_versions
alerts.alerts
audit.audit_events
```

**Hard rule:** No module accesses another module’s `DbSet` directly. Cross-module data access goes through ports implemented in Infrastructure.

### 4.3 Application owns workflows; modules own capabilities

```text
Aegis.Api  (transport adapter only)
    ↓
Aegis.Application  (IngestAndEvaluateStructuring)
    ↓
module capabilities via contracts
    ↓
Infrastructure (port implementations)
    ↓
AegisDbContext / PostgreSQL
```

- Controllers do not contain AML semantics or multi-module orchestration.
- The same use case must be invokable later by CSV, Kafka, T+1 batch, or replay without duplicating the workflow.

### 4.4 Synchronous pipeline for the slice; events later

No event bus in this slice. Keep the path synchronous and deterministic. Domain events / outbox may replace the *caller* later without changing module contracts.

### 4.5 Contract consumption (not compile-time “DbSet” coupling)

Replace ambiguous dependency arrows with this meaning:

```text
Application orchestration (IngestAndEvaluateStructuring)
        │
        ├──────────────► Transactions
        │                    │
        │                    │ ITransactionReadPort
        │                    ▼
        ├──────────────► Features
        │                    │
        │                    │ IFeatureContext
        │                    ▼
        ├──────────────► Aml
        │                    │
        │                    │ RuleEvaluationResult
        │                    ▼
        ├──────────────► Alerts
        │
        └──────────────► Audit

Infrastructure
    implements persistence / read ports
          │
          ▼
   AegisDbContext
          │
          ▼
     PostgreSQL
```

**The arrows represent contract consumption / orchestration calls, not direct database access. Infrastructure is the only layer that accesses EF Core persistence.**

Modules must not reference each other’s persistence models. Application may reference module application services / ports; Infrastructure references modules as needed to map entities.

---

## 5. Core contracts

### 5.1 Tenant context

- Local JWT issuer in the API for the slice (claim shape compatible with a future external IdP).
- Claims include at least: `sub` (user id), `tenant_id`, roles.
- Middleware builds `ITenantContext { TenantId, UserId, Roles }`.
- Every repository/query requires tenant: `GetByTenantAndId(tenantId, id)`, `GetByTenant(...)`.
- Prefer DB constraints / FKs that make cross-tenant references invalid or explicitly validated (e.g. transaction’s `tenant_id` must match related customer/account tenant).

### 5.2 Canonical Transaction

- Extend existing `CanonicalTransaction` in `Transactions`.
- Required fields for the slice: `TenantId`, `ExternalReference` (idempotency), `AccountId`, `CustomerId`, `Amount` + currency, `Direction`, `TransactionType`, `Channel`, **business/event `Timestamp`**, `Status`, optional country/counterparty/metadata.
- Also record `CreatedAt` (ingestion/persistence time) separately from business `Timestamp`.
- Transactions **reference** `CustomerId` / `AccountId`; they do **not** own Customer/Account aggregates.
- Idempotency key: `(tenant_id, external_reference)` unique.

#### Money representation

- API may accept `"amount": 95000, "currency": "KES"`.
- Domain and persistence must **not** use floating-point for money.
- Use `decimal` / .NET decimal (or equivalent) in domain; PostgreSQL `numeric` for amounts.
- Currency is an ISO-4217 code string.
- Rule threshold values (e.g. `450000`) are interpreted in the **same currency as the feature** being compared.
- **This slice operates on a single currency (KES).** Multi-currency normalization / FX is deferred.

#### Timestamp semantics for features

> Feature windows are calculated using the transaction’s **business/event timestamp**, not the database insertion timestamp (`CreatedAt`).

Example: event time `2026-09-23T10:00:00Z`, ingested at `10:04:32Z` → windows use **10:00**.

For this slice:

> Transaction timestamps must not be **materially in the future** relative to server UTC at ingest time. Reject such requests with a validation error. Exact skew tolerance is an implementation detail for the slice (recommend a small fixed tolerance such as a few minutes), not a compliance product decision.

### 5.3 Rule contract

```text
(Active AmlRuleVersion, IFeatureContext, FocusType, FocusId)
        → RuleEvaluationResult
```

- Focus is explicit: for structuring in this slice, `FocusType = CUSTOMER`, `FocusId = customerId`. The engine must not infer focus from opaque IDs.
- Rules consume **features only**, never raw transaction tables.
- Reuse existing `IRuleEvaluationEngine` / `RuleEvaluationResult` / evidence types.
- **Seeded structuring rule must use the same JSON rule definition and evaluator as future compliance-authored rules.** No special-case `if (count >= 5 && ...)` service in C#.
- **The seed mechanism must populate the same persisted `Rule` / `RuleVersion` model used by future compliance configuration; it must not bypass rule persistence by embedding the rule definition only in application code or in-memory-only objects.** Flow:

```text
seed → aml.rules + aml.rule_versions → generic evaluator
```

### 5.4 Alert contract

- Alerts are created only from a **triggered** `RuleEvaluationResult`.
- Persist: `RuleId`, `RuleVersionId`, focus, severity, evidence JSON, `DeduplicationKey`.
- Evidence must include: rule code, version number / version id, facts (feature, value, threshold, operator), and contributing transaction IDs.
- Status for slice: start at `OPEN` (maps to PRD “NEW” for MVP).

#### Deduplication (alert-level only)

- Evaluation window (AML): **rolling 24 hours** (based on business timestamps).
- Dedupe bucket (alert suppression): **UTC calendar day** — explicitly *not* the same concept as the evaluation window.
- Deduplication key:

```text
{tenantId}:{ruleId}:{ruleVersionId}:{focusType}:{focusEntityId}:{utcCalendarDay}
```

Including `ruleVersionId` so activating a new version is not suppressed by an alert from a prior version in the same day.

- On key collision: return existing alert; do not create another.

**Dedupe is alert-level behavior, not rule-evaluation behavior.**

```text
AML Engine  →  “Did the rule trigger?”
Alerts      →  “Should this trigger produce a new alert?”
```

The engine may return `TRIGGERED` multiple times for the same focus/day. The Alerts module applies dedupe. Do **not** put deduplication inside the AML evaluator.

### 5.5 Audit contract

- Immutable append via `IAuditWriter` / `AuditEvent` (WHO, WHAT, WHEN, BEFORE, AFTER, REASON, CorrelationId).
- **Audit significant business actions**, not every non-triggered rule evaluation (noise/scale).
- Minimum for the slice:

```text
CUSTOMER_CREATED
ACCOUNT_CREATED
TRANSACTION_INGESTED
ALERT_CREATED
```

Separation of concerns:

```text
Audit          = what the system did
Alert evidence = why the system generated this alert
Logs/metrics   = how the system behaved
```

- Explainability for detection lives primarily on **alert evidence**. Optional `RULE_EVALUATED` audit is out of scope for v1.
- Evaluation telemetry (non-trigger) belongs in logs/metrics, not the audit table, unless a later compliance requirement demands otherwise.

#### Transactionality

For the successful synchronous pipeline:

> **Transaction persistence, alert persistence (when created), and corresponding audit writes must participate in the same database transaction where practical.**

At minimum, for a new ingest that produces an alert:

```text
CanonicalTransaction
Alert
TRANSACTION_INGESTED audit
ALERT_CREATED audit
```

must commit atomically. Do not leave an alert without its audit record because a later audit write failed outside the transaction.

Duplicate ingest paths that do not create work should not emit duplicate `TRANSACTION_INGESTED` audits.

Later, an outbox can handle asynchronous external publication; an event bus is not required to achieve atomicity for this slice.

### 5.6 Application contract: `IngestAndEvaluateStructuring`

Cross-module workflow owned by `Aegis.Application` (not by Api controllers, not by Transactions).

**Input**

```text
TenantId
ActorId
IngestTransactionCommand / CanonicalTransaction payload
CorrelationId
```

**Output**

```text
TransactionId
WasCreated          // false if idempotent duplicate
Evaluations[]       // empty when WasCreated == false
AlertIds[]          // empty when no new/existing-from-this-run alerts needed; may include existing id if triggered+deduped
```

**Behavior**

```text
NEW transaction
    → persist CanonicalTransaction
    → calculate features (business Timestamp window)
    → evaluate active structuring rule version(s)
    → for each TRIGGERED result: Alerts create-or-dedupe
    → audit TRANSACTION_INGESTED (+ ALERT_CREATED when a new alert row is inserted)
    → commit atomically with related writes
    → return result

DUPLICATE transaction (same tenant + externalReference)
    → return existing TransactionId
    → WasCreated = false
    → NO feature recalculation
    → NO rule evaluation
    → NO new alert
    → NO duplicate TRANSACTION_INGESTED audit
```

Invariant:

> A transaction is evaluated only when it is newly accepted into the system.

The API knows only: “invoke this use case.” It does not know AML semantics.

---

## 6. Module ownership

| Concern | Module | Notes |
|---------|--------|--------|
| Tenant, User, Role, Permission, login/JWT | `Identity` | Extend existing `Tenant` |
| Customer, Individual/Business (minimal), Account | `Customers` (new) | Account stays here for the slice |
| Canonical transaction, ingest, `ITransactionReadPort` | `Transactions` | |
| Feature calculation | `Features` | Owns `IFeatureCalculator` |
| Rules, versions, evaluation | `Aml` | Seed persists `STRUCTURING_001` into `aml.rules` / `aml.rule_versions` |
| Alert lifecycle + dedupe | `Alerts` | Dedupe lives here, not in AML |
| Immutable audit | `Audit` | |
| EF, repos, port implementations | `Infrastructure` | Only EF accessor |
| Cross-module workflow | `Application` | Thin; not a dumping ground |
| HTTP | `Api` | Thin controllers |

Empty shells (KYC, Screening, Cases, Risk, …) remain untouched.

---

## 7. Slice data flow

### 7.1 New successful ingest

```text
HTTP POST /api/v1/transactions
  → Api maps DTO → IngestAndEvaluateStructuring
  → validate (incl. not materially future-dated Timestamp)
  → idempotency check
  → persist NEW CanonicalTransaction
  → Features.Calculate(CUSTOMER, customerId, window=24h from business Timestamp)
  → Aml.Evaluate(active structuring version, features, FocusType=CUSTOMER, FocusId)
  → if TRIGGERED: Alerts.CreateOrGetByDedupeKey → maybe new alert
  → audit business actions inside same DB transaction
  → return TransactionId, WasCreated=true, Evaluations, AlertIds
```

### 7.2 Duplicate ingest

```text
existing transaction
  → return existing
  → WasCreated=false
  → NO feature recalculation
  → NO rule evaluation
  → NO new alert
  → NO duplicate ingest audit
```

Later T+1 / replay may re-evaluate deliberately without changing this ingestion contract.

### 7.3 Feature set (structuring v1)

Customer-scoped, rolling 24h on **business timestamps**, KES only:

```text
transaction_count_24h
transaction_sum_24h
max_single_amount_24h
(+ transaction ids for evidence)
```

Aggregation spans **all accounts** belonging to the customer.

### 7.4 Seeded rule (`STRUCTURING_001` v1)

Persisted seed (illustrative thresholds for demo/tests):

```json
{
  "code": "STRUCTURING_001",
  "scenario": "STRUCTURING",
  "conditions": {
    "all": [
      { "field": "transaction_count_24h", "operator": "gte", "value": 5 },
      { "field": "transaction_sum_24h", "operator": "gte", "value": 450000 },
      { "field": "max_single_amount_24h", "operator": "lt", "value": 100000 }
    ]
  },
  "actions": [{ "type": "CREATE_ALERT", "severity": "HIGH" }]
}
```

No rule management APIs in this slice. Optional read-only `GET /api/v1/rules` may be added for demo convenience; not required for DoD.

---

## 8. HTTP API (slice)

All business APIs require JWT and operate inside `ITenantContext`.

| Method | Path | Behavior |
|--------|------|----------|
| `POST` | `/api/v1/auth/login` | Issue access token |
| `POST` | `/api/v1/tenants` | Bootstrap (dev/seed; tighten later) |
| `POST` | `/api/v1/customers` | Create customer |
| `GET` | `/api/v1/customers/{id}` | Get customer |
| `POST` | `/api/v1/customers/{id}/accounts` | Create account |
| `GET` | `/api/v1/accounts/{id}` | Get account |
| `POST` | `/api/v1/transactions` | Map to `IngestAndEvaluateStructuring` |
| `POST` | `/api/v1/transactions/bulk` | Call use case per **new** row |
| `GET` | `/api/v1/transactions/{id}` | Get transaction |
| `GET` | `/api/v1/alerts` | List (filters: status, customer) |
| `GET` | `/api/v1/alerts/{id}` | Detail + evidence |
| `GET` | `/api/v1/audit-events` | List (filters: type, entity) |

### Ingest body (canonical)

```json
{
  "externalReference": "TX-10001",
  "accountId": "...",
  "customerId": "...",
  "amount": 95000,
  "currency": "KES",
  "direction": "CREDIT",
  "transactionType": "TRANSFER",
  "channel": "MOBILE",
  "timestamp": "2026-09-23T10:00:00Z",
  "counterpartyCountry": "KE",
  "metadata": {}
}
```

### Alert evidence (stored)

```json
{
  "ruleCode": "STRUCTURING_001",
  "ruleVersion": 1,
  "ruleVersionId": "...",
  "facts": [
    { "feature": "transaction_count_24h", "value": 7, "threshold": 5, "operator": "gte" },
    { "feature": "transaction_sum_24h", "value": 665000, "threshold": 450000, "operator": "gte" },
    { "feature": "max_single_amount_24h", "value": 95000, "threshold": 100000, "operator": "lt" }
  ],
  "transactionIds": ["...", "..."]
}
```

---

## 9. Separation of concerns (non-negotiable)

| Layer | Knows |
|-------|--------|
| API | How to invoke an application use case over HTTP |
| Application | How to orchestrate ingest → features → rules → alerts → audit |
| Transactions | What a canonical transaction is; how to persist/query it |
| Features | How transaction facts are calculated for a focus + window |
| AML | How configurable JSON rules are evaluated against features |
| Alerts | How a triggered result becomes a persisted, explainable, deduped alert |
| Audit | How immutable compliance history is recorded |
| Infrastructure | How ports map to EF / PostgreSQL |

**The API does not own AML semantics.**

---

## 10. Testing requirements (PR 5)

### Positive

- Tenant A → Customer A → Account A → **7 × 95,000 KES** credits within 24h (business timestamps)  
- Expect: 7 transactions; features count=7, sum=665,000, max=95,000; all three conditions true; **exactly one** alert for the dedupe identity; evidence has rule/version/facts/tx ids; ingest audits + `ALERT_CREATED`; Tenant B cannot read Tenant A’s alert.

### Negative

- **4 × 95,000 = 380,000** → **0 alerts** (count &lt; 5 with seeded rule).

### Boundary

- **5 × 90,000 = 450,000** → **triggers** (`count >= 5`, `sum >= 450000`, `max < 100000`).

### Duplicate

- Re-POST same `externalReference` → same transaction id; `WasCreated=false`; no second evaluation; no second alert; no duplicate ingest audit.

### Cross-tenant

- Tenant B token cannot `GET` Tenant A resources (alert, customer, transaction).

### Optional hardening checks for the slice

- Materially future-dated `timestamp` is rejected.
- Amounts persist/query with decimal/`numeric` fidelity (no float drift).

---

## 11. Non-goals for implementation quality bar

This slice must be production-*shaped* (tenant isolation, idempotency, explainability, audit, atomic pipeline writes, tests) but not production-*complete* (no pen-test, no full observability stack, no DR). Hardening is a later milestone per the PRD.

---

## 12. Open items intentionally left to Product/Compliance (not blocked for coding)

Threshold values in the seeded rule are **demo defaults** for tests. Real institutional thresholds remain open decisions in the PRD (§67). Engineering must not hardcode regulatory policy into C#; only into configurable rule JSON (and later compliance-owned config).

---

## 13. Next step after approval

Once this design is accepted as the implementation contract:

1. Write a detailed implementation plan (PR-by-PR tasks, files, interfaces, migrations, tests, and explicit non-goals).
2. Start **PR 1** on `aegis`.

Do not begin coding until this document is approved as the baseline.

---

## Document history

| Date | Change |
|------|--------|
| 2026-09-23 | Initial vertical-slice design from brainstorming (Sections 1–3 locked with refinements) |
| 2026-09-23 | Review pass: contract-consumption diagram; `IngestAndEvaluateStructuring` I/O; business timestamp semantics; money/`numeric` + KES-only; atomic tx/alert/audit; alert-level dedupe; persisted rule seed |
