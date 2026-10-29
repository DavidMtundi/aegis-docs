# Phase 5.2–5.6 — Hardening (recommended defaults)

**Status:** Implemented (auto-approved defaults)  
**Date:** 2026-10-28  
**Note:** 5.7 security review remains a **separate session**.

## 5.2 Limits & forwarded headers ✅
- `UseForwardedHeaders` (X-Forwarded-For/Proto) before rate limiting
- Login limiter partitions by forwarded client IP
- Global fixed-window limiter partitioned by `tenant_id` claim (else IP): configurable permits/window; `[EnableRateLimiting("api")]` on non-auth controllers
- Kestrel `MaxRequestBodySize` + multipart limit (default 10 MB)

## 5.3 Refresh + logout ✅
- Persist refresh tokens (hashed) in `identity.refresh_tokens`
- Login returns `accessToken` + `refreshToken`
- `POST /auth/refresh` rotates (revoke old, issue new)
- `POST /auth/logout` revokes refresh (and clears console session)
- Console stores refresh in sessionStorage; logout calls API

## 5.4 CI ✅
- aegis: build/test + `dotnet list package --vulnerable` + container build
- aegis-console: lint / typecheck / test / build / `npm audit`
- aegis-docs: required-path + non-empty markdown + README link check

## 5.5 Load test ✅
- `aegis/scripts/load-test-ingest.py` burst ingest; results template in `designs/2026-10-28-phase-5-5-load-baseline.md` (§43 baseline; PRD has no hard TPS)

## 5.6 Backup/restore ✅
- Runbook `runbooks/postgres-backup-restore.md` + `aegis/scripts/backup-restore-smoke.sh` (`pg_dump`/`pg_restore` against docker postgres)
