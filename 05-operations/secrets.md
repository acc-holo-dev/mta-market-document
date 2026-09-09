status: current
version: 1.0
last_verified: 2026-09-09

# Secrets / key management (O-004)

Rules: secrets live in the environment (`.env` locally, container env /
secret manager in production), **never in the repository, never in logs**
(`apps/server/src/lib/logger.ts` redacts sensitive keys), never in backups
unencrypted ([backup.md](backup.md)). Production startup fails closed when a
required secret is absent or weak (`src/lib/startupValidation.ts`, A-012;
tested in `tests/startup-policy.test.ts` incl. exit non-zero).

## Key ownership table

| Secret / key | Owner (system) | Consumer (code) | Storage | Rotation procedure | Compromise response |
|---|---|---|---|---|---|
| `JWT_SECRET` (≥32 chars) | site | `src/lib/jwt.ts`, `src/lib/tokenSecurity.ts` | env only | Re-issue: set a new secret; all access tokens invalidate within `JWT_ACCESS_EXPIRY` (15 m default); refresh tokens are DB-backed session rows with stored hashes — invalidate all `Session` rows (logout-all) at rotation; users re-login | same as rotation + investigate refresh reuse (`reuseDetected` flags) |
| `JWT_ACCESS_EXPIRY` / `JWT_REFRESH_EXPIRY` | site | `src/lib/jwt.ts` | env (config, not secret) | n/a (version IDs: 15 m / 7 d defaults) | n/a |
| `DRM_SERVER_PRIVATE_KEY` (base64 PKCS8 Ed25519) | site (DRM v2) | `src/lib/drm/crypto.ts`, `src/routes/drm/v2.ts` | env only; public half in DB (`ServerSigningKey`) | **DRM rotate CLI**: `rotateServerSigningKey()` generates a new ACTIVE keypair (public stored, private returned once — install into env); previous key stays PREVIOUS/trusted (G-007). Required at startup in production | rotate immediately; for active compromise also revoke the key — leases verifiable only by a revoked key fail with `DRM_SERVER_KEY_REVOKED` |
| `DRM_MASTER_KEY` (base64, 32 bytes) | site (artifact encryption) | `src/lib/artifact/encryption.ts` (`loadMasterKey`) | env only; never exposed to clients (G-005) | No online rotation implemented (wrapped DEKs are bound to it). Rotation requires re-wrapping all `ArtifactEncryption` rows — treat as a migration, PLANNED tooling | **critical**: if exfiltrated, wrapped DEKs are decryptable server-side — rotate server + re-wrap; if lost, encrypted versions are unrecoverable (see [backup.md](backup.md)) |
| `ARTIFACT_SIGNING_PRIVATE_KEY` (base64 PKCS8 Ed25519) | site (artifact signing) | `src/lib/artifact/signing.ts` (`ensurePublisherKey`) | env only; public half in DB (`PublisherKey`) | Set a new key; stale per-seller `PublisherKey` rows are auto-revoked on next use; re-sign versions that must stay verifiable (`signVersionArtifact` is idempotent) | rotate + re-sign affected versions |
| `YOOKASSA_SHOP_ID` / `YOOKASSA_SECRET_KEY` | YooKassa merchant account | `src/lib/yookassa.ts` (HTTP Basic) | env only | Rotate in the **YooKassa merchant dashboard** (API keys), then update env and restart | revoke in dashboard, issue new key; review `Payment`/`ReconciliationReport` rows for foreign activity |
| `YOOKASSA_NOTIFICATION_PASSWORD` | site ↔ YooKassa webhook | `src/lib/yookassaWebhook.ts` (`verifyYooKassaAuth`) | env only | Set in the YooKassa dashboard (notification settings) + env restart | change password — forged webhooks then fail Basic verification (401) |
| `DISCORD_CLIENT_ID` / `DISCORD_CLIENT_SECRET` / `DISCORD_REDIRECT_URI` | Discord developer app | `src/lib/providers/discord.ts`, `src/routes/auth.ts` | env only | Rotate in Discord developer portal; required at startup in production | reset secret in portal; active sessions unaffected (access tokens are ours) |
| `YANDEX_CLIENT_ID` / `YANDEX_CLIENT_SECRET` / `YANDEX_REDIRECT_URI` | Yandex OAuth app | `src/lib/providers/yandex.ts` | env only | Yandex OAuth console | reset secret in console |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_REDIRECT_URI` | Google Cloud OAuth client | `src/lib/providers/google.ts` | env only | Google Cloud Console → Credentials | reset secret in console |
| `DATABASE_URL` (Postgres credentials) | site | `src/prisma/db.ts`, `prisma.config.ts` | env only | Rotate DB password in Postgres (`ALTER USER … PASSWORD`), update env/compose, rolling restart | revoke sessions from the leaked host, rotate password, check audit log |
| `REDIS_URL` | site | `src/lib/redis.ts` | env only | Rotate Redis password (`requirepass`/ACL), update env | rotate; rate-limit state is ephemeral |
| `S3_BUCKET` / `S3_REGION` / `S3_ACCESS_KEY` / `S3_SECRET_KEY` / `S3_ENDPOINT` | object storage (R2/S3) | `src/lib/s3.ts` | env only; required at startup when `S3_ENABLED=true` | Rotate the storage API token in the provider console, update env | revoke token, audit bucket access logs |
| `SMTP_USER` / `SMTP_PASS` (`EMAIL_ENABLED=true`) | mail provider | `src/lib/email.ts` | env only | Rotate in the mail provider console | revoke password |
| `CORS_ORIGINS`, `TRUST_PROXY`, `S3_SIGNED_URL_TTL` (300 s default, max 900), rate-limit maxima (`STRICT/STANDARD/AUTH_RATE_LIMIT_MAX` = 10/60/5), `RECONCILIATION_INTERVAL_MS` | site | `src/app.ts`, `src/lib/rateLimit.ts`, `src/jobs/reconciliation.ts` | env (config, not secrets) | n/a | n/a |

Version IDs for key material: `ServerSigningKey.id` (DRM leases carry
`serverKeyId`), `PublisherKey.id` (artifact signatures carry `keyId`),
`ArtifactEncryption.dekId` (per-version DEK). These identifiers are the
rotation/audit handles — logs and mismatch reports reference them.

## Storage rules

1. Env only. `.env` is gitignored; `.env.example` carries no real values.
2. Never in the database: the only key bytes stored in Postgres are **public**
   keys (`ServerSigningKey.publicKey`, `PublisherKey.publicKey`) and wrapped
   DEKs (`ArtifactEncryption`).
3. Never in logs: `logger.ts` redacts keys matching
   `access[_-]?token|refresh[_-]?token|password|secret|authorization|credentials|private[_-]?key|notification[_-]?password|cookie|bearer`
   (implemented; no dedicated test — see
   [../01-project/status.md](../01-project/status.md)).
4. Never in backups unencrypted: [backup.md](backup.md) (including the
   documented violation in the current `scripts/backup.sh`).
5. Client-side: the module's installation private key never leaves the
   machine (INV-010, `mta-market-module/source/drm/key_store.cpp`); the server
   never receives it.

## Revocation summary

| Key revoked/compromised | Immediate action | Residual window |
|---|---|---|
| Installation private key (module side) | revoke the installation (`revokeInstallation`) — management plane cut off immediately | existing leases run to natural expiry (ADR-001); faster cutoff = server key rotation |
| Server signing key | rotate; revoke if compromised | leases signed by a revoked key fail verification (`DRM_SERVER_KEY_REVOKED`) |
| JWT_SECRET | rotate + invalidate all sessions | ≤ 15 min (access token TTL) |
| Provider/OAuth/DB/S3 credentials | rotate at the provider; restart | none once rotated |
