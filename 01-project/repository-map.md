# MTA Market — Repository Map

**Last updated:** 2026-09-07

---

## Project Structure

MTA Market consists of three separate repositories:

### 1. mta-market-site
**Purpose:** Web platform (backend + frontend)

**Location:** https://github.com/acc-holo-dev/mta-market-site

**Tech stack:**
- Backend: Node.js + Express + TypeScript + Prisma
- Frontend: Next.js 15 + React 19 + TailwindCSS
- Database: PostgreSQL 16
- Cache: Redis 7
- Storage: S3/R2

**Responsibilities:**
- User authentication & authorization
- Seller profiles & catalog
- Resource/service metadata
- Payments & financial ledger
- Moderation & admin
- DRM license server
- API endpoints

---

### 2. mta-market-module
**Purpose:** Native MTA client module (C++)

**Location:** https://github.com/acc-holo-dev/mta-market-module

**Tech stack:**
- C++20
- MTA Module SDK
- Cryptographic libraries (OpenSSL/libsodium)

**Responsibilities:**
- Installation identity & keypair generation
- License verification (signature checking)
- Artifact hash verification
- Local lease enforcement
- Communication with license API
- Anti-tamper checks

---

### 3. mta-market-document
**Purpose:** Technical documentation & specifications

**Location:** https://github.com/acc-holo-dev/mta-market-document

**Contents:**
- Architecture documentation
- API contracts
- DRM protocol specification
- Security requirements
- Production gates
- ADRs (Architecture Decision Records)
- Stage reports & audit findings

---

## Repository Boundaries

### What belongs where

| Concern | Repository |
|---|---|
| User accounts | mta-market-site |
| Seller dashboard | mta-market-site |
| Product catalog | mta-market-site |
| Payment processing | mta-market-site |
| License issuance | mta-market-site |
| Installation activation | mta-market-site |
| Moderation | mta-market-site |
| Admin tools | mta-market-site |
| | |
| Installation keypair | mta-market-module |
| License signature verification | mta-market-module |
| Artifact verification | mta-market-module |
| Local lease enforcement | mta-market-module |
| MTA runtime integration | mta-market-module |
| | |
| Architecture specs | mta-market-document |
| API contracts | mta-market-document |
| DRM protocol | mta-market-document |
| Security requirements | mta-market-document |
| Production readiness | mta-market-document |

---

## Cross-repository contracts

### DRM Protocol v2
Versioned protocol between site and module:
- Lease request/response format
- Signature algorithms
- Error codes
- Compatibility matrix

**Documented in:** `mta-market-document/03-features/drm/protocol-v2.md`

### Artifact Format
Resource package structure:
- Manifest schema
- Signature format
- Metadata fields

**Documented in:** `mta-market-document/03-features/artifacts/format.md`

---

## Development workflow

### Working on site features
1. Clone `mta-market-site`
2. Make changes
3. Update `mta-market-document` if API/protocol changes
4. Submit PR

### Working on module features
1. Clone `mta-market-module`
2. Make changes
3. Update `mta-market-document` if protocol changes
4. Test compatibility with site
5. Submit PR

### Working on documentation
1. Clone `mta-market-document`
2. Update specifications
3. Ensure status.md reflects implementation reality
4. Submit PR

---

## CI/CD coordination

### Site CI
- Lint, typecheck, build
- Unit & integration tests
- Protocol compatibility check (against documented version)

### Module CI
- Format, compile (Windows/Linux)
- Unit tests
- Protocol compatibility check (against documented version)

### Cross-repo compatibility
When protocol changes:
1. Update specification in `mta-market-document`
2. Version bump in both site and module
3. Compatibility matrix update
4. CI ensures both sides support the version

---

## Historical note

Previous repository names (deprecated):
- `mta-market` → renamed to `mta-market-site`
- `mta-guard-module` → renamed to `mta-market-module`

All documentation updated 2026-09-07 to reflect current names.
