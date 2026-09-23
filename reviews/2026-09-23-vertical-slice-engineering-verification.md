# Aegis Vertical Slice — Engineering Verification Review

**Date:** 2026-09-23  
**Scope:** Code-level verification of progress-report claims vs implementation + tests  
**Principle:** Do not mark complete because a happy-path E2E passes.

---

## Claim classification legend

| Label | Meaning |
|-------|---------|
| **Verified** | Implemented and proven by automated tests (or live exercise with reproducible criteria) |
| **Implemented, under-tested** | Present in code; proof incomplete |
| **Partial** | Present but weaker than the report wording |
| **Not implemented** | Absent |
| **Deferred** | Explicitly out of slice / planned |

---

## Claim-by-claim matrix

| Claim | Classification | Evidence / caveat |
|-------|----------------|-------------------|
| Tenant isolation (read by JWT tenant) | **Verified** (reads) | Controllers use `_tenant.TenantId`; repos filter `TenantId`. E2E cross-tenant GET → 404 for customer/tx/alert; PR1 audit isolation test. |
| Tenant isolation (no client `tenantId` override) | **Verified** | Ingest/customer/account bodies have no tenant field; tenant from JWT middleware only. |
| Tenant isolation (rules API) | **Deferred / N/A** | No public rules API; rules seeded per tenant in DB. |
| Tenant isolation (alert *modify*) | **Deferred** | No assign/dismiss APIs yet — cannot modify via API. |
| Auth: JWT signature / issuer / audience / exp | **Implemented, under-tested** | `JwtBearer` validates issuer, audience, signing key, lifetime. No automated invalid/expired token tests. |
| Authorization / RBAC | **Partial** | Roles on JWT; **no** policy/role checks on endpoints. Any authenticated tenant user can do everything the API allows. |
| Dev bootstrap gated | **Verified** | Integration test: Production + `AllowDevBootstrap=false` → 404. **Risk:** `appsettings.json` defaults bootstrap **true** and embeds JWT key. |
| Sequential idempotency | **Verified** | Integration + E2E: second POST → `wasCreated=false`. Unique index `(tenant_id, ExternalReference)`. |
| Concurrent idempotency | **Verified** (post-fix) | Was **bug** (HTTP 500). Fixed: UoW maps Postgres `23505` → re-read → idempotent. New concurrent integration test (20 parallel). |
| Configurable structuring thresholds | **Verified** | Thresholds in persisted `RuleDefinition` JSON; generic `ConditionEvaluator`. Unit test: change thresholds → detection flips. |
| No hardcoded structuring thresholds in C# | **Verified** | No `if (count >= 5 && sum >= …)`. |
| Orchestration filters `STRUCTURING_001` | **Partial** | Use case filters to that rule code (slice scope), not threshold hardcoding. |
| Feature window correctness | **Implemented, under-tested → improved** | Code: inclusive `[asOf−24h, asOf]`. New unit boundary test. Design text sometimes says half-open — document inclusive as source of truth. |
| Money as decimal/`numeric` | **Verified** | EF `numeric(18,2)`; domain `Money` decimal; E2E decimal round-trip. |
| Alert dedupe identity | **Verified** (app + DB) | Key: `{tenant}:{ruleId}:{ruleVersionId}:{focusType}:{focusEntityId}:{yyyy-MM-dd}` (UTC day of **tx Timestamp**). Unique index on `(tenant_id, deduplication_key)`. Unit + integration dedupe tests. |
| Concurrent alert dedupe | **Implemented, under-tested** | DB unique protects; concurrent race may still 500 or roll back whole ingest if alert unique fails mid-batch. Not fully hardened like tx idempotency. |
| Alert explainability / version snapshot | **Partial** | Alert stores `RuleVersionId`, evidence JSON (features, conditions, tx ids, ruleCode). **Not** a frozen full rule JSON snapshot. Changing v1 definition in place would blur history (versions are intended immutable when ACTIVE). |
| “Immutable” audit trail | **Partial — wording too strong** | Append-only *writer* API; no update/delete endpoints; domain has no mutators. **Was** client `POST /audit-events` (arbitrary invent). **Removed** client create. Still no DB trigger/RLS preventing raw SQL updates. Prefer: “append-only application audit log”. |
| Atomic write (tx+audits+alert) | **Implemented, under-tested** | Single `SaveChangesAsync` in use case. No forced mid-failure rollback integration test yet. |
| Error handling (no SQL leak) | **Partial** | Concurrent unique now mapped. Other failures may still hit Developer Exception Page in Development (stack traces). |
| DB constraints vs app-only | **Partial** | Unique: tx external ref, alert dedupe, rule code, rule version. **Missing FKs:** account→customer, tx→account/customer, alert→rule version. |
| Cross-tenant ingest referencing foreign IDs | **Verified** by code path | Customer/account must exist **in JWT tenant** or BadRequest. |
| Rules API / cases / KYC / screening / risk | **Deferred** | Module shells only. |
| Production IdP / secrets / observability | **Deferred** | Local JWT + committed signing key. |

---

## A. Verified

- End-to-end structuring path: ingest → features → JSON rule → alert → system audits  
- Tenant-scoped **reads** for customers, accounts, transactions, alerts, audits  
- Tenant identity from JWT, not request body  
- Sequential transaction idempotency + DB unique index  
- Concurrent transaction idempotency (after fix) + automated parallel test  
- Generic rule evaluation; thresholds data-driven  
- Alert dedupe key includes rule version + UTC business day; DB unique index  
- Decimal/numeric money path  
- Dev bootstrap can be disabled  

## B. Gaps

- No meaningful RBAC beyond “authenticated”  
- No invalid/expired JWT automated tests  
- Feature window semantics documented inconsistently (inclusive vs half-open)  
- Concurrent **alert** uniqueness not API-hardened like transactions  
- No atomicity failure/rollback integration test  
- No EF global query filters (isolation depends on every query remembering TenantId)  
- Architecture NetArchTest suite empty  
- Alert evidence is not a full frozen rule-document snapshot  
- No FK constraints across aggregates  

## C. Bugs (found / addressed)

| Bug | Status |
|-----|--------|
| Concurrent same `externalReference` → HTTP 500 | **Fixed** (`UniqueConstraintViolationException` + re-read) |
| Client could invent audit events via `POST /audit-events` | **Fixed** (endpoint removed; MethodNotAllowed asserted) |
| Concurrent alert unique violation can fail entire ingest batch | **Open** |

## D. Security concerns (before any external exposure)

1. JWT signing key + `AllowDevBootstrap: true` in `appsettings.json`  
2. CORS `AllowAnyOrigin`  
3. Auto-migrate on startup  
4. Swagger enabled in Development (OK if env-gated)  
5. Developer exception page can leak internals in Development  
6. Authentication ≠ authorization (any user in tenant is omnipotent)  
7. Audit not cryptographically / DB-enforced immutable  

## E. Required before compliance frontend

1. ~~Concurrent ingest idempotency~~ (done)  
2. ~~Remove client audit create~~ (done)  
3. Alert list **filter/pagination** (status, date, severity) — console unusable otherwise  
4. Alert **assign / status transition** APIs  
5. Transaction + customer **list/search**  
6. **GET rules / rule versions** (read-only) for explainability UI  
7. Gate bootstrap + move secrets to env/user-secrets  
8. Consistent problem+json errors (no stack traces to clients)  

## F. Deferred (safe outside slice)

- Full rule CRUD / simulate / approve  
- Cases, KYC, KYB, screening, risk, reporting  
- Production IdP, hardened RBAC model  
- Event bus / outbox / webhooks  
- Bulk ingest (nice-to-have soon, not slice-blocking)  
- Full observability / DR  

## G. Test additions

| Guarantee | Test | Status |
|-----------|------|--------|
| Tenant isolation | E2E cross-tenant GET | Exists |
| Sequential idempotency | Integration/E2E | Exists |
| Concurrent idempotency | `ConcurrentIdempotencyTests` 20-way parallel | **Added** |
| Rule configurability | Threshold change flips trigger | **Added** |
| Feature window boundary | Inclusive 24h edges | **Added** |
| Alert dedupe | Integration 7×95k → 1 alert | Exists |
| Concurrent alert dedupe | Parallel matching ingest | **Still needed** |
| Audit invent rejected | POST → 405 | **Added** |
| Audit immutability (DB) | No update/delete API | Soft — no raw SQL test |
| Atomicity rollback | Forced failure | **Still needed** |
| Rule versioning v1 vs v2 | Different dedupe keys / historic RuleVersionId | Partial (unit key + CreateActive); **E2E activate v2 still needed** |
| Auth invalid JWT | 401 | **Still needed** |

---

## Report wording corrections (for supervisor)

| Avoid | Prefer |
|-------|--------|
| “immutable audit trail” | “append-only application audit log (system-written; client invent removed)” |
| “fully hardened ingestion” | “idempotent under sequential and concurrent duplicate externalReference (alert concurrency still open)” |
| “60+ tests prove correctness” | “61+ tests; invariant matrix is the proof, not the count” |
| “production-shaped complete” | “architecture-validated vertical slice; operator APIs and frontend still required” |

---

## API gap list (minimum for compliance console)

**Alerts:** filter, paginate, assign, dismiss/resolve, detail (exists), create-case  
**Transactions:** list/search/filter by customer/date/amount, detail (exists), bulk  
**Customers:** list/search, detail (exists), accounts, related alerts/txs  
**Rules:** list/read active version + definition (write lifecycle later)  
**Auth:** refresh/logout; real user admin  

---

## Bottom line

The vertical slice **does** prove the architectural assumption and most report claims that are carefully worded. Several claims were **stronger than the code** (immutable audit, concurrent idempotency, RBAC). Concurrent ingest and client audit invent are **fixed in this review pass**. Remaining blockers before a real frontend are **operator APIs**, **secrets/bootstrap hygiene**, and **alert-concurrency / atomicity tests**—not re-proving structuring detection.
