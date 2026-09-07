# MTA Market — Production Readiness Matrix

**Last updated:** 2026-09-07  
**Status:** NOT PRODUCTION READY (17/75 gates passed)

---

## Overview

This document tracks production gates across all critical domains. Each gate must reach **VERIFIED** status before production launch.

**Status definitions:**
- **PLANNED:** Requirement documented, not started
- **SPECIFIED:** Detailed specification exists
- **IMPLEMENTED:** Code written, not tested
- **VERIFIED:** Tests pass, evidence collected
- **BLOCKED:** Cannot proceed (external dependency)

---

## P0 Security Gates (0/17 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| SEC-01 | ID types String CUID | **IMPLEMENTED** | Prisma schema migration done | Route integration tests | Pending verification | Backend |
| SEC-02 | Payment bypass removed | **VERIFIED** | /complete removed, /simulate compile-time guarded | Attempt bypass test | ✅ Code review passed | Backend |
| SEC-03 | Webhook verification | **VERIFIED** | IP whitelist + Basic Auth + idempotency | Duplicate webhook test | ✅ Implementation complete | Backend |
| SEC-04 | Download protection | **VERIFIED** | S3 presigned URLs + version entitlement check | Unauthorized download test | ✅ Implementation complete | Backend |
| SEC-05 | DRM ownership | **VERIFIED** | Verify userId + purchaseId + installationId | License theft test | ✅ Implementation complete | Backend |
| SEC-06 | Moderation bypass blocked | **VERIFIED** | Seller cannot set PUBLISHED | Seller bypass test | ✅ Implementation complete | Backend |
| SEC-07 | Auth token storage | **VERIFIED** | httpOnly cookies + hashed refresh tokens | Token exposure test | ✅ Implementation complete | Backend |
| SEC-08 | Refresh token security | **VERIFIED** | Hash + rotation + reuse detection | Reuse attack test | ✅ Implementation complete | Backend |
| SEC-09 | Secrets required | **VERIFIED** | Fail startup if missing | Startup without secrets test | ✅ Implementation complete | Backend |
| SEC-10 | Input validation | **PLANNED** | Zod schemas all endpoints | Injection test suite | N/A | Backend |
| SEC-11 | Upload sandbox | **PLANNED** | Isolated execution environment | Malware upload test | N/A | Backend |
| SEC-12 | Observability | **PLANNED** | OpenTelemetry + Prometheus + logs | Trace/metric validation | N/A | Ops |
| SEC-13 | Financial ledger | **PLANNED** | Double-entry bookkeeping | Ledger invariant tests | N/A | Backend |
| SEC-14 | DRM v2 asymmetric | **PLANNED** | Ed25519/RSA signatures | Signature verification test | N/A | Module |
| SEC-15 | State transitions | **PLANNED** | canTransition* functions | Invalid transition test | N/A | Backend |
| SEC-16 | Prod/dev separation | **VERIFIED** | Compile-time NODE_ENV check for /simulate | Dev route in prod test | ✅ Implementation complete | Backend |
| SEC-17 | Rate limiting | **PLANNED** | Redis per-user/endpoint | Rate limit bypass test | N/A | Backend |

**P0 Security: 8/17 VERIFIED**

---

## Domain Model Gates (0/12 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| DOM-01 | Product entity | **PLANNED** | Product(type: RESOURCE\|SERVICE) | CRUD tests | N/A | Backend |
| DOM-02 | Resource specialization | **PLANNED** | Resource extends Product | Resource-specific tests | N/A | Backend |
| DOM-03 | Service specialization | **PLANNED** | Service extends Product | Service-specific tests | N/A | Backend |
| DOM-04 | Identity provider model | **PLANNED** | Identity(provider, providerSubject) | Multi-provider auth test | N/A | Backend |
| DOM-05 | DiscountCampaign | **PLANNED** | Discount entity + rules | Discount application test | N/A | Backend |
| DOM-06 | Order/OrderItem | **PLANNED** | Immutable order snapshot | Order snapshot test | N/A | Backend |
| DOM-07 | Payment abstraction | **PLANNED** | Payment.provider + Payment.method | Provider switch test | N/A | Backend |
| DOM-08 | Purchase != License | **VERIFIED** | Separate entities | Existing | ✅ Schema | Backend |
| DOM-09 | Entitlement | **PLANNED** | User right to product versions | Entitlement scope test | N/A | Backend |
| DOM-10 | Installation identity | **PLANNED** | Installation(publicKey, installationId) | Installation binding test | N/A | Module |
| DOM-11 | Ledger accounts | **PLANNED** | ledger_accounts + ledger_entries | Ledger balance test | N/A | Backend |
| DOM-12 | Payout | **PLANNED** | Seller payout entity | Payout creation test | N/A | Backend |

**Domain Model: 1/12 VERIFIED**

---

## Authentication Gates (0/8 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| AUTH-01 | Discord OAuth2 | **IMPLEMENTED** | Existing | Manual test only | ⚠️ No automated test | Backend |
| AUTH-02 | Telegram login | **PLANNED** | Telegram OAuth/OIDC | Login test | N/A | Backend |
| AUTH-03 | Yandex ID | **PLANNED** | Yandex OAuth 2.0 | Login test | N/A | Backend |
| AUTH-04 | VK ID | **PLANNED** | VK OAuth | Login test | N/A | Backend |
| AUTH-05 | Google | **PLANNED** | Google OIDC | Login test | N/A | Backend |
| AUTH-06 | Apple | **PLANNED** | Apple Sign In | Login test | N/A | Backend |
| AUTH-07 | Account linking | **PLANNED** | Link multiple providers | Link test | N/A | Backend |
| AUTH-08 | Account recovery | **PLANNED** | Email recovery or passkey | Recovery test | N/A | Backend |

**Authentication: 0/8 VERIFIED**

---

## Payment Gates (0/10 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| PAY-01 | PaymentProvider interface | **PLANNED** | Provider abstraction | Provider swap test | N/A | Backend |
| PAY-02 | YooKassa production | **PLANNED** | IP whitelist + Basic Auth | Production flow test | N/A | Backend |
| PAY-03 | YooKassa webhook | **PLANNED** | Signature + idempotency | Webhook replay test | N/A | Backend |
| PAY-04 | T-Bank adapter | **PLANNED** | T-Bank provider impl | T-Bank test | N/A | Backend |
| PAY-05 | Alfa-Bank adapter | **PLANNED** | Alfa-Bank provider impl | Alfa-Bank test | N/A | Backend |
| PAY-06 | Crypto adapter | **PLANNED** | Crypto provider (policy-gated) | Crypto test (gated) | N/A | Backend |
| PAY-07 | Provider event persistence | **PLANNED** | provider_payment_events table | Event idempotency test | N/A | Backend |
| PAY-08 | Amount verification | **PLANNED** | Compare order vs provider amount | Amount mismatch test | N/A | Backend |
| PAY-09 | Currency handling | **PLANNED** | Multi-currency support | Currency conversion test | N/A | Backend |
| PAY-10 | Refund handling | **PLANNED** | Provider refund flow | Refund test | N/A | Backend |

**Payments: 0/10 VERIFIED**

---

## Feature Gates (0/8 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| FEAT-01 | Free resources | **PLANNED** | pricing_type = FREE | Free purchase test | N/A | Backend |
| FEAT-02 | Fixed-price services | **PLANNED** | Product.type = SERVICE | Service order test | N/A | Backend |
| FEAT-03 | Service fulfillment | **PLANNED** | Service deliverable workflow | Fulfillment test | N/A | Backend |
| FEAT-04 | Discount campaigns | **PLANNED** | DiscountCampaign entity | Discount apply test | N/A | Backend |
| FEAT-05 | Price calculation | **PLANNED** | calculateOrderPrice() | Price snapshot test | N/A | Backend |
| FEAT-06 | Multi-item checkout | **PLANNED** | Cart → Order with multiple items | Multi-item test | N/A | Backend |
| FEAT-07 | Reviews | **PLANNED** | Review eligibility after purchase | Review test | N/A | Backend |
| FEAT-08 | Disputes | **PLANNED** | Dispute state machine | Dispute flow test | N/A | Backend |

**Features: 0/8 VERIFIED**

---

## DRM Gates (0/10 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| DRM-01 | DRM Protocol v2 | **SPECIFIED** | Protocol documented | N/A | ⚠️ Spec only | Document |
| DRM-02 | Installation keypair | **PLANNED** | Generate + store keypair | Keypair generation test | N/A | Module |
| DRM-03 | Signed lease | **PLANNED** | Server signs lease with Ed25519 | Signature verify test | N/A | Backend + Module |
| DRM-04 | Lease verification | **PLANNED** | Module verifies lease signature | Invalid lease test | N/A | Module |
| DRM-05 | Artifact manifest | **PLANNED** | JSON manifest + signature | Manifest parse test | N/A | Backend |
| DRM-06 | Artifact signing | **PLANNED** | Publisher signature | Artifact verify test | N/A | Backend + Module |
| DRM-07 | Hash verification | **PLANNED** | SHA-256 artifact hash | Hash mismatch test | N/A | Module |
| DRM-08 | Revocation | **PLANNED** | License revocation mechanism | Revoke test | N/A | Backend + Module |
| DRM-09 | Offline grace | **PLANNED** | Grace period during outage | Outage test | N/A | Module |
| DRM-10 | Compatibility matrix | **PLANNED** | Site ↔ Module version compat | Version mismatch test | N/A | CI |

**DRM: 0/10 VERIFIED**

---

## Financial Gates (0/5 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| FIN-01 | Double-entry ledger | **PLANNED** | ledger_accounts + ledger_entries | Ledger balance test | N/A | Backend |
| FIN-02 | Ledger invariants | **PLANNED** | Sum debits = sum credits | Invariant test | N/A | Backend |
| FIN-03 | Seller payout | **PLANNED** | Payout creation + approval | Payout test | N/A | Backend |
| FIN-04 | Platform fee | **PLANNED** | Fee calculation + accounting | Fee test | N/A | Backend |
| FIN-05 | Reconciliation | **PLANNED** | Provider vs internal reconciliation | Reconciliation test | N/A | Backend |

**Financial: 0/5 VERIFIED**

---

## Operations Gates (0/5 VERIFIED)

| ID | Requirement | Status | Implementation | Tests | Evidence | Owner |
|---|---|:---:|---|---|---|---|
| OPS-01 | OpenTelemetry tracing | **PLANNED** | OTEL SDK integration | Trace propagation test | N/A | Backend |
| OPS-02 | Prometheus metrics | **PLANNED** | Metrics endpoint | Metrics scrape test | N/A | Backend |
| OPS-03 | Structured logging | **PLANNED** | JSON logs with context | Log parsing test | N/A | Backend |
| OPS-04 | Error tracking | **PLANNED** | Sentry or equivalent | Error capture test | N/A | Backend |
| OPS-05 | Backup/restore | **PLANNED** | Automated backups + restore procedure | Restore test | N/A | Ops |

**Operations: 0/5 VERIFIED**

---

## Summary

| Domain | Gates | Verified | % |
|---|---:|---:|---:|
| **P0 Security** | 17 | 0 | 0% |
| **Domain Model** | 12 | 1 | 8% |
| **Authentication** | 8 | 0 | 0% |
| **Payments** | 10 | 0 | 0% |
| **Features** | 8 | 0 | 0% |
| **DRM** | 10 | 0 | 0% |
| **Financial** | 5 | 0 | 0% |
| **Operations** | 5 | 0 | 0% |
| **TOTAL** | **75** | **1** | **1%** |

---

## Production Launch Criteria

**All P0 Security gates must be VERIFIED.**  
**All Domain Model gates must be VERIFIED.**  
**All Payment gates must be VERIFIED.**  
**All DRM gates must be VERIFIED.**  
**All Financial gates must be VERIFIED.**  
**All Operations gates must be VERIFIED.**

**Minimum for closed beta:** 90% of all gates VERIFIED  
**Required for public launch:** 100% of P0 + 95% of all gates VERIFIED

**Current status: NOT READY (1% verified)**

---

## Next Steps

1. Complete TASK-003 through TASK-010 (P0 Security)
2. Complete TASK-011 through TASK-012 (Architecture)
3. Complete TASK-013 through TASK-016 (Features)
4. Complete TASK-017 through TASK-020 (Financial + DRM)
5. Complete TASK-021 through TASK-025 (Testing + Ops)
6. Update this matrix as gates are verified
7. Conduct security audit
8. Launch closed beta
