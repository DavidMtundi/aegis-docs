# Phase 4.1 — Create, suspend, resume, retire rules

**Status:** Approved (recommended defaults; implement without further product Q&A)  
**Date:** 2026-10-28  
**Repos:** `aegis` API + domain, `aegis-console` rules UI  
**Out of scope:** 4.2 maker-checker, 4.3 simulation, 4.4 schedule lookback

## Problem

Compliance can only list active rules, add a draft onto an *existing* rule, and activate a version. They cannot create a rule from scratch, temporarily stop an active rule, restore it, or permanently retire it — even though the domain already sketches `Disable` / `Retire` and audit types `RULE_DISABLED` / `RULE_RETIRED`.

## Decisions (locked)

| Topic | Choice |
|--------|--------|
| Suspend vs retire | Suspend = temporary off (reversible). Retire = permanent end. |
| Create + go-live | If caller has `rule.activate`, optional `activate: true` creates and activates in one request; otherwise DRAFT. |
| After suspend | Resume restores the **same** version to ACTIVE (no new draft). |
| Console create UX | Guided fields (code, name, description, scenario, severity/risk) + JSON definition body. |
| Maker-checker | Not in 4.1 (stays 4.2). Activate may still approve+activate as today. |

## Approaches considered

1. **Extend existing RulesController + domain (recommended)** — Add version statuses/`Suspend`/`Resume`/`Retire`, `POST /rules`, and lifecycle endpoints on the current controller; thin console pages. Matches Phase 3 style; lowest risk.
2. **Application commands/handlers** — Cleaner long-term, more files for a small surface; defer until 4.2 pulls in approval workflow.
3. **Full PRD surface** (`validate`/`simulate`/`submit`/`approve`) — Bundles 4.2–4.3; too large.

**Chosen:** Approach 1.

## Design

### Domain

- Add `RuleVersionStatus.SUSPENDED`.
- `AmlRuleVersion`:
  - `Suspend()` — only from `ACTIVE` → `SUSPENDED`.
  - `Resume()` — only from `SUSPENDED` → `ACTIVE`.
  - `Retire()` — from `ACTIVE` or `SUSPENDED` → `RETIRED`; set `EffectiveTo` if missing.
- `AmlRule` already has `Disable()` / `Retire()` / `Activate()` — call in sync with version transitions.
- Evaluation keeps using `GetActiveByTenantAsync` (ACTIVE only) → suspended/retired rules stop firing without ingest changes.
- Retired rules: reject new draft versions (`409` / bad request with clear message).

### API (`/api/v1/rules`)

Align closer to PRD while keeping existing version activate path:

| Method | Path | Permission | Behavior |
|--------|------|------------|----------|
| `POST` | `/` | `rule.create` | Create rule + v1. Body: code, name, description, scenarioType, definition, optional `activate`. If `activate` and caller lacks `rule.activate` → `403`. Unique code per tenant → `409`. Validate definition. |
| `GET` | `/` | `rule.read` | List **current** version per rule: prefer ACTIVE, else SUSPENDED, else latest non-SUPERSEDED (so drafts and retired remain visible). |
| `GET` | `/{ruleId}` | `rule.read` | Same “current” resolution (not active-only). |
| `POST` | `/{ruleId}/suspend` | `rule.activate` | Active version → SUSPENDED; rule → DISABLED. Audit `RULE_DISABLED`. |
| `POST` | `/{ruleId}/resume` | `rule.activate` | Suspended version → ACTIVE; rule → ACTIVE. Audit `RULE_ACTIVATED` (or keep DISABLED→ACTIVATED narrative). |
| `POST` | `/{ruleId}/retire` | `rule.activate` | Active or suspended → RETIRED; rule → RETIRED. Audit `RULE_RETIRED`. Idempotent-ish: already retired → `200` current. |

Existing: `POST /{id}/versions`, `POST /versions/{versionId}/activate` unchanged.

### Repositories

- `IAmlRuleRepository.GetByTenantAndIdAsync`.
- `IAmlRuleVersionRepository.ListCurrentByTenantAsync` (or implement selection in controller from a list-all-latest query). Prefer one efficient query: versions for tenant, group in memory by `RuleId`.

### Console

- Rules list: status column; “New rule” → `/rules/new` (`rule.create`).
- `/rules/new`: form fields + JSON textarea for definition; checkbox “Activate immediately” if `rule.activate`.
- Detail: Suspend (ACTIVE), Resume (SUSPENDED), Retire (ACTIVE|SUSPENDED); hide Activate for retired/suspended appropriately (resume is the path for suspended).

### Testing

- Unit: version state machine (illegal transitions throw / no-op policy — prefer throw `InvalidOperationException` caught as `409`).
- Integration: create draft; create+activate; suspend stops appearing in active evaluation set; resume restores; retire blocks new drafts; duplicate code `409`.

### Non-goals

- Separate `rule.suspend` permission.
- Soft-delete of history rows.
- Changing how seeded default rules are created.
- Simulation / approval workflow.

## Success criteria

1. Operator with `rule.create` can author a new rule in the console and see it listed.
2. Operator with `rule.activate` can activate-on-create, suspend, resume, and retire; evaluation ignores non-ACTIVE versions.
3. Audit events recorded for create / disable / retire / activate (resume).
4. Automated tests cover happy paths and conflict cases above.
