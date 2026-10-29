# Phase 5.5 — Ingest + evaluate load baseline (§43)

**Status:** Script shipped; run locally and paste numbers below  
**Date:** 2026-10-28  
**Script:** `aegis/scripts/load-test-ingest.py`

## Context

PRD §43 requires measurable expensive paths and designs that avoid full-history scans per transaction. It does **not** publish a hard TPS SLA. This step records a **local baseline** for ingest + rule evaluation.

## How to run

```bash
# Terminal 1 — raise API limiter for the burst
cd aegis
RateLimiting__Api__PermitLimit=100000 ASPNETCORE_ENVIRONMENT=Development \
  dotnet run --project src/Aegis.Api --urls http://127.0.0.1:5092

# Terminal 2
python3 scripts/load-test-ingest.py --count 200 --concurrency 20 \
  --api http://127.0.0.1:5092
```

## Recorded baseline

| Field | Value |
|---|---|
| Host / date | local darwin · 2026-10-28 |
| count / concurrency | **200 / 20** (`RateLimiting__Api__PermitLimit=100000`) |
| ok / fail | 200 / 0 |
| txs/sec | **104.44** |
| latency p50 / p95 (ms) | 175.2 / 407.8 |

Earlier spot check (100 / 10 against a prior process): 92.25 txs/sec, p50 62.5 ms, p95 233.2 ms.

Interpretation: laptop/dev baseline, not a capacity commitment. Re-run on pilot hardware before go-live and attach results here.
