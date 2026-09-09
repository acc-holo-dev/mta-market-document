status: current
version: 1.0
last_verified: 2026-09-09

# Architecture — three-repository system

Source of truth for code facts: the repositories themselves. This document
describes how the three repositories compose into one product. Related:
[repository-contract.md](repository-contract.md),
[compatibility-matrix.md](compatibility-matrix.md),
[decisions.md](decisions.md).

## Repositories

| Repository | Language / stack | Role |
|---|---|---|
| [mta-market-document](https://github.com/acc-holo-dev/mta-market-document) | Markdown | Canonical documentation: architecture, DRM protocol, artifact format, payment model, security requirements, operations. No implementation. |
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | TypeScript monorepo (pnpm + Turbo) | Marketplace: Express backend (`apps/server`), Next.js 15 + React 19 frontend (`apps/web`), PostgreSQL contract schema, Redis, S3/R2 storage. |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | C++ (CMake) | Native MTA:SA server module: Lua-binding SDK (`source/sdk/**`) + DRM client subsystem (`source/drm/**`) implementing the frozen DRM Protocol v2. |

`mta-market-document` is the only place where the cross-repo contracts (DRM
protocol v2, artifact format, API behavior) are specified. READMEs in the code
repositories start projects and link here; they do not restate contracts.

## Runtime boundaries

```
+--------------------+        HTTPS (REST /auth /resources /payments /drm/v2 …)
| MTA:SA player      | ------------------------------+               |
| (browser)          |                               |               |
+--------------------+                               v               v
                                        +-----------------------------+
+--------------------+   HTTPS /drm/v2/*|        apps/server          |
| MTA:SA server      | ----------------->|  Express (apps/server)      |
| mta-market-module  |                  |  - auth (JWT + cookie)      |
|  source/drm client |                  |  - commerce / payments      |
+--------------------+                  |  - ledger / reconciliation  |
                                        |  - DRM v2 protocol (frozen) |
                                        |  - artifact signing / DEK   |
                                        +------+-------------+--------+
                                               |             |
                                    PostgreSQL 16          Redis 7
                                    (contract.prisma,      (rate limiting)
                                     43 models)                 |
                                                               |  S3 / R2
                                        +----------------+     v
                                        | apps/web       |   object storage
                                        | Next.js 15 SSR |   (artifacts,
                                        | (thin frontend)|    signed URLs)
                                        +----------------+
```

- **Browser → server**: REST over HTTPS, cookie-based refresh token
  (HttpOnly), access token in memory. Same-origin behind nginx in production
  (CORS closed; `apps/server/src/app.ts`).
- **Module → server**: machine protocol `/drm/v2/*` only (frozen contract,
  `apps/server/src/lib/drm/protocol.ts`): installation registration requires a
  browser-authenticated owner; machine endpoints prove possession of the
  installation private key. TLS 1.2+ with system trust store
  (`source/drm/http_client.{hpp,cpp}`).
- **Server → PostgreSQL**: contract ORM `db.orm` (Prisma 8 RC,
  `apps/server/src/prisma/contract.prisma`, 43 models). Domain IDs are UUID
  strings.
- **Server → Redis**: rate limiting (fails open when Redis is absent),
  configurable via env.
- **Server → object storage**: artifacts; paid downloads only through
  short-lived presigned GET URLs (TTL 300 s default, max 900 s). Local-storage
  mode serves downloads through the authorized route, never static public
  paths.
- **Server → YooKassa**: outbound API (HTTP Basic) + inbound webhook
  (IP allowlist + Basic notification auth + provider re-fetch). No other
  payment provider is implemented; the registry (`IPaymentProvider`) keeps the
  seam for future ones.

## Responsibilities

| Concern | Owner |
|---|---|
| Accounts, sessions, OAuth identities, entitlements, licenses registry | site |
| Catalog, versions, moderation lifecycle, disputes, services | site |
| Checkout, payments, refunds, ledger, reconciliation | site |
| DRM Protocol v2 server side (leases, challenges, DEK envelope) | site |
| Artifact manifest/signing/encryption (publication pipeline) | site |
| DRM Protocol v2 client side: key store, challenge proof, lease verification/renewal, DEK fetch + payload decryption | module |
| MTA ABI, Lua bindings, runtime (timers, scheduler) | module |
| Contracts and specs (DRM v2, artifact format, threat model, operations) | document |

The module never persists its private key off the machine and never talks to
the payment layer; the server never learns the installation private key
(INV-010).

## Dependency direction

```
module  --implements-->  DRM Protocol v2  <--implements--  server
                          (spec: 03-features/drm/protocol-v2.md;
                           frozen code: apps/server/src/lib/drm/protocol.ts)
```

Either side may implement ahead of the other only within the frozen protocol
version. Changing any constant in `protocol.ts` requires a protocol version
bump (v3) and a new compatibility-matrix row — never an in-place edit.

## Deployment topology

- docker-compose (dev) / docker-compose.prod + nginx (prod): reverse proxy
  terminates TLS, `/api/` routes to the backend, exactly one trusted proxy hop
  (`trust proxy = 1`) so webhook IP allowlisting and rate limiting see the
  real client address.
- The MTA module ships as a binary (`base.so` / `base.dll`) installed into an
  MTA server's `modules/` directory; it is distributed outside the site's
  release pipeline today (release channels are post-MVP, see
  [05-operations/release-channels.md](../05-operations/release-channels.md)).

## What is deliberately out of scope

- DRM as absolute protection: the model is cost-raising (mass copying becomes
  expensive), not unbreakable. Residual risk: memory of a running MTA server.
- DRM encrypts Lua scripts, not DFF/TXD assets (ADR-005).
- Service orders never require a DRM license (INV-015).
