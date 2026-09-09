status: current
version: 1.0
last_verified: 2026-09-09

# Release channels (R-003)

**Status: post-MVP.** Stable / beta / legacy channels are not implemented —
nothing in `mta-market-site` or `mta-market-module` models a channel today
(see [../01-project/status.md](../01-project/status.md)). This document fixes
the intended model now so that "PLANNED" has a concrete meaning, and documents
what **does** exist: the YANK mechanism as the current rollback primitive.

## Current primitive: YANK (implemented, tested)

- `ResourceVersion.releaseStatus`: `CANDIDATE → VERIFIED → PUBLISHED → YANKED`
  (`apps/server/src/prisma/contract.prisma`).
- `POST /admin/versions/:id/yank` (`apps/server/src/routes/admin.ts`) sets
  `YANKED` (I-005).
- Effect: **new DRM lease issuance is blocked** for the version
  (`DRM_INSUFFICIENT_CAPABILITIES`); already-issued leases keep their natural
  expiry (ADR-001, expire-at-lease-end data plane). Tested:
  `tests/block7.test.ts` ("verify advances CANDIDATE -> VERIFIED; yank blocks
  new lease issuance").
- YANK is the only rollback mechanism today: it stops new installations from
  obtaining a compromised/broken version but does not stop currently-leased
  ones (bounded by the ≤ 7-day lease TTL).

## Intended channel model (post-MVP)

| Channel | Meaning | Opt-in | Rollback |
|---|---|---|---|
| **stable** | default for every buyer/lease | none — everything defaults to stable | YANK + prefer-previous resolution |
| **beta** | pre-release versions of a resource | per **resource**, buyer-visible flag (explicit user choice) | version pinned back to stable by the seller/platform |
| **legacy** | old versions pinned for compatibility | pinned **by version range** (e.g. servers that cannot upgrade yet) | not applicable — legacy is the rollback target |

Design constraints (must hold when implemented):

1. Channels are **not** a DRM protocol change: leases already bind
   `resourceVersionId` + `artifactHash`, so channel membership resolves at
   lease issuance/renewal time only.
2. A beta→stable promotion is additive; a stable→YANK is the emergency path.
3. Channel state must be an explicit, queryable field on the release
   (`ResourceVersion`), not an inference from ordering.
4. The module treats "which version should I update to" as data returned by
   the server (`HeartbeatResponse.shouldUpdate/updateVersionId` — already in
   the wire types, currently always `false`/absent). Channel resolution plugs
   in there without touching the frozen protocol constants.
5. Compatibility matrix rules apply unchanged
   ([../02-architecture/compatibility-matrix.md](../02-architecture/compatibility-matrix.md)).

## Site/module release practice today

- Site: docker images from CI on `main` (`.github/workflows/ci.yml`, docker
  job); no staging channel, no semver discipline beyond 0.1.x.
- Module: built per-platform in its own CI (Linux, MinGW, MSVC); distribution
  of the binary is manual.
- Both are pre-1.0; "release" currently means a git tag, not a channel
  assignment.

## Open decisions (post-MVP, before implementation)

- Whether beta applies per resource, per seller, or platform-wide (current
  lean: per resource).
- Whether legacy pins are enforced at download time, lease time, or both.
- Whether channel changes require an audit-log entry (recommend: yes, via the
  existing `AuditLog` / `ModerationEvent` pattern).
