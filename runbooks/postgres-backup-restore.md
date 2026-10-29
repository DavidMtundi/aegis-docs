# Postgres backup & restore (local / pilot)

**Audience:** operators running the docker-compose Postgres used by Aegis API.  
**Related script:** `aegis/scripts/backup-restore-smoke.sh`

## What we back up

- Database `aegis` (default compose credentials: user `aegis`, password from compose).
- Format: custom (`-Fc`) so restores can be selective and parallel later.

Redis is ephemeral cache for MVP; do not treat it as durable state.

## Backup (manual)

```bash
cd aegis
docker compose up -d postgres
CONTAINER=$(docker compose ps -q postgres)
mkdir -p .backups
docker exec "$CONTAINER" pg_dump -U aegis -Fc -d aegis > ".backups/aegis-$(date -u +%Y%m%dT%H%M%SZ).dump"
```

Store dumps off the host disk that holds the volume (object storage / encrypted USB for pilots).

## Restore into a new database (safe verify)

```bash
# create empty target
docker exec -i "$CONTAINER" psql -U aegis -d postgres -c "CREATE DATABASE aegis_restored OWNER aegis;"
docker exec -i "$CONTAINER" pg_restore -U aegis -d aegis_restored --no-owner --role=aegis < .backups/YOUR.dump
```

Point a throwaway API instance at `aegis_restored` via connection string and hit `/health` + login before swapping production DNS.

## Restore overwriting the live database (destructive)

1. Stop API writers (`docker compose stop` API process / scale to 0).
2. `DROP DATABASE aegis;` then `CREATE DATABASE aegis OWNER aegis;`
3. `pg_restore` into `aegis`.
4. Start API with `Database:MigrateOnStartup` **false** if the dump already includes schema; otherwise allow migrate once and re-check.

## Smoke test

From `aegis/`:

```bash
chmod +x scripts/backup-restore-smoke.sh
./scripts/backup-restore-smoke.sh
```

Expect: dump written under `.backup-smoke/`, restore into `aegis_restore_smoke`, table count ≥ 1, smoke DB dropped, dump retained.

## Cadence (pilot default)

| Cadence | Action |
|---|---|
| Daily | Automated `pg_dump -Fc` retained 14 days |
| Weekly | Restore-to-scratch verification (this smoke script) |
| Before major migrate | Extra named dump |

## Notes

- Forwarded secrets (JWT signing key, tenant admin passwords) are **not** in Postgres alone — rotate keys after a restore onto a new host if compromise was the trigger.
- Phase 5.7 security review should validate retention, encryption at rest, and access to dump storage.
