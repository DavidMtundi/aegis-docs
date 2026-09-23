# Vertical Slice Implementation Plan (PR1–PR5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the approved vertical slice — tenant-scoped ingest → feature calculation → configurable structuring rule → explainable alert → immutable audit — via five small PRs on `aegis`.

**Architecture:** Modular monolith (.NET 9). Thin `Aegis.Api` → `Aegis.Application` (`IngestAndEvaluateStructuring`) → module contracts. Single `AegisDbContext` with per-module PostgreSQL schemas. Infrastructure is the only EF accessor. Design baseline: `aegis-docs/designs/2026-09-23-vertical-slice-ingest-structuring-alert-audit.md`.

**Tech Stack:** .NET 9, ASP.NET Core, EF Core 9 + Npgsql, PostgreSQL 16, Redis (compose present; unused in slice), xUnit, Testcontainers or docker-compose Postgres for integration/E2E.

## Global Constraints

- Follow the **APPROVED DESIGN BASELINE** exactly; do not reopen architecture.
- `TenantId` / `ActorId` from `ITenantContext` only — never from trusted client body fields.
- Money: `decimal` / Postgres `numeric`; no floats; **KES-only** this slice.
- Feature windows use business **`Timestamp`**, not `CreatedAt`.
- Reject materially future-dated transaction timestamps.
- Dedupe lives in **Alerts**, not AML.
- Seed rule into **persisted** `aml.rules` / `aml.rule_versions`; generic evaluator only.
- Atomic commit for new ingest: transaction + `TRANSACTION_INGESTED` + alert (if new) + `ALERT_CREATED` (if new).
- Bulk: **one** use-case execution / DB transaction **per row**.
- `POST /api/v1/tenants` is **dev-only**.
- Packages: do not add npm/pypi/crates younger than 14 days without explicit override (prefer existing EF/Jwt/Serilog versions already in repo).

## Explicit Non-Goals (do not implement)

- KYC, KYB, Screening, Risk models, Cases, Network, Reporting, AI
- Rule CRUD / simulation UI or compliance rule-management APIs
- Event bus, outbox, Kafka, T+1 batch jobs
- Multi-currency / FX
- Auditing every non-triggered evaluation
- Production IdP, pen-test, full observability stack, DR
- Expanding beyond structuring scenario
- Putting `ITransactionReadPort` under Infrastructure
- Hardcoding structuring thresholds in C#

---

## Target solution layout (new / touched)

```text
aegis/
  src/
    Aegis.Api/                          # controllers, JWT middleware, DI, Program.cs
    Aegis.Application/                  # NEW — IngestAndEvaluateStructuring
    Aegis.Shared/                       # ITenantContext, Result, ids, Money
    Aegis.Infrastructure/               # AegisDbContext, configs, repos, JWT issuer helpers
    Aegis.Modules.Identity/             # Tenant, User, Role, Permission
    Aegis.Modules.Customers/            # NEW — Customer, Account
    Aegis.Modules.Transactions/         # CanonicalTransaction + ITransactionReadPort
    Aegis.Modules.Features/             # IFeatureCalculator
    Aegis.Modules.Aml/                  # existing engine + rule persistence ports
    Aegis.Modules.Alerts/               # Alert + dedupe service
    Aegis.Modules.Audit/                # IAuditWriter + AuditEvent
  tests/
    Unit/                               # domain/engine tests
    Integration/                        # repos + atomic pipeline pieces
    EndToEnd/                           # PR5 scenarios
```

Untouched module shells: Kyc, Kyb, Screening, Risk, Cases, Network, Reporting (leave `Class1.cs`).

---

## PR 1 — Platform foundation

**Exit:** Authenticated user operates inside a tenant; audit event can be written; migrations apply against docker-compose Postgres.

### Interfaces to introduce

```csharp
// Aegis.Shared/Security/ITenantContext.cs
public interface ITenantContext
{
    TenantId TenantId { get; }
    Guid UserId { get; }
    IReadOnlyCollection<string> Roles { get; }
}

// Aegis.Modules.Audit/Application/IAuditWriter.cs
public interface IAuditWriter
{
    Task AppendAsync(AuditEvent auditEvent, CancellationToken ct = default);
}

// Aegis.Modules.Identity/Application/IUserRepository.cs (pattern for all repos)
Task<User?> GetByTenantAndIdAsync(TenantId tenantId, Guid id, CancellationToken ct);
Task<User?> GetByTenantAndEmailAsync(TenantId tenantId, string email, CancellationToken ct);
```

### Files

- Create: `src/Aegis.Shared/Security/ITenantContext.cs`, `TenantContext.cs`
- Create: `src/Aegis.Modules.Identity/Domain/User.cs`, `Role.cs`, `Permission.cs`
- Create: `src/Aegis.Modules.Identity/Application/*Repository*.cs` interfaces
- Create: `src/Aegis.Modules.Audit/Application/IAuditWriter.cs`
- Create: `src/Aegis.Infrastructure/Persistence/AegisDbContext.cs`
- Create: `src/Aegis.Infrastructure/Persistence/Configurations/Identity/*.cs`, `Audit/*.cs`
- Create: `src/Aegis.Infrastructure/Persistence/Repositories/*.cs`
- Create: `src/Aegis.Infrastructure/Auth/JwtTokenService.cs`, `PasswordHasher` wrapper
- Create: `src/Aegis.Api/Controllers/AuthController.cs`
- Create: `src/Aegis.Api/Controllers/DevTenantsController.cs` (env-gated)
- Create: `src/Aegis.Api/Middleware/TenantContextMiddleware.cs` (or claims principal factory)
- Modify: `src/Aegis.Api/Program.cs` — register DbContext, auth, modules
- Modify: `src/Aegis.Api/appsettings.json` — connection string, Jwt signing key (dev)
- Create: EF migration under `src/Aegis.Infrastructure/Persistence/Migrations/` (or `migrations/` at repo root per README — pick **Infrastructure** and document)
- Test: `tests/Integration/Identity/AuthAndAuditTests.cs`

### Schema (identity.*, audit.*)

```text
identity.tenants
identity.users (tenant_id, email unique per tenant, password_hash, status)
identity.roles
identity.permissions
identity.user_roles
identity.role_permissions

audit.audit_events (immutable; no update/delete APIs)
```

### Tasks

- [ ] **PR1.1** Add `ITenantContext` + ambient accessor registered scoped from JWT claims (`tenant_id`, `sub`, roles).
- [ ] **PR1.2** Implement `User` / `Role` / `Permission` domain + tenant-aware repository interfaces.
- [ ] **PR1.3** Implement `AegisDbContext` with `HasDefaultSchema` per entity config (`identity`, `audit`).
- [ ] **PR1.4** Implement `IAuditWriter` as insert-only; reject updates in configuration (`IsReadOnly` / no Update methods exposed).
- [ ] **PR1.5** Local JWT login: `POST /api/v1/auth/login` `{ email, password, tenantSlug }` → access token. Hash passwords with ASP.NET Identity `PasswordHasher<T>` or equivalent — never store plaintext.
- [ ] **PR1.6** `POST /api/v1/tenants` only when `Environment.IsDevelopment()` **or** `Aegis:AllowDevBootstrap=true`; return 404/403 otherwise. Creates tenant + admin user.
- [ ] **PR1.7** Integration test: bootstrap (dev) → login → append audit → read back by tenant; second tenant cannot read first tenant’s audit by id.
- [ ] **PR1.8** `dotnet ef migrations add InitialPlatform` + `docker-compose up -d` + apply migration.
- [ ] **PR1.9** Commit on branch `feat/pr1-platform-foundation`.

### Acceptance criteria

- [ ] Login returns JWT with `tenant_id` and `sub`.
- [ ] Authenticated call resolves `ITenantContext`.
- [ ] Audit append + tenant-scoped read works.
- [ ] Dev tenant endpoint blocked when not in allowed mode.
- [ ] CI `dotnet test` green for new integration tests (Postgres service already in workflow).

### PR1 non-goals

No customers, transactions, rules, alerts, Application project yet (optional stub OK but unused).

---

## PR 2 — Customer / account / transaction

**Exit:** Tenant can create customer + account, ingest/query transactions with idempotency; amounts are `numeric`; future-dated timestamps rejected.

### Interfaces

```csharp
// Aegis.Modules.Customers
public interface ICustomerRepository { /* GetByTenantAndId, Add */ }
public interface IAccountRepository { /* GetByTenantAndId, Add; enforce customer.tenant_id match */ }

// Aegis.Modules.Transactions/Application/ITransactionRepository.cs
Task<CanonicalTransaction?> GetByTenantAndExternalReferenceAsync(TenantId tenantId, string externalReference, CancellationToken ct);
Task AddAsync(CanonicalTransaction tx, CancellationToken ct);

// Aegis.Modules.Transactions/Application/ITransactionReadPort.cs  // OWNED BY TRANSACTIONS
public interface ITransactionReadPort
{
    Task<IReadOnlyList<CanonicalTransaction>> GetForCustomerWindowAsync(
        TenantId tenantId,
        CustomerId customerId,
        DateTimeOffset windowStartInclusive,
        DateTimeOffset windowEndExclusive,
        CancellationToken ct = default);
}
```

Extend `CanonicalTransaction` with: `ExternalReference`, ensure `CustomerId`/`AccountId` required for slice, keep `Money` as decimal.

### Files

- Create: `src/Aegis.Modules.Customers/` project + add to `Aegis.sln` + Api/Infrastructure references
- Create: Domain `Customer`, `IndividualProfile` (or fields on customer), `Account`
- Modify: `CanonicalTransaction` — add `ExternalReference`; factory requires it
- Create: EF configs under `customers.*`, `transactions.*`
- Unique index: `(tenant_id, external_reference)` on transactions
- Create: Controllers `CustomersController`, `AccountsController`, `TransactionsController` (ingest persistence only in PR2 — **or** thin persist API; orchestration arrives in PR3/Application — prefer **persist-only ingest** in PR2 without AML, then PR3+ wires Application)
- Test: idempotency, tenant isolation, future timestamp rejection, money round-trip

**Recommendation:** PR2 exposes `POST /transactions` as **persist-only** (`IngestTransaction`) returning `{ transactionId, wasCreated }`. PR3 introduces `Aegis.Application` and switches the endpoint to `IngestAndEvaluateStructuring` (same route). Document the switch in the PR3 description so reviewers expect it.

### Tasks

- [ ] **PR2.1** Create `Aegis.Modules.Customers` csproj; wire solution + references.
- [ ] **PR2.2** Customer + Account aggregates; tenant-aware repos; FK/validation that account.customer belongs to same tenant.
- [ ] **PR2.3** Extend transaction model + `ITransactionRepository` + `ITransactionReadPort` interface in Transactions module; implement both in Infrastructure.
- [ ] **PR2.4** Migration for `customers.*`, `transactions.*`.
- [ ] **PR2.5** APIs: customers, accounts, persist-only transaction ingest + get.
- [ ] **PR2.6** Validation: currency must be `KES` for slice; timestamp not materially future (e.g. skew ≤ 5 minutes).
- [ ] **PR2.7** Tests: duplicate externalReference → same id, `wasCreated=false`; Tenant B cannot read Tenant A tx.
- [ ] **PR2.8** Commit `feat/pr2-customers-transactions`.

### Acceptance criteria

- [ ] Full customer → account → transaction happy path under JWT.
- [ ] Idempotent ingest.
- [ ] `numeric`/decimal fidelity for amounts (e.g. `95000.00`).

### PR2 non-goals

No features, rules, alerts, Application orchestration.

---

## PR 3 — Structuring detection

**Exit:** Seeded `STRUCTURING_001` v1 in DB evaluates via generic engine; features computed for customer 24h window on business timestamps.

### Interfaces

```csharp
// Aegis.Modules.Features/Application/IFeatureCalculator.cs
public interface IFeatureCalculator
{
    Task<IFeatureContext> CalculateAsync(
        TenantId tenantId,
        FocusType focusType,
        string focusEntityId,
        DateTimeOffset asOfTimestamp,
        TimeSpan window,
        CancellationToken ct = default);
}

// Aegis.Application/Transactions/IngestAndEvaluateStructuring.cs
public sealed record IngestAndEvaluateStructuringCommand(
    TenantId TenantId,
    Guid ActorId,
    IngestTransactionPayload Payload,
    string CorrelationId);

public sealed record IngestAndEvaluateStructuringResult(
    Guid TransactionId,
    bool WasCreated,
    IReadOnlyList<RuleEvaluationResult> Evaluations,
    IReadOnlyList<Guid> AlertIds);

public interface IIngestAndEvaluateStructuring
{
    Task<IngestAndEvaluateStructuringResult> ExecuteAsync(
        IngestAndEvaluateStructuringCommand command,
        CancellationToken ct = default);
}
```

Structuring seed JSON (persist definition on `AmlRuleVersion`):

```json
{
  "code": "STRUCTURING_001",
  "name": "Structuring",
  "focus": "CUSTOMER",
  "schedule": { "frequency": "realtime", "lookback": "24h" },
  "conditions": {
    "all": [
      { "field": "transaction_count_24h", "operator": "gte", "value": 5 },
      { "field": "transaction_sum_24h", "operator": "gte", "value": 450000 },
      { "field": "max_single_amount_24h", "operator": "lt", "value": 100000 }
    ]
  },
  "severity": "HIGH",
  "riskScore": 80,
  "actions": ["CREATE_ALERT"]
}
```

Align field names with what `ConditionEvaluator` already supports; adapt seed or evaluator mapping in the smallest change that keeps JSON-driven evaluation (extend evaluator feature lookup if needed — **no hardcoded structuring service**).

### Files

- Create: `src/Aegis.Application/` + solution entry
- Create: Features calculator implementation using `ITransactionReadPort`
- Create: AML rule repository + seed on startup or migration data seed
- Modify: `POST /transactions` → Application use case (Alert creation stubbed until PR4: if PR3 lands before alerts, return evaluations only and leave `AlertIds` empty — **prefer landing PR3+PR4 closely**; if split, PR3 returns evaluations without persisting alerts)
- Unit tests: feature math; engine `conditions.all` AND semantics with DictionaryFeatureContext

**Preferred split:** PR3 = features + seed + evaluate inside use case **without** alert persistence; PR4 adds alert+dedupe+atomic audit completion. Alternatively combine PR3+PR4 if review load allows — design assumes separate PRs.

### Tasks

- [ ] **PR3.1** Add `Aegis.Application` project; reference Modules (not Infrastructure).
- [ ] **PR3.2** Implement `IFeatureCalculator` for CUSTOMER focus; window `[asOf-24h, asOf)`; features `transaction_count_24h`, `transaction_sum_24h`, `max_single_amount_24h` + tx id list in context metadata.
- [ ] **PR3.3** Persist seed rule/version ACTIVE for system/demo tenant or global template copied per tenant on bootstrap — **per-tenant row** preferred (each tenant gets own rule version ids).
- [ ] **PR3.4** Wire `IngestAndEvaluateStructuring`: on `WasCreated`, calculate features with `asOf = tx.Timestamp`, evaluate active STRUCTURING rules with explicit `FocusType.CUSTOMER`.
- [ ] **PR3.5** Unit tests: 7×95k features; 4×95k does not satisfy count; boundary 5×90k; prove `all` AND via generic evaluator.
- [ ] **PR3.6** Commit `feat/pr3-structuring-detection`.

### Acceptance criteria

- [ ] No C# `if (count >= 5 && sum >= ...)` structuring shortcut.
- [ ] Seed rows visible in `aml.rules` / `aml.rule_versions`.
- [ ] Evaluator returns triggered/not with evidence feature values.

### PR3 non-goals

Alert persistence/dedupe (PR4); E2E suite (PR5); rule CRUD APIs.

---

## PR 4 — Alerting

**Exit:** Triggered results become explainable alerts; dedupe by key including `ruleVersionId` + UTC day; atomic commit with audits.

### Interfaces

```csharp
// Aegis.Modules.Alerts/Application/IAlertService.cs
public interface IAlertService
{
    /// <summary>
    /// AML may return TRIGGERED many times; this method decides create vs return existing.
    /// </summary>
    Task<AlertUpsertResult> CreateOrGetAsync(
        TenantId tenantId,
        RuleEvaluationResult triggered,
        AlertEvidence evidence,
        DateTimeOffset utcNow,
        CancellationToken ct = default);
}

public sealed record AlertUpsertResult(Guid AlertId, bool WasCreated);
```

Dedupe key format:

```text
{tenantId:D}:{ruleId:D}:{ruleVersionId:D}:{focusType}:{focusEntityId}:{yyyy-MM-dd}
```

UTC calendar date from `utcNow` (or from triggering tx timestamp — **lock to business timestamp date of the triggering transaction** for stability; document choice in code: prefer **`tx.Timestamp` UTC date** so ingest delay doesn’t shift bucket).

### Atomic unit of work

Use single EF `SaveChangesAsync` / explicit transaction scope in Application:

```text
BEGIN
  insert transaction (if new)
  insert TRANSACTION_INGESTED
  insert alert (if new)
  insert ALERT_CREATED (if new alert)
COMMIT
```

Feature/rule computation in-memory before commit.

### Tasks

- [ ] **PR4.1** Map `RuleEvaluationResult` → `AlertEvidence` (include thresholds/operators in `AdditionalContext` or extend evidence record minimally).
- [ ] **PR4.2** Implement dedupe in Alerts module; unique index on `deduplication_key` per tenant.
- [ ] **PR4.3** Complete `IngestAndEvaluateStructuring` atomic writes + audits.
- [ ] **PR4.4** `GET /api/v1/alerts`, `GET /api/v1/alerts/{id}`, `GET /api/v1/audit-events`.
- [ ] **PR4.5** Integration test: triggered path creates one alert; second ingest same day same version dedupes; new rule version can create another.
- [ ] **PR4.6** Commit `feat/pr4-alerting`.

### Acceptance criteria

- [ ] Alert contains `RuleVersionId` + explainable facts + transaction ids.
- [ ] Failure mid-write rolls back transaction+alert+audits together (simulate by forcing audit failure in test if feasible).
- [ ] Dedupe not implemented inside `RuleEvaluationEngine`.

### PR4 non-goals

Case creation, assign/dismiss APIs beyond what’s needed to read alerts, notification channels.

---

## PR 5 — E2E vertical slice tests

**Exit:** Automated proof of the design assumption + isolation/idempotency/negative/boundary.

### Files

- Create: `tests/EndToEnd/VerticalSlice/StructuringSliceTests.cs`
- Shared fixture: WebApplicationFactory or API + Testcontainers Postgres
- Seed helper: tenant A/B, users, rule seed

### Scenarios (must automate)

- [ ] **PR5.1 Positive:** 7×95,000 KES → features 7 / 665000 / 95000 → one alert → audits `TRANSACTION_INGESTED`×7 + `ALERT_CREATED`≥1 → evidence rule version + tx ids.
- [ ] **PR5.2 Negative:** 4×95,000 → 0 alerts.
- [ ] **PR5.3 Boundary:** 5×90,000 → triggers.
- [ ] **PR5.4 Duplicate:** re-POST same `externalReference` → `wasCreated=false`, alert count unchanged, no extra ingest audit.
- [ ] **PR5.5 Cross-tenant:** Tenant B cannot GET Tenant A alert/customer/transaction (404/403).
- [ ] **PR5.6 Optional:** future-dated timestamp → 400; amount decimal round-trip.
- [ ] **PR5.7** Commit `feat/pr5-e2e-vertical-slice` and open PR summarizing the slice.

### Acceptance criteria

- [ ] All scenarios green in CI.
- [ ] README snippet: how to run compose + migrate + e2e locally.

---

## Suggested branch / PR hygiene

```text
main
 └── feat/pr1-platform-foundation        → merge
      └── feat/pr2-customers-transactions → merge
           └── feat/pr3-structuring-detection → merge
                └── feat/pr4-alerting → merge
                     └── feat/pr5-e2e-vertical-slice → merge
```

Each PR description must list **non-goals** from this plan and link the design baseline.

---

## Spec coverage checklist

| Design requirement | Plan location |
|--------------------|---------------|
| Tenant context / JWT | PR1 |
| Dev-only tenant bootstrap | PR1.6 |
| Customers + Account | PR2 |
| Canonical tx + idempotency | PR2 |
| `ITransactionReadPort` owned by Transactions | PR2.3 |
| Features + 24h business timestamp window | PR3 |
| Persisted seeded JSON rule + generic engine | PR3 |
| `IngestAndEvaluateStructuring` | PR3–PR4 |
| Alert + alert-level dedupe + version in key | PR4 |
| Atomic tx/alert/audit | PR4.3 |
| Per-row bulk boundary | PR4 (bulk controller loops use case) |
| E2E positive/negative/boundary/dupe/cross-tenant | PR5 |
| Non-goals / deferred modules | Global non-goals |

---

## Execution handoff

Plan complete. Two execution options:

1. **Subagent-Driven (recommended)** — fresh subagent per task group (PR1→PR5), review between PRs  
2. **Inline Execution** — execute in this session with checkpoints after each PR

Which approach?
