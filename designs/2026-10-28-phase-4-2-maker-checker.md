# Phase 4.2 — Maker-checker rule approval

**Status:** Approved (recommended defaults)  
**Date:** 2026-10-28  
**Repos:** `aegis` API + domain, `aegis-console`  
**Depends on:** 4.1 create/suspend/resume/retire  
**Out of scope:** 4.3 simulation, 4.4 rich editor / approval inbox polish beyond detail actions

## Problem

Activation today auto-approves the acting user (`Approve` + `Activate` in one step). Create-with-`activate: true` also skips review. PRD §48 requires Creator ≠ Approver for sensitive config.

## Locked decisions

| Topic | Choice |
|--------|--------|
| Scope | Every tenant-authored go-live uses submit → approve → activate. No create+activate shortcut. |
| Separation | Approver user id ≠ version `CreatedBy`. Activator may be creator or approver (four-eyes is on approve). |
| Reject / withdraw | Reject (approver) and withdraw (creator) both return `PENDING_APPROVAL` → `DRAFT`. |
| Seed / system | `DefaultRuleSeeder` / `CreateActive` unchanged (system actor). |
| Resume | Still direct (already-approved policy returning to service). |

## Approaches

1. **Version state machine + permissions (recommended)** — `Submit` / `Approve` / `Reject` / `Withdraw` on `AmlRuleVersion`; `rule.approve` for Reviewer+Admin; activate only from `APPROVED`.
2. **Separate Approval aggregate** — heavier audit model; defer.
3. **Admin bypass** — rejected; weakens §48.

**Chosen:** 1.

## Design

### Domain (`AmlRuleVersion`)

- `SubmitForApproval()` — `DRAFT` → `PENDING_APPROVAL`
- `Approve(approvedBy)` — only from `PENDING_APPROVAL`; throw if `approvedBy == CreatedBy`
- `Reject()` — `PENDING_APPROVAL` → `DRAFT`; clear approval fields if any
- `Withdraw()` — same as reject (creator path); domain can share `ReturnToDraft()`
- `Activate` — only from `APPROVED` (caller enforces); remove silent approve-from-draft in controller

### Permissions

- Add `rule.approve`
- Grants: **Admin**, **Reviewer** (not Analyst/Viewer)
- Mirror in console `permissions.ts`

### API

| Method | Path | Permission | Notes |
|--------|------|------------|-------|
| `POST` | `/versions/{versionId}/submit` | `rule.create` | DRAFT → PENDING |
| `POST` | `/versions/{versionId}/approve` | `rule.approve` | Creator ≠ self → APPROVED; audit `RULE_APPROVED` |
| `POST` | `/versions/{versionId}/reject` | `rule.approve` | → DRAFT; audit `RULE_UPDATED` or dedicated if exists |
| `POST` | `/versions/{versionId}/withdraw` | `rule.create` | Only `CreatedBy`; → DRAFT |
| `POST` | `/versions/{versionId}/activate` | `rule.activate` | Only `APPROVED` (or resume `SUSPENDED` via existing resume/activate path) |

Create: drop `activate` (ignore with `400` if true). Always DRAFT v1.

Sync parent `AmlRule` status on submit/approve/activate where cheap (`SubmitForApproval` / `Approve` / `Activate`).

### Console

- Remove “Activate immediately” on new rule.
- Version row actions: Submit (DRAFT + create), Approve/Reject (PENDING + approve), Withdraw (PENDING + creator), Activate (APPROVED + activate).

### Tests

- Unit: cannot approve own; illegal transitions.
- Integration: two users — creator submit; creator approve fails; other approve; activate succeeds; draft activate fails.

## Success

1. No path from DRAFT → ACTIVE without a different user’s approve.
2. Seeded defaults still ACTIVE for new tenants.
3. Console supports the queue actions on the rule detail page.
