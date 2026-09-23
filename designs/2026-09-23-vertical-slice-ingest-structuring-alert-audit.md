# Vertical Slice Design: Ingest → Structuring → Alert → Audit

**Status:** Draft for review  
**Date:** 2026-09-23  
**Product:** Aegis AML Compliance Platform  
**Repositories:** `aegis` (implementation), `aegis-docs` (this document)  
**Related:** [Product Requirements & Technical Architecture](./product-requirements-and-architecture.md)

---

## 1. Goal

Prove the core architectural assumption of Aegis:

> Can a tenant ingest financial data, evaluate a configurable AML policy, generate an explainable alert, and preserve an auditable trail end-to-end?

This is a **vertical-slice milestone**, delivered as a sequence of small PRs — not “build all of Sprint 1 in isolation.”

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

Do **not** architect these out: keep module shells, ports, and schemas so they plug in later.

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
Aegis.Api  (transport)
    ↓
Aegis.Application  (IngestAndEvaluateStructuring)
    ↓
Transactions → Features → Aml → Alerts → Audit
    ↓
AegisDbContext / PostgreSQL
```

- Controllers do not contain AML semantics or multi-module orchestration.
- The same use case must be invokable later by CSV, Kafka, T+1 batch, or replay without duplicating the workflow.

### 4.4 Synchronous pipeline for the slice; events later

No event bus in this slice. Keep the path synchronous and deterministic. Domain events / outbox may replace the *caller* later without changing module contracts.

### 4.5 Dependency direction (via contracts)

```text
Transactions  (owns canonical tx + ITransactionReadPort)
      ↑
   Features   (IFeatureCalculator; consumes read port)
      ↑
     Aml      (IRuleEvaluationEngine; consumes IFeatureContext only)
      ↓
   Alerts     (consumes RuleEvaluationResult only)
      ↓
    Audit     (IAuditWriter; immutable append)
```

---

## 5. Five core contracts

### 5.1 Tenant context

- Local JWT issuer in the API for the slice (claim shape compatible with a future external IdP).
- Claims include at least: `sub` (user id), `tenant_id`, roles.
- Middleware builds `ITenantContext { TenantId, UserId, Roles }`.
- Every repository/query requires tenant: `GetByTenantAndId(tenantId, id)`, `GetByTenant(...)`.
- Prefer DB constraints / FKs that make cross-tenant references invalid or explicitly validated (e.g. transaction’s `tenant_id` must match related customer/account tenant).

### 5.2 Canonical Transaction

- Extend existing `CanonicalTransaction` in `Transactions`.
- Required fields for the slice: `TenantId`, `ExternalReference` (idempotency), `AccountId`, `CustomerId`, `Amount` + currency, `Direction`, `TransactionType`, `Channel`, `Timestamp`, `Status`, optional country/counterparty/metadata.
- Transactions **reference** `CustomerId` / `AccountId`; they do **not** own Customer/Account aggregates.
- Idempotency key: `(tenant_id, external_reference)` unique. Duplicate ingest returns the existing transaction and **does not** recalculate features, evaluate rules, or create alerts.

### 5.3 Rule contract

```text
(Active AmlRuleVersion, IFeatureContext, FocusType, FocusId)
        → RuleEvaluationResult
```

- Focus is explicit: for structuring in this slice, `FocusType = CUSTOMER`, `FocusId = customerId`. The engine must not infer focus from opaque IDs.
- Rules consume **features only**, never raw transaction tables.
- Reuse existing `IRuleEvaluationEngine` / `RuleEvaluationResult` / evidence types.
- **Seeded structuring rule must use the same JSON rule definition and evaluator as future compliance-authored rules.** No special-case `if (count >= 5 && ...)` service in C#.

### 5.4 Alert contract

- Alerts are created only from a **triggered** `RuleEvaluationResult`.
- Persist: `RuleId`, `RuleVersionId`, focus, severity, evidence JSON, `DeduplicationKey`.
- Evidence must include: rule code, version number / version id, facts (feature, value, threshold, operator), and contributing transaction IDs.
- Status for slice: start at `OPEN` (maps to PRD “NEW” for MVP).

**Deduplication**

- Evaluation window (AML): **rolling 24 hours**.
- Dedupe bucket (alert suppression): **UTC calendar day** — explicitly *not* the same concept as the evaluation window.
- Deduplication key:

```text
{tenantId}:{ruleId}:{ruleVersionId}:{focusType}:{focusEntityId}:{utcCalendarDay}
```

Including `ruleVersionId` so activating a new version is not suppressed by an alert from a prior version in the same day.

- On key collision: return existing alert; do not create another.

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

- Explainability for detection lives primarily on **alert evidence**. Optional `RULE_EVALUATED` audit is out of scope for v1.
- Evaluation telemetry (non-trigger) belongs in logs/metrics, not the audit table, unless a later compliance requirement demands otherwise.

---

## 6. Module ownership

| Concern | Module | Notes |
|---------|--------|--------|
| Tenant, User, Role, Permission, login/JWT | `Identity` | Extend existing `Tenant` |
| Customer, Individual/Business (minimal), Account | `Customers` (new) | Account stays here for the slice |
| Canonical transaction, ingest, `ITransactionReadPort` | `Transactions` | |
| Feature calculation | `Features` | Owns `IFeatureCalculator` |
| Rules, versions, evaluation | `Aml` | Seed `STRUCTURING_001` via same JSON schema |
| Alert lifecycle | `Alerts` | |
| Immutable audit | `Audit` | |
| EF, repos, port implementations | `Infrastructure` | |
| Cross-module workflow | `Application` | Thin; not a dumping ground |
| HTTP | `Api` | Thin controllers |

Empty shells (KYC, Screening, Cases, Risk, …) remain untouched.

---

## 7. Slice data flow

### 7.1 New successful ingest

```text
POST /api/v1/transactions
  → validate
  → idempotency check
  → persist NEW CanonicalTransaction
  → audit TRANSACTION_INGESTED
  → Features.Calculate(CUSTOMER, customerId, window=24h)
  → Aml.Evaluate(active structuring version, features, focus)
  → if triggered: dedupe → Alert.Create → audit ALERT_CREATED
  → return transaction + evaluation summary + alert id(s)
```

Invariant:

> A transaction is evaluated only when it is newly accepted into the system.

### 7.2 Duplicate ingest

```text
existing transaction
  → return existing
  → NO feature recalculation
  → NO rule evaluation
  → NO new alert
```

Later T+1 / replay may re-evaluate deliberately without changing this ingestion contract.

### 7.3 Feature set (structuring v1)

Customer-scoped, rolling 24h:

```text
transaction_count_24h
transaction_sum_24h
max_single_amount_24h
(+ transaction ids for evidence)
```

Aggregation spans **all accounts** belonging to the customer.

### 7.4 Seeded rule (`STRUCTURING_001` v1)

Configurable JSON (illustrative thresholds for demo/tests):

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
| `POST` | `/api/v1/transactions` | Ingest one + run orchestration |
| `POST` | `/api/v1/transactions/bulk` | Ingest many; orchestrate per **new** row |
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
| Alerts | How a triggered result becomes a persisted, explainable alert |
| Audit | How immutable compliance history is recorded |

**The API does not own AML semantics.**

---

## 10. Testing requirements (PR 5)

### Positive

- Tenant A → Customer A → Account A → **7 × 95,000 KES** credits within 24h  
- Expect: 7 transactions; features count=7, sum=665,000, max=95,000; all three conditions true; **exactly one** alert for the dedupe identity; evidence has rule/version/facts/tx ids; ingest audits + `ALERT_CREATED`; Tenant B cannot read Tenant A’s alert.

### Negative

- **4 × 95,000 = 380,000** → **0 alerts** (count < 5 and/or sum < 450,000 depending on operators — with seeded rule, count fails).

### Boundary

- **5 × 90,000 = 450,000** → **triggers** (`count >= 5`, `sum >= 450000`, `max < 100000`).

### Duplicate

- Re-POST same `externalReference` → same transaction id; no second evaluation; no second alert.

### Cross-tenant

- Tenant B token cannot `GET` Tenant A resources (alert, customer, transaction).

---

## 11. Non-goals for implementation quality bar

This slice must be production-*shaped* (tenant isolation, idempotency, explainability, audit, tests) but not production-*complete* (no pen-test, no full observability stack, no DR). Hardening is a later milestone per the PRD.

---

## 12. Open items intentionally left to Product/Compliance (not blocked for coding)

Threshold values in the seeded rule are **demo defaults** for tests. Real institutional thresholds remain open decisions in the PRD (§67). Engineering must not hardcode regulatory policy into C#; only into configurable rule JSON (and later compliance-owned config).

---

## 13. Next step after approval

Once this design is reviewed and accepted:

1. Write a detailed implementation plan (PR-by-PR tasks) via the planning workflow.
2. Start **PR 1** on `aegis`.

Do not begin coding until this document is approved.

---

## Document history

| Date | Change |
|------|--------|
| 2026-09-23 | Initial vertical-slice design from brainstorming (Sections 1–3 locked with refinements) |
