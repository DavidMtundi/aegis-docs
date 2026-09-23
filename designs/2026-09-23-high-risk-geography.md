# High-Risk Geography Scenario Design

**Status:** APPROVED  
**Date:** 2026-09-23  
**Product:** Aegis AML Compliance Platform  
**Related:** PRD §50.3; existing STRUCTURING_001 / RAPID_MOVEMENT_001 seeds

## Goal

Prove the third MVP AML scenario via the **same** rule evaluation engine (no scenario-specific branching in Application).

> Ingest a transaction with a high-risk counterparty country → configurable rule fires → explainable alert.

## Approach (locked)

**Features + rule `IN` list** — jurisdiction policy lives in seed/draft rule JSON, not in FeatureCalculator.

## Features (add)

| Feature | Source |
|---------|--------|
| `counterparty_country` | Triggering transaction’s `CounterpartyCountry` (empty string if null) |
| `transaction_amount` | Triggering transaction’s amount |

“Triggering” = the customer transaction whose `Timestamp == asOfTimestamp` (the just-ingested row). Fallback: latest tx in the loaded window.

## Seed rule

- Code: `HIGH_RISK_GEOGRAPHY_001`
- Scenario: `HIGH_RISK_GEOGRAPHY`
- Focus: CUSTOMER
- Lookback: `24h` (unused beyond feature load; conditions are on current-tx features)
- Conditions (ALL):
  - `counterparty_country` `IN` demo list: `KP`, `IR`, `SY`
  - `transaction_amount` `>=` `10000`
- Severity: HIGH, riskScore: 70, actions: `CREATE_ALERT`

List is demonstrably editable later via existing draft/activate APIs.

## Wire-up

- Extend `StructuringRuleSeeder.EnsureSeededAsync` (same class as RAPID)
- No new modules, no migration, no console changes (alerts already generic)

## Tests

- Positive: `counterpartyCountry=KP`, amount 15_000 → alert with rule code `HIGH_RISK_GEOGRAPHY_001`
- Negative: `counterpartyCountry=KE`, amount 15_000 → no geo alert
- Optional: amount below threshold with KP → no alert

## Non-goals

Customer risk score, direction filters, FATF feed, tenant jurisdiction table, hardcoding countries in FeatureCalculator.
