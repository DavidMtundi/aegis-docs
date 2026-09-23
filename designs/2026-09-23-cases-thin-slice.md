# S1 — Cases Thin Slice Design

**Status:** Approved for implementation (MVP demo acceleration plan)  
**Date:** 2026-09-23  
**Goal:** Close Alert → Case → Disposition → Audit for the supervisor demo script.

## Scope

In:
- Case aggregate (status, disposition, notes, linked alert ids, optional customer id)
- APIs: create-from-alert, list/get, assign, notes, close (disposition + required conclusion)
- Audit events: CASE_CREATED, CASE_ASSIGNED, CASE_UPDATED (notes), CASE_CLOSED
- Console: create case from alert, cases list/detail, notes, close
- Integration/E2E proof

Out:
- Escalation workflows, due dates, multi-alert merge UX, KYC/screening 360, documents/tasks

## Domain

```text
Case
  id, tenantId, customerId?, title, status, priority, assignedTo?
  openedAt, closedAt?, disposition?, conclusion?
  linkedAlertIds[]
  notes[] { id, text, authorId, createdAt }
```

Statuses: OPEN, INVESTIGATING, PENDING_REVIEW, ESCALATED, CLOSED  
Dispositions: FALSE_POSITIVE, NO_SUSPICIOUS_ACTIVITY, SUSPICIOUS_ACTIVITY, ESCALATED, REPORTED, OTHER

Rules:
- Creating from alert sets status OPEN, priority from alert severity mapping, links alert id, customerId when focusEntityId is a GUID
- Close requires disposition + non-empty conclusion; sets CLOSED + closedAt
- Notes append-only on the aggregate (stored as jsonb list for thin slice)

## Persistence

Schema `cases`, table `cases`. Notes and alert links as jsonb columns (YAGNI — no separate note table until needed).

## APIs

- `POST /api/v1/alerts/{id}/create-case`
- `GET /api/v1/cases?page=&pageSize=`
- `GET /api/v1/cases/{id}`
- `POST /api/v1/cases/{id}/assign` `{ assignedTo? }`
- `POST /api/v1/cases/{id}/notes` `{ text }`
- `POST /api/v1/cases/{id}/close` `{ disposition, conclusion }`

## Console

- Alert detail: Create case button
- `/cases`, `/cases/[id]` with notes + close form
- Nav link Cases
