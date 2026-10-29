# Phase 4.4 — Console rule editor, validation, approval queue

**Status:** Approved (recommended defaults)  
**Date:** 2026-10-28

## Scope

Close the Phase 4 console gap: schema validation before save/submit, a dedicated approval queue, and draft definition editing. Simulation results already exist on detail (4.3); add lookback control.

## Design

### API

| Endpoint | Permission | Behavior |
|----------|------------|----------|
| `POST /api/v1/rules/validate` | `rule.create` | Body `{ definition }` → `{ valid, errors[] }` via `RuleDefinitionValidator` |
| `GET /api/v1/rules/pending-approval` | `rule.approve` | All `PENDING_APPROVAL` versions for tenant |
| `PUT /api/v1/rules/versions/{id}` | `rule.create` | Update definition on **DRAFT** only (creator or any create-capable user); re-validate |

### Console

- `/rules/approvals` — queue table with Approve / Reject (and link to rule)
- Rules list header link to Approvals when `rule.approve`
- `/rules/new` — Validate before Create; show errors
- Rule detail — for DRAFT: editable JSON, Validate, Save; Simulate lookback days selector

### Out of scope

Visual condition builder, full approval inbox notifications, editing PENDING without withdraw.
