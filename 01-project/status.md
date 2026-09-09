status: current
version: 1.0
last_verified: 2026-09-09

# Статус проекта

Status is written from code and test runs, not from the roadmap. Every row was
re-verified on **2026-09-09** against the actual repositories:

- `mta-market-site` — apps/server: **222/222 tests passed (vitest, 21 files)**;
  `tsc --noEmit` clean. apps/web: `tsc --noEmit` clean.
- `mta-market-module` — DRM client unit tests: **ALL TESTS PASSED**
  (`make -f source/drm/Makefile test`, Linux x64, 2026-09-09).

Status vocabulary:

- **VERIFIED** — implemented and covered by a passing test run cited in Evidence.
- **IMPLEMENTED_UNVERIFIED** — implemented in code, no dedicated test observed.
- **PLANNED** — nothing implemented for it in code (post-MVP unless stated).

## Component | Status | Evidence | Last Verified

| Component | Status | Evidence (mta-market-site unless noted) | Last Verified |
|---|---|---|---|
| Auth: OAuth login (Discord, Yandex, Google) | VERIFIED | `src/lib/identityProvider.ts`, `src/lib/providers/{discord,yandex,google}.ts`, `src/routes/auth.ts`; tests `tests/identity.test.ts` | 2026-09-09 |
| Auth: identity linking / unlink (last-method protection) | VERIFIED | `src/routes/auth.ts` (/auth/:provider/link, /auth/identities); tests `tests/identity.test.ts` ("linking flow", "refuses to unlink the only login method") | 2026-09-09 |
| Auth: session rotation, refresh-token reuse detection, logout | VERIFIED | `src/routes/auth.ts` (POST /auth/refresh, `reuseDetected` family revocation); tests `tests/auth-flow.test.ts` ("replaying the OLD refresh token triggers reuse detection") | 2026-09-09 |
| Auth: JWT access/refresh, HttpOnly refresh cookie | VERIFIED | `src/lib/jwt.ts`, `src/lib/cookies.ts`, `src/lib/tokenSecurity.ts`; tests `tests/jwt.test.ts`, `tests/auth-flow.test.ts` | 2026-09-09 |
| Commerce: free/paid resource checkout, Order aggregate, completion | VERIFIED | `src/lib/commerce.ts` (createResourceCheckout / completeResourceOrderItem), `src/prisma/contract.prisma` (Order, OrderItem); tests `tests/commerce.test.ts` | 2026-09-09 |
| Commerce: discount campaigns (validation, immutable line snapshot) | VERIFIED | `src/lib/discount.ts`, `src/lib/commerce.ts`; test `tests/commerce.test.ts` ("applies a seller campaign and snapshots the line immutably") | 2026-09-09 |
| Commerce: discount usage atomicity under concurrency (C-007 CAS) | VERIFIED | `src/lib/discount.ts` (compare-and-set via `updateAndCount`); test `tests/commerce.test.ts` ("usage_limit=1 with 20 concurrent completions") | 2026-09-09 |
| Commerce: services (order, delivery, revision, accept; no DRM license, INV-015) | VERIFIED | `src/routes/services.ts`, `src/lib/commerce.ts` (createServiceCheckout, markServicePurchasePaid); tests `tests/services.test.ts` | 2026-09-09 |
| Payments: provider-neutral interface + registry (IPaymentProvider) | VERIFIED | `src/lib/paymentProvider.ts` (`IPaymentProvider`, `PaymentProviderRegistry`); exercised by `tests/payments-webhook.test.ts` | 2026-09-09 |
| Payments: YooKassa provider (create / refund / re-fetch) | VERIFIED (provider off + stubbed in CI) | `src/lib/yookassa.ts`, `src/lib/providers/payment-yookassa.ts`; `tests/payments-webhook.test.ts` (runs with `YOOKASSA_ENABLED=false`, stubbed re-fetch). No live-credential run recorded | 2026-09-09 |
| Payments: webhook verification (IP allowlist + Basic auth + provider re-fetch, A-010) | VERIFIED | `src/lib/yookassaWebhook.ts`, `src/routes/payments.ts`; tests `tests/payments-webhook.test.ts` (IP 403, forged Basic 401, replay ×10 → one effect, wrong amount/currency/purchase quarantined) | 2026-09-09 |
| Payments: state machine (E-003) | VERIFIED | `src/lib/paymentStateMachine.ts` (PENDING → SUCCEEDED → SETTLEMENT_PENDING → SETTLED; REFUNDED/PARTIALLY_REFUNDED; FAILED/CANCELED terminal); test `tests/ledger-refunds.test.ts` ("allows the canonical flow and rejects illegal jumps") | 2026-09-09 |
| Payments: refunds (INV-013 cap; K-004 license policy) | VERIFIED | `src/lib/refunds.ts`, `src/routes/payments.ts` (admin-only /payments/refunds); tests `tests/ledger-refunds.test.ts` (partial keeps license, full revokes, over-cap 409) | 2026-09-09 |
| Payments: production simulate path removed (A-005) | VERIFIED | `src/routes/payments.ts` (simulate only when `NODE_ENV !== "production"`); test `tests/app-security.test.ts` ("returns 404 for /payments/:id/simulate in a production-like environment") | 2026-09-09 |
| Ledger: double-entry LedgerAccount/LedgerEntry, INV-012 | VERIFIED | `src/lib/ledger.ts` (`postLedgerEntries` throws `LedgerUnbalancedError`); tests `tests/ledger-refunds.test.ts` ("settlement posts a balanced transaction", "rejects unbalanced postings (INV-012)", "zero-price settlement posts nothing") | 2026-09-09 |
| Ledger: refund postings balanced (F-003) | VERIFIED | `src/lib/refunds.ts` (DEBIT seller/platform, CREDIT platform_cash); covered by `tests/ledger-refunds.test.ts` refund cases | 2026-09-09 |
| DRM v2 (server): installation identity, challenge, lease, nonce replay protection | VERIFIED | `src/lib/drm/{protocol,crypto,service,types}.ts`, `src/routes/drm/v2.ts`; tests `tests/drm-v2.test.ts` (INV-007/INV-011, "rejects nonce reuse"), `tests/drm-crypto.test.ts` | 2026-09-09 |
| DRM v2 (server): DEK envelope release with possession proof (G-005) | VERIFIED | `src/lib/drm/service.ts` (issueVersionDek), `src/lib/artifact/encryption.ts`; tests `tests/drm-g6.test.ts` ("releases the DEK only to the lease holder with possession proof") | 2026-09-09 |
| DRM v2 (server): key rotation + trusted key window (G-007) | VERIFIED | `src/lib/drm/service.ts` (rotateServerSigningKey, getTrustedServerKeys); test `tests/drm-g6.test.ts` ("keeps the previous key trusted and lets new leases use the new key") | 2026-09-09 |
| DRM v2 (server): revocation policy (ADR-001), clock skew (90 s) | VERIFIED | `docs/adr/ADR-001-drm-lease-revocation.md` (site repo), `src/lib/drm/protocol.ts`; tests `tests/drm-g6.test.ts` ("runs the full cycle: … revoke -> denied", skew tests) | 2026-09-09 |
| DRM v1: deprecated — activate/verify return 410, license management kept (A-007) | VERIFIED | `src/routes/drm.ts`; test `tests/drm-v2.test.ts` ("v1 activation protocol is blocked with 410 (A-007)") | 2026-09-09 |
| Module DRM client (mta-market-module `source/drm/`) | VERIFIED (unit, Linux x64) | `source/drm/{http_client,key_store,license_client,ed25519,aead,json,base64}.{hpp,cpp}`; `make -f source/drm/Makefile test` → ALL TESTS PASSED (24 checks); inventory `docs/H-001-inventory.md` | 2026-09-09 |
| Module DRM client: end-to-end against a live server | PLANNED | No integration run recorded in `mta-market-module` (`other/tests/integration/README.md` describes the intended harness only) | 2026-09-09 |
| Module DRM client: Windows execution (DPAPI key store) | IMPLEMENTED_UNVERIFIED | `source/drm/key_store.hpp` (DPAPI path behind `_WIN32`); CI builds (`windows-mingw`, `windows-msvc` jobs); DRM unit tests not yet executed on Windows | 2026-09-09 |
| Moderation: seller/admin state machine (A-008, J-001) | VERIFIED | `src/lib/moderation.ts`, `src/routes/admin.ts`; tests `tests/moderation.test.ts`, `tests/block7.test.ts` | 2026-09-09 |
| Moderation: immutable event log (J-002) | VERIFIED | `src/prisma/contract.prisma` (ModerationEvent), `src/routes/admin.ts`; test `tests/block7.test.ts` ("records an event for every admin transition with actor and reason") | 2026-09-09 |
| Upload sandbox: static validation | VERIFIED | `src/lib/sandbox/static.ts`; tests `tests/sandbox-static.test.ts` (33 tests) | 2026-09-09 |
| Upload sandbox: Docker execution runner | IMPLEMENTED_UNVERIFIED | `src/lib/sandbox/runner.ts` (Docker-based, mock-capable implementation; no container-runtime test) | 2026-09-09 |
| Publication pipeline: validate → sign → dependency graph → publish gate | VERIFIED | `src/lib/artifact/{signing,dependencies}.ts`, `src/routes/admin.ts` (gate blocks unsigned/FAILED versions); tests `tests/publication-pipeline.test.ts`, `tests/block7.test.ts` | 2026-09-09 |
| Release lifecycle: CANDIDATE → VERIFIED → PUBLISHED → YANKED (I-005) | VERIFIED | `src/prisma/contract.prisma` (ResourceVersion.releaseStatus), `src/routes/admin.ts` (POST /admin/versions/:id/yank); test `tests/block7.test.ts` ("yank blocks new lease issuance") | 2026-09-09 |
| Artifact downloads: entitlement-gated, short-lived signed URLs / confined local paths (A-009) | VERIFIED | `src/routes/versions.ts` (GET …/download), `src/lib/s3.ts` (presigned TTL ≤ 900 s); tests `tests/download-auth.test.ts` | 2026-09-09 |
| Disputes: lifecycle, messages, events, attachments, admin resolution | VERIFIED | `src/routes/disputes.ts`, `src/prisma/contract.prisma` (Dispute, DisputeMessage, DisputeEvent, DisputeAttachment); test `tests/block7.test.ts` ("opens a dispute, freezes the target, records messages and events, resolves") | 2026-09-09 |
| Seller onboarding: apply → admin approve/reject → listing gate | VERIFIED | `src/routes/seller.ts`, `src/lib/permissions.ts`; test `tests/block7.test.ts` ("apply -> admin approve -> listing creation succeeds") | 2026-09-09 |
| Seller analytics (L-004) | PLANNED | No analytics endpoints in `src/routes/seller.ts`; no dedicated model in `src/prisma/contract.prisma` | 2026-09-09 |
| Observability: structured JSON logs with secret redaction (B-006) | IMPLEMENTED_UNVERIFIED | `src/lib/logger.ts` (`redact`, SENSITIVE_KEY regex); no dedicated redaction test observed | 2026-09-09 |
| Observability: request-id middleware + access logs (B-007) | VERIFIED | `src/middleware/requestId.ts`, `src/middleware/observability.ts`; tests `tests/block3-observability.test.ts` | 2026-09-09 |
| Observability: Prometheus `/metrics`, liveness `/live`, readiness `/ready` | VERIFIED | `src/lib/metrics.ts`, `src/app.ts`; tests `tests/block3-observability.test.ts` | 2026-09-09 |
| Reconciliation: 5-step cycle + daily scheduler (B-003, E-009) | VERIFIED | `src/lib/reconciliation/{service,internal,types}.ts`, `src/jobs/reconciliation.ts`; tests `tests/reconciliation.test.ts` | 2026-09-09 |
| Startup secret validation (A-012) | VERIFIED | `src/lib/startupValidation.ts`; tests `tests/startup-policy.test.ts` (incl. "missing secret exits non-zero") | 2026-09-09 |
| Frontend: 13 pages (catalog, resource, auth, dashboard, seller, admin, services, disputes) | IMPLEMENTED_UNVERIFIED | `apps/web/src/app/**/page.tsx` (Next.js 15.1 + React 19 + Tailwind + Zustand 5 + TanStack Query 5); type-check clean; no component/E2E tests | 2026-09-09 |
| Frontend: browser auth E2E (A-001 in a real browser) | PLANNED | No Playwright/E2E suite in the repo; the auth contract is covered server-side by `tests/auth-flow.test.ts` | 2026-09-09 |
| Tests (server) | VERIFIED | 21 files / 222 tests passed — `vitest run` against PostgreSQL 16 + Redis 7 (docker), 2026-09-09 | 2026-09-09 |
| Tests (module) | VERIFIED (unit, Linux x64) | `make -f source/drm/Makefile test` → ALL TESTS PASSED, 2026-09-09 | 2026-09-09 |
| CI (site repo): lint + type-check, backend/frontend builds, docker push on main | VERIFIED (jobs exist in workflow) | `.github/workflows/ci.yml` (lint, build-backend, build-frontend, docker) | 2026-09-09 |
| CI (site repo): test job, migration check, secret scan (O-005) | PLANNED | Not present in `.github/workflows/ci.yml`; being added separately | 2026-09-09 |
| CI (module repo): Linux + MinGW + MSVC build jobs | VERIFIED (jobs exist in workflow) | `mta-market-module/.github/workflows/ci.yml` | 2026-09-09 |
| Release channels: stable / beta / legacy (R-003) | PLANNED | No channel model in code; current rollback primitive is YANK — see [05-operations/release-channels.md](../05-operations/release-channels.md) | 2026-09-09 |
| Crypto payments (E-010 registry readiness) | PLANNED | `src/lib/paymentProvider.ts` registry admits new providers; no crypto provider implementation exists | 2026-09-09 |
| Backup automation matching O-003 target policy | IMPLEMENTED_UNVERIFIED | `scripts/backup.sh` (manual pg_dump + uploads tar + unencrypted `.env` copy, 7-day local retention); gaps documented in [05-operations/backup.md](../05-operations/backup.md) | 2026-09-09 |

## Production readiness

Not production-ready. Money, upload sandboxing and asymmetric DRM are
implemented and tested at the code level, but there is no deployed
staging/production environment, no browser E2E, no test/migration/secret-scan
CI gates, and no executed backup-restore drill. Gates live in
[production-readiness.md](production-readiness.md).

## Known honest gaps (2026-09-09)

- Domain IDs are UUID v4 (`@default(uuid())`); the validating middleware is
  still named `validateCuid` (`src/middleware/validateCuid.ts`) but validates
  the UUID format. Renaming is cosmetic debt only.
- `@prisma/client` is 7.10 while the Prisma CLI / `@prisma/orm-postgres`
  contract ORM (`db.orm`) are 8.0.0-rc.*. The 2026-09-07 schema-debt items
  (`Review.resourceId` Int, `FinancialTransaction.relatedPurchaseId` Int,
  duplicate `PurchaseStatus`) are resolved in the current contract.
- The YooKassa provider is exercised in CI with the provider disabled and a
  stubbed re-fetch; no live-credential run is recorded.
- Logger redaction (B-006) has no dedicated test; the regex list in
  `src/lib/logger.ts` is the only guard.
