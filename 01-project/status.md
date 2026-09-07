# MTA Market — Project Status

**Last updated:** 2026-09-07  
**Current phase:** MVP / Pre-Production  
**Status:** NOT PRODUCTION READY

---

## Executive Summary

MTA Market is a DRM-protected marketplace for MTA:SA server resources and services. The project has a working MVP implementation with core features (backend API, frontend, Docker deployment) but **requires substantial security, architectural, and feature work before production launch**.

**Key metrics:**

- ✅ Backend API skeleton operational
- ✅ Frontend pages (Next.js 15)
- ✅ Docker + CI/CD pipeline
- ❌ **17 P0 security issues remain unresolved**
- ❌ ID type inconsistency (Int vs String CUID)
- ❌ Payment bypass vulnerabilities exist
- ❌ No provider-neutral authentication
- ❌ Free resources not implemented
- ❌ Services not implemented
- ❌ Discount system not implemented
- ❌ DRM v2 not implemented
- ❌ Financial ledger incomplete
- ❌ No sandbox validation
- ❌ No observability infrastructure

---

## Repository Structure

| Repository | Purpose | Status |
|---|---|---|
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | Backend + Frontend | MVP (incomplete) |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | C++ DRM client | PoC (needs v2) |
| [mta-market-document](https://github.com/acc-holo-dev/mta-market-document) | Documentation | Current |

---

## ✅ What Works (Partially)

### Backend API
- **Authentication:** Discord OAuth2 only (needs multi-provider)
- **Resources:** CRUD operations (needs Product/Resource split)
- **Purchases:** Basic flow (has payment bypass vulnerabilities)
- **DRM:** v1 prototype (symmetric crypto, needs v2)
- **Admin:** Basic moderation (seller can bypass)

### Frontend
- **Pages:** Home, catalog, resource detail, dashboard
- **UI:** TailwindCSS + React 19

### Infrastructure
- **Database:** PostgreSQL 16 + Prisma (Int IDs, needs String CUID)
- **Cache:** Redis 7
- **Deployment:** Docker Compose

---

## ❌ Critical P0 Security Issues (UNRESOLVED)

### P0-01: ID Type Consistency
**Status:** ❌ NOT FIXED  
**Issue:** Schema uses `Int @id @default(autoincrement())` instead of `String @id @default(cuid())`  
**Risk:** parseInt() attacks, type confusion, route parameter vulnerabilities  
**Required:** Migrate all domain IDs to String CUID

### P0-02: Payment Bypass
**Status:** ❌ NOT FIXED  
**Issue:** Endpoints exist that can mark purchases complete without verified payment  
**Risk:** Free access to paid resources  
**Required:** Remove all dev/simulate payment completion routes from production build

### P0-03: YooKassa Webhook Security
**Status:** ❌ NOT FIXED  
**Issue:** Webhook verification incomplete, no idempotency table  
**Risk:** Duplicate charges, replay attacks, amount tampering  
**Required:** Provider event verification + idempotent processing with `provider_payment_events` table

### P0-04: Download Protection
**Status:** ❌ NOT FIXED  
**Issue:** Artifact downloads not protected with signed URLs  
**Risk:** Unauthorized access to paid resources  
**Required:** S3/R2 presigned URLs with entitlement check

### P0-05: DRM Activation Ownership
**Status:** ❌ NOT FIXED  
**Issue:** License activation does not verify purchase ownership  
**Risk:** License theft  
**Required:** Verify userId + purchaseId + installationId binding

### P0-06: Seller Moderation Bypass
**Status:** ❌ NOT FIXED  
**Issue:** Seller can set resource status to PUBLISHED directly  
**Risk:** Unmoderated malicious content  
**Required:** Block seller status transitions to PUBLISHED

### P0-07: Auth Token Storage
**Status:** ❌ NOT FIXED  
**Issue:** Tokens may be exposed in URLs or localStorage  
**Risk:** Token theft, XSS attacks  
**Required:** Refresh tokens in httpOnly cookies, access tokens memory-only

### P0-08: Refresh Token Security
**Status:** ❌ NOT FIXED  
**Issue:** Refresh tokens not hashed in DB, no rotation  
**Risk:** Token reuse attacks  
**Required:** Store token hash, implement rotation + reuse detection

### P0-09: Secrets Management
**Status:** ❌ NOT FIXED  
**Issue:** Application may start with default secrets  
**Risk:** Production compromise  
**Required:** Fail startup if required secrets missing in production

### P0-10: Input Validation
**Status:** ❌ PARTIAL  
**Issue:** Not all endpoints have Zod validation  
**Risk:** Injection attacks, data corruption  
**Required:** Zod schemas on all input

### P0-11: File Upload Sandbox
**Status:** ❌ NOT IMPLEMENTED  
**Issue:** Seller uploads executed without sandbox validation  
**Risk:** Malware, exploits, server compromise  
**Required:** Isolated sandbox execution before artifact publication

### P0-12: Observability
**Status:** ❌ NOT IMPLEMENTED  
**Issue:** No structured logging, tracing, or metrics  
**Risk:** Unable to detect/diagnose production issues  
**Required:** OpenTelemetry + Prometheus + structured logs

### P0-13: Financial Ledger
**Status:** ❌ INCOMPLETE  
**Issue:** No double-entry bookkeeping  
**Risk:** Money loss, reconciliation failures  
**Required:** Proper ledger_accounts + ledger_entries + transactions

### P0-14: DRM v1 Symmetric Crypto
**Status:** ❌ NEEDS V2  
**Issue:** Current DRM uses symmetric keys  
**Risk:** Key distribution, revocation issues  
**Required:** Asymmetric signing (Ed25519/RSA) in DRM v2

### P0-15: State Machine Validation
**Status:** ❌ NOT IMPLEMENTED  
**Issue:** No transition validation for stateful entities  
**Risk:** Invalid state changes  
**Required:** `canTransition*` functions for all stateful entities

### P0-16: Production/Dev Separation
**Status:** ❌ NOT ENFORCED  
**Issue:** Dev simulation endpoints may be accessible in production  
**Risk:** Bypass security controls  
**Required:** Compile-time removal of dev routes when NODE_ENV=production

### P0-17: Rate Limiting
**Status:** ❌ INCOMPLETE  
**Issue:** No per-user or per-endpoint rate limits  
**Risk:** API abuse, DDoS  
**Required:** Redis-backed rate limiting per user/endpoint

---

## ⏳ Missing Core Features (MVP Requirements)

### Authentication
- ❌ Multi-provider identity (only Discord exists)
- ❌ Identity model (provider-neutral)
- ❌ Telegram login
- ❌ Yandex ID
- ❌ VK ID
- ❌ Google
- ❌ Apple
- ❌ Account recovery mechanism

### Products
- ❌ Product entity (Resource/Service split)
- ❌ Free resources (pricing_type = FREE)
- ❌ Services (Product.type = SERVICE)
- ❌ Service fulfillment workflow
- ❌ Service deliverables

### Discounts
- ❌ DiscountCampaign entity
- ❌ Seller-controlled discounts
- ❌ Immutable price snapshots in Order
- ❌ Discount validation & constraints

### Orders
- ❌ Cart/CartItem (ephemeral)
- ❌ Order/OrderItem (immutable snapshot)
- ❌ Multi-item checkout
- ❌ Price calculation service

### Payments
- ❌ PaymentProvider interface
- ❌ Provider-agnostic domain model
- ❌ T-Bank adapter (architecture-ready)
- ❌ Alfa-Bank adapter (architecture-ready)
- ❌ Crypto adapter (policy-gated)

### Financial
- ❌ Double-entry ledger
- ❌ Seller payout reconciliation
- ❌ Platform fee accounting
- ❌ Reconciliation worker
- ❌ Audit trail

### DRM
- ❌ DRM Protocol v2 (asymmetric)
- ❌ Installation identity (keypair-based)
- ❌ Signed lease format
- ❌ Artifact manifest
- ❌ Publisher signatures
- ❌ Compatibility matrix
- ❌ Update signatures
- ❌ Rollback mechanism

### Moderation
- ❌ State machine enforcement
- ❌ Moderation event history
- ❌ Manual review workflow
- ❌ DMCA takedown process

### Safety
- ❌ Sandbox validation
- ❌ Static analysis
- ❌ Malware scanning
- ❌ Archive bomb protection
- ❌ Path traversal protection

### Operations
- ❌ OpenTelemetry tracing
- ❌ Prometheus metrics
- ❌ Structured logging (JSON)
- ❌ Error tracking (Sentry)
- ❌ Uptime monitoring

### Testing
- ❌ Integration tests (payment flow)
- ❌ E2E tests (buyer/seller journeys)
- ❌ Compatibility tests (module ↔ site)
- ❌ Load tests (webhook handling)
- ❌ Security tests (OWASP Top 10)

---

## 📊 Implementation Progress

| Domain | Status | Completion |
|---|:---:|:---:|
| **Backend Core** | ⚠️ | 30% |
| Authentication | ⚠️ | 20% (Discord only) |
| Authorization | ❌ | 10% (role-only) |
| Domain Model | ⚠️ | 40% (missing Product/Identity/Discount/Order) |
| **Features** | ❌ | 15% |
| Resources (paid) | ⚠️ | 50% (security issues) |
| Free Resources | ❌ | 0% |
| Services | ❌ | 0% |
| Discounts | ❌ | 0% |
| **Payments** | ❌ | 20% |
| YooKassa | ⚠️ | 30% (webhook insecure) |
| Provider abstraction | ❌ | 0% |
| **Financial** | ❌ | 15% |
| Ledger | ❌ | 20% (basic logging) |
| Reconciliation | ❌ | 0% |
| Payouts | ❌ | 0% |
| **DRM** | ⚠️ | 25% |
| DRM v1 (PoC) | ⚠️ | 60% (symmetric, insecure) |
| DRM v2 (production) | ❌ | 0% |
| **Security** | ❌ | 20% |
| Input validation | ⚠️ | 30% |
| Upload sandbox | ❌ | 0% |
| Artifact signing | ❌ | 0% |
| **Operations** | ❌ | 10% |
| Observability | ❌ | 5% (console logs only) |
| Testing | ❌ | 10% (no integration/e2e) |

**Overall: ~20% production-ready**

---

## 🚫 Production Blockers

**DO NOT launch until ALL of these are VERIFIED:**

### Security Gates
- [ ] P0-01: ID types migrated to String CUID
- [ ] P0-02: Payment bypass removed
- [ ] P0-03: YooKassa webhook verified + idempotent
- [ ] P0-04: Downloads protected with signed URLs
- [ ] P0-05: DRM activation ownership verified
- [ ] P0-06: Seller moderation bypass blocked
- [ ] P0-07: Auth tokens not in URLs/localStorage
- [ ] P0-08: Refresh tokens hashed + rotated
- [ ] P0-09: Secrets required on startup
- [ ] P0-10: All endpoints have input validation
- [ ] P0-11: Upload sandbox operational
- [ ] P0-12: Observability infrastructure live
- [ ] P0-13: Financial ledger correct
- [ ] P0-14: DRM v2 implemented
- [ ] P0-15: State transitions validated
- [ ] P0-16: Dev routes removed from production
- [ ] P0-17: Rate limiting enforced

### Feature Gates
- [ ] Multi-provider authentication (Telegram, Yandex, VK, Google)
- [ ] Free resources functional
- [ ] Discounts functional
- [ ] Services functional (if enabled)
- [ ] PaymentProvider abstraction
- [ ] Order/OrderItem model
- [ ] Artifact signing verified
- [ ] Compatibility matrix tested
- [ ] Update + rollback tested

### Financial Gates
- [ ] Double-entry ledger verified
- [ ] Reconciliation worker tested
- [ ] Seller payout flow tested
- [ ] Platform fee accounting correct

### Operational Gates
- [ ] Integration tests pass
- [ ] E2E tests pass
- [ ] Compatibility tests pass (module ↔ site)
- [ ] Load tests pass
- [ ] Monitoring/alerting active
- [ ] Backup/restore tested
- [ ] Incident response documented

---

## 🛣️ Roadmap

### Phase 0: Documentation & Cleanup (Current)
- ✅ Synchronize repository names
- ⏳ Update status documentation
- ⏳ Create production readiness matrix

### Phase 1: P0 Security (Next)
**Tasks:** TASK-003 through TASK-010  
**Duration:** 4-6 weeks  
**Outcome:** Core security issues resolved

### Phase 2: Architecture Refactor
**Tasks:** TASK-011, TASK-012  
**Duration:** 2-3 weeks  
**Outcome:** Provider abstractions in place

### Phase 3: New Features
**Tasks:** TASK-013 through TASK-016  
**Duration:** 4-5 weeks  
**Outcome:** Free resources, discounts, services

### Phase 4: Financial & DRM
**Tasks:** TASK-017 through TASK-020  
**Duration:** 6-8 weeks  
**Outcome:** Production-grade DRM v2 + ledger

### Phase 5: Testing & Ops
**Tasks:** TASK-021 through TASK-025  
**Duration:** 3-4 weeks  
**Outcome:** Full test coverage + observability

### Phase 6: Closed Beta
**Duration:** 4-6 weeks  
**Outcome:** Real-world validation with 10-20 trusted sellers

### Phase 7: Public Launch
**Outcome:** Production-ready marketplace

**Estimated time to production:** 5-7 months from 2026-09-07

---

## 📞 Contact

For security issues: security@mtamarket.com (configure real email)  
For contributions: See [CONTRIBUTING.md](../06-development/contributing.md)

---

**Status legend:**
- ✅ Complete and verified
- ⏳ In progress
- ⚠️ Partial / has issues
- ❌ Not started or blocked
