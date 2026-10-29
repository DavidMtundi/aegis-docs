# Phase 4.3 — Rule simulation

**Status:** Approved (recommended defaults)  
**Date:** 2026-10-28

## Goal

Run a rule version’s definition against historical tenant activity **without** creating alerts or changing live rules. Compare projected triggers to the currently active version of the same rule when one exists (PRD §23).

## Approach (chosen)

**Per-customer feature eval over a lookback window** — select up to N customers with recent transactions; for each, compute features at their latest tx timestamp and evaluate candidate (+ active baseline). Read-only.

Not chosen: full transaction replay (heavy) or SQL-only threshold count (too inaccurate for multi-condition rules).

## API

`POST /api/v1/rules/versions/{versionId}/simulate`  
Permission: `rule.read` (any compliance reader can dry-run; create/approve stay separate)

Body: `{ "lookbackDays": 30 }` (1–90, default 30)

Response:

```json
{
  "versionId": "...",
  "ruleId": "...",
  "lookbackDays": 30,
  "customersSampled": 120,
  "transactionsInSample": 4000,
  "candidateTriggers": 42,
  "activeTriggers": 61,
  "triggerDelta": -19,
  "sampleTriggeredCustomerIds": ["..."]
}
```

`activeTriggers` / `triggerDelta` null when no ACTIVE sibling.

## Caps

- Max 200 customers, 5 000 transactions scanned for sampling.
- No DB writes; no audit required for MVP (optional later).

## Console

Rule detail: “Simulate” on non-retired versions → show summary counts.
