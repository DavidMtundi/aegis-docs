# Phase 4.2 Maker-Checker Implementation Plan

> **For agentic workers:** Use subagent-driven-development or executing-plans. Steps use `- [ ]`.

**Goal:** Enforce Creator ≠ Approver before a rule version can activate.

**Architecture:** Extend `AmlRuleVersion` transitions; add `rule.approve`; wire submit/approve/reject/withdraw; restrict activate to APPROVED; update console + tests.

**Spec:** `aegis-docs/designs/2026-10-28-phase-4-2-maker-checker.md`

---

### Task 1: Domain + permission

- [ ] `SubmitForApproval`, tighten `Approve` (pending only + ≠ creator), `ReturnToDraft`
- [ ] `Permissions.RuleApprove` + Reviewer/Admin grants
- [ ] Unit tests
- [ ] Commit

### Task 2: API

- [ ] submit / approve / reject / withdraw endpoints
- [ ] Activate only from APPROVED; remove create `activate`
- [ ] Integration two-user tests; fix RuleLifecycleTests
- [ ] Commit

### Task 3: Console

- [ ] Client helpers; remove activate-on-create; detail actions
- [ ] Commit

### Task 4: Verify

- [ ] Unit + integration; smoke with two roles if feasible
- [ ] Roadmap 4.2 ✅
