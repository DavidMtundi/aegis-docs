# High-Risk Geography Implementation Plan

> **For agentic workers:** Execute task-by-task. Prefer TDD.

**Goal:** Seed `HIGH_RISK_GEOGRAPHY_001` and feature fields so ingest with a high-risk `counterpartyCountry` produces an explainable alert via the shared engine.

**Design:** `aegis-docs/designs/2026-09-23-high-risk-geography.md`

## Files

| File | Change |
|------|--------|
| `aegis/src/Aegis.Modules.Features/Application/FeatureCalculator.cs` | Add `counterparty_country`, `transaction_amount` from triggering tx |
| `aegis/src/Aegis.Infrastructure/Aml/StructuringRuleSeeder.cs` | Seed `HIGH_RISK_GEOGRAPHY_001` |
| `aegis/tests/Integration/Aml/HighRiskGeographyTests.cs` | Positive + negative cases |
| `aegis/tests/EndToEnd/...` (optional) | One E2E if vertical-slice suite expects three codes |

## Task 1 — Failing integration test

Write `HighRiskGeographyTests` mirroring `RapidMovementTests`:
1. Bootstrap tenant, customer (country KE), account
2. POST tx with `counterpartyCountry: "KP"`, amount `15000` → assert some `alertIds` and list/rules or evaluation contains `HIGH_RISK_GEOGRAPHY_001`
3. Separate fact: `counterpartyCountry: "KE"` → no alert with that rule code

Run; expect fail (rule not seeded / features missing).

## Task 2 — Features

In `FeatureCalculator`, after loading txs, resolve triggering tx (`Timestamp == asOfTimestamp`, else max timestamp). Set:

```csharp
["counterparty_country"] = triggering?.CounterpartyCountry?.Trim().ToUpperInvariant() ?? ""
["transaction_amount"] = triggering?.Amount.Amount ?? 0m
```

Include triggering id in `TransactionIds` if not already in the 24h set (or always union).

## Task 3 — Seed rule

In `StructuringRuleSeeder`, add `HighRiskGeographyCode = "HIGH_RISK_GEOGRAPHY_001"` and `EnsureHighRiskGeographyAsync` with `IN` Values list `KP`,`IR`,`SY` and amount `>= 10000`. Call from `EnsureSeededAsync`.

## Task 4 — Verify

`dotnet test` filter `HighRiskGeography` + existing Rapid/Structuring still green. Commit and ff-merge to `aegis` main.
