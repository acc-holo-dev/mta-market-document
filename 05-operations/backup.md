status: current
version: 1.0
last_verified: 2026-09-09

# Backup (O-003)

Scope: PostgreSQL, object-storage artifacts + metadata, critical
configuration, key recovery. This document defines the **target policy** and
records honestly what exists today (`mta-market-site/scripts/backup.sh`) and
where it falls short.

## Targets (MVP)

| Parameter | Target |
|---|---|
| RPO | **24 h** (daily full backup; no point-in-time recovery yet) |
| RTO | **4 h** (restore onto a fresh Postgres 16 container + re-point storage/config) |
| Retention | **30 days** |
| Locations | local `./backups` (current script) — must be shipped to a second, off-host location (S3/R2 bucket or second host) to satisfy "backup survives host loss" |
| Encryption | backups containing any secret-bearing material must be encrypted at rest in the off-host location |

## What is backed up

| Asset | Method | Today |
|---|---|---|
| PostgreSQL (`mtamarket`) | `pg_dump` (plain SQL) via `docker-compose -f docker-compose.prod.yml exec -T postgres` | implemented in `scripts/backup.sh`, manual/daily-by-cron, local dir |
| Object storage: artifacts + manifests (or `./uploads` in local mode) | `tar -czf` of `./uploads` | implemented (local mode only; no S3-side snapshot policy yet) |
| Environment (`.env`) | file copy | implemented — **policy violation, see below** |
| Redis | not backed up | acceptable: cache/rate-limit only, no durable state |
| Key material | **never backed up raw** — see rules below | — |

## Key material rules (hard requirements)

- `DRM_MASTER_KEY`, `DRM_SERVER_PRIVATE_KEY`, `ARTIFACT_SIGNING_PRIVATE_KEY`,
  `JWT_SECRET`, `YOOKASSA_SECRET_KEY`, `YOOKASSA_NOTIFICATION_PASSWORD`,
  OAuth client secrets, DB/Redis/S3 credentials **must never appear
  unencrypted in backups**.
- Recovery for DRM/artifact keys is **re-issue + re-sign**, not restore:
  - `DRM_SERVER_PRIVATE_KEY`: generate a new keypair (`pnpm drm:keygen`),
    rotate (`rotateServerSigningKey()` — the old public key stays trusted as
    PREVIOUS for existing leases); leases signed by the lost key are allowed
    to expire naturally (ADR-001) or are cut off by key revocation for
    compromise cases.
  - `ARTIFACT_SIGNING_PRIVATE_KEY`: generate a new keypair; per-seller
    `PublisherKey` rows are re-issued on next use and old signatures are
    re-signed (`signVersionArtifact` is idempotent per version) for versions
    that must stay verifiable.
  - `DRM_MASTER_KEY`: if lost, wrapped DEKs cannot be unwrapped and encrypted
    versions are permanently undecryptable — store one encrypted copy (e.g.
    age/age-encrypted file in the secrets vault) explicitly, or accept
    re-encryption of artifacts at re-publication. This trade-off must be an
    explicit operator decision, not an accident.
- `scripts/backup.sh` currently copies `.env` **unencrypted** into the backup
  directory — this violates the rule above. Until the script is fixed, the
  backup directory must be treated as secret-bearing: 0600/0700 perms,
  encrypted off-host shipping, and exclusion from any broader storage
  snapshot.

## Restore procedure (documented steps; drill not yet executed)

1. Provision a host with Docker; restore `.env` from the encrypted off-host
   copy (never from the plaintext local backup).
2. Start `postgres` (postgres:16-alpine) and `redis` from
   `docker-compose.prod.yml`.
3. Create the database:
   `docker exec -i mta-market-postgres psql -U mtamarket -c "CREATE DATABASE mtamarket;"`
4. Apply the latest dump:
   `cat backups/db_backup_<date>.sql | docker exec -i mta-market-postgres psql -U mtamarket -d mtamarket`
   (plain-SQL dump; restore into an empty DB).
5. Verify the contract signature matches the restored schema:
   `pnpm --filter @mta-market/server exec prisma db verify --db "<DATABASE_URL>"`.
6. Restore uploads/artifacts: untar `uploads_backup_<date>.tar.gz` into the
   storage path (or verify object-storage bucket contents).
7. Start backend + frontend (`docker compose -f docker-compose.prod.yml up -d`);
   check `/live` and `/ready` (DB must answer; Redis optional).
8. Smoke: log in, open catalog, trigger one DRM v2 activation against a test
   license; confirm the reconciliation summary endpoint answers.
9. Post-restore integrity: run one reconciliation cycle
   (`reconciliation:run`) and review the latest report for mismatches.

**Restore test schedule (policy):** quarterly drill on a scratch host; the
first executed drill must be recorded here with its date. No drill has been
executed as of 2026-09-09.

## WAL archiving (recommendation, not implemented)

For RPO < 24 h, enable PostgreSQL WAL archiving (`archive_mode = on`,
`archive_command` to the off-host bucket) plus `recovery_command`/
`pg_basebackup`-based restore. Not implemented in the current compose
topology; the 24 h RPO assumes plain daily dumps.

## Gaps vs target (2026-09-09)

1. Retention in `scripts/backup.sh` is **7 days**, not 30.
2. Backups stay on-host; no off-host copy job.
3. `.env` copied unencrypted.
4. No scheduling in the repo (cron is operator-managed, unwritten).
5. No executed restore drill; RTO 4 h is an estimate.
6. No WAL archiving (RPO is bounded by dump frequency).
