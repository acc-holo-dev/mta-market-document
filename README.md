# MTA Market — Documentation & Audit Repository

**Project status:** MVP / Pre-Production  
**Main repo:** https://github.com/acc-holo-dev/mta-market  
**Guard module:** https://github.com/acc-holo-dev/mta-guard-module

---

## Overview

This repository contains comprehensive audit findings and project documentation for MTA Market — a DRM-protected marketplace for MTA:SA server resources.

**Contents:**

1. [MTA_MARKET_FULL_AUDIT.md](MTA_MARKET_FULL_AUDIT.md) — Complete technical audit (2750 lines)
2. [PROJECT_STATUS.md](PROJECT_STATUS.md) — Current implementation status & production gates

---

## Quick Links

- **Main implementation:** [github.com/acc-holo-dev/mta-market](https://github.com/acc-holo-dev/mta-market)
- **DRM client module:** [github.com/acc-holo-dev/mta-guard-module](https://github.com/acc-holo-dev/mta-guard-module)
- **Project documentation:** `/document` folder in main repo

---

## Audit Summary

### Status: MVP / Pre-Production

The audit identifies:
- ✅ **7 P0 issues resolved** (payment bypass, JWT secrets, token handling)
- ⏳ **10 P0 issues remaining** (DRM v2, financial ledger, observability)
- 📋 **29 production gates** before public launch

### Key Findings

**Strengths:**
- Solid modular monolith architecture
- Good domain model (Purchase ≠ License ≠ Payment)
- Real MTA integration via mta-guard-module
- Docker + CI/CD operational

**Critical gaps:**
- Production payment flow incomplete
- DRM v1 uses symmetric crypto (needs keypair-based v2)
- Financial ledger lacks double-entry bookkeeping
- No sandbox validation for malicious uploads
- Observability insufficient for production

---

## Implementation Progress

| Component | Status | Notes |
|---|:---:|---|
| Backend API | ✅ 90% | 38 endpoints operational |
| Frontend | ✅ 85% | Next.js 15 + TailwindCSS |
| Authentication | ✅ 95% | Discord OAuth2 + JWT |
| Payments | ⚠️ 40% | YooKassa skeleton only |
| DRM | ⚠️ 50% | v1 works, v2 needed |
| Financial Ledger | ⏳ 20% | Basic logging only |
| Moderation | ⏳ 30% | Manual workflow missing |
| Testing | ❌ 10% | No integration tests |
| Observability | ❌ 15% | Console logs only |

---

## Production Checklist

See [PROJECT_STATUS.md](PROJECT_STATUS.md) for complete gates.

**Security & Payments:**
- [ ] YooKassa production flow
- [ ] Payment event idempotency
- [ ] Input validation (Zod)
- [ ] S3 signed URLs
- [ ] Artifact signatures

**DRM v2:**
- [ ] Keypair-based licensing
- [ ] Public key distribution
- [ ] Signature verification (MTA Guard)

**Financial:**
- [ ] Double-entry ledger
- [ ] Seller payouts
- [ ] Reconciliation

**Safety:**
- [ ] Sandbox validation
- [ ] Malware scanning
- [ ] Moderation workflow

**Operations:**
- [ ] OpenTelemetry tracing
- [ ] Prometheus metrics
- [ ] Error tracking
- [ ] Integration + E2E tests

---

## Architecture

```
[Frontend: Next.js 15]
       ↓ HTTPS
[Nginx: SSL + rate limit]
       ↓
[Backend: Express + Prisma]
       ↓
[PostgreSQL 16] + [Redis 7] + [S3/R2]
```

**Tech stack:**
- Backend: Node.js 24 + Express + TypeScript
- Database: PostgreSQL 16 + Prisma 8
- Frontend: Next.js 15 + React 19 + TailwindCSS
- DevOps: Docker + GitHub Actions
- Integrations: Discord OAuth2, YooKassa, S3/R2

---

## Contributing

This is a documentation-only repository. For code contributions:
- Main implementation → [github.com/acc-holo-dev/mta-market](https://github.com/acc-holo-dev/mta-market)
- DRM module → [github.com/acc-holo-dev/mta-guard-module](https://github.com/acc-holo-dev/mta-guard-module)

For audit feedback or documentation improvements, open an issue in this repo.

---

## License

MIT License — see main repository for details.

---

**Last updated:** 2025-01-09
