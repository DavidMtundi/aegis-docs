# Phase 4.1 Rule Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let compliance create new AML rules and suspend / resume / retire the current version, with console UI and tests.

**Architecture:** Extend `AmlRuleVersion` state machine with `SUSPENDED` + suspend/resume/retire; add create + lifecycle endpoints on `RulesController`; list/get resolve “current” version (ACTIVE > SUSPENDED > latest); console form + action buttons.

**Tech Stack:** .NET 9, EF Core, Next.js console, existing `RuleDefinitionValidator`, audit events.

**Spec:** `aegis-docs/designs/2026-10-28-phase-4-1-rule-lifecycle.md`

## Global Constraints

- No maker-checker (4.2); no simulate (4.3).
- Evaluation continues to load ACTIVE versions only.
- Permissions: create=`rule.create`; activate/suspend/resume/retire=`rule.activate`.
- Match existing controller/audit/repository patterns; minimal diff.

---

## File map

| File | Role |
|------|------|
| `Aegis.Modules.Aml/Domain/RuleVersionStatus.cs` | Add `SUSPENDED` |
| `Aegis.Modules.Aml/Domain/AmlRuleVersion.cs` | Suspend / Resume / Retire |
| `Aegis.Modules.Aml/Application/IAmlRepositories.cs` | Get rule by id; list current helpers |
| `Aegis.Infrastructure/.../Repositories.cs` | Implement repo methods |
| `Aegis.Api/Controllers/RulesController.cs` | POST create, suspend, resume, retire; list/get current |
| `tests/Unit/Aml/AmlRuleVersionLifecycleTests.cs` | State machine unit tests |
| `tests/Integration/.../RulesLifecycleApiTests.cs` | API integration |
| `aegis-console/.../lib/api/rules.ts` + types | Client helpers |
| `aegis-console/.../rules/page.tsx` | Status + New rule |
| `aegis-console/.../rules/new/page.tsx` | Create form |
| `aegis-console/.../rules/[id]/page.tsx` | Suspend / Resume / Retire |

---

### Task 1: Domain lifecycle

**Files:** `RuleVersionStatus.cs`, `AmlRuleVersion.cs`, new unit test file

- [x] Add `SUSPENDED` to enum (after `ACTIVE` or before `SUPERSEDED`).
- [x] Implement `Suspend()`, `Resume()`, `Retire()` with `InvalidOperationException` on illegal transitions.
- [x] Unit tests: ACTIVE→SUSPENDED→ACTIVE; ACTIVE→RETIRED; SUSPENDED→RETIRED; DRAFT cannot suspend; RETIRED cannot resume.
- [ ] Commit: `feat(aml): suspend/resume/retire rule version states`

### Task 2: Repositories + list current

**Files:** `IAmlRepositories.cs`, `Repositories.cs`

- [x] `IAmlRuleRepository.GetByTenantAndIdAsync`.
- [x] `ListByTenantAsync` on versions (all versions for tenant) **or** dedicated current-resolution helper used by controller.
- [ ] Commit: `feat(aml): rule repo get-by-id and list for lifecycle`

### Task 3: API create + lifecycle

**Files:** `RulesController.cs`, integration tests

- [x] `POST /api/v1/rules` create (+ optional activate).
- [x] Change `GET /` and `GET /{id}` to current-resolution.
- [x] `POST /{id}/suspend|resume|retire`.
- [x] Block `CreateDraft` when rule is `RETIRED`.
- [x] Integration tests for create, duplicate code, suspend/resume/retire, activate-on-create permission.
- [ ] Commit: `feat(api): create/suspend/resume/retire rules`

### Task 4: Console

**Files:** rules API client, list, new, detail pages

- [x] Client: `createRule`, `suspendRule`, `resumeRule`, `retireRule`.
- [x] List status + New rule link.
- [x] New rule page (guided + JSON + activate checkbox).
- [x] Detail actions.
- [ ] Commit: `feat(console): rule create and lifecycle actions`

### Task 5: Verify

- [x] Run unit + relevant integration tests.
- [x] Manual smoke: create draft, create+activate, suspend (no fire), resume, retire.
- [x] Update roadmap checkbox for 4.1 if present.
- [ ] Commit docs if roadmap touched.
