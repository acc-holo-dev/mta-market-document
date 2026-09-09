status: current
version: 1.0
last_verified: 2026-09-09

# Artifact compatibility

Two mechanisms exist and must not be confused:

1. **Declared compatibility** — metadata a seller declares about the
   environment (MTA version, OS, architecture). Consumed by buyers; reported
   by the sandbox.
2. **Dependency graph validation** — a server-enforced publication gate on
   declared dependencies between resources.

## Declared compatibility object

From `apps/server/src/lib/artifact/types.ts` (`Compatibility`):

| Field | Type | Meaning |
|---|---|---|
| `mta.min` / `mta.max` | optional semver-ish strings | supported MTA:SA version range |
| `mta.tested` | string[] | versions actually tested by the publisher |
| `os` | `('linux' \| 'windows')[]` | target operating systems (default `["linux","windows"]`) |
| `architecture` | `('x64' \| 'x86')[]` | target architectures (default `["x64"]`) |
| `requiredModules` | string[] | other modules that must be present |
| `conflicts` | `{resourceName, resourceId?, reason}[]` | known-incompatible resources |

There is **no runtime enforcement** of the declared range today: the platform
records and exposes it, the sandbox reports it, and buyers read it. Honest
labeling: compatibility metadata is advisory (see status.md, sandbox row).

## Sandbox compatibility report

`apps/server/src/lib/sandbox/types.ts` (`CompatibilityReport`) is produced by
the sandbox service and stored per version run (persisted through
`SandboxRun` rows):

- `mtaVersion {min, max, tested[]}` — as detected/declared;
- `os[]`, `architecture[]` — as declared;
- `dependencies[]` (`{name, version?, optional}`) — as discovered from the
  artifact;
- `requiredModules[]`;
- `resourceType?`, `hasServer?`, `hasClient?`, `hasShared?` — artifact shape.

Sandbox run statuses: `success | failed | timeout | security_violation`.
Static validation (`SandboxStaticResult`) additionally reports
`suspiciousFiles`, archive structure, file counts/sizes. Security issues carry
`severity: critical | high | medium | low`.

Publication gate (`apps/server/src/routes/admin.ts`): a version whose sandbox
run FAILED cannot be published ("Version … failed sandbox/static validation
and cannot be published."). Docker execution itself is
IMPLEMENTED_UNVERIFIED (see [../01-project/status.md](../01-project/status.md));
static validation is fully tested (`tests/sandbox-static.test.ts`, 33 tests).

## Dependency graph rules (I-002)

`apps/server/src/lib/artifact/dependencies.ts` — `validateResourceDependencies(resourceSlug)`
runs before publication and resolves the graph against the **PUBLISHED**
catalog, transitively (declarations of targets are walked too):

| Error | Condition |
|---|---|
| `MISSING: … is not a published resource` | dependency slug has no published resource |
| `UNSUPPORTED: …` | declared dependency type does not match the target resource's type |
| `VERSION_CONFLICT: "slug" declared with conflicting minVersions` | two declarations on the same slug carry different `minVersion` values |
| `CIRCULAR` | the graph contains a cycle (including self-dependency) |

Version comparison is dotted-numeric (`versionSatisfiesMin`): `1.10.0 >= 1.9.0`
component-wise, missing components count as 0, empty `minVersion` always
satisfied. Result: `{ok, errors[], edges}`; `ok === false` blocks publication.

## Release lifecycle linkage (I-005)

`ResourceVersion.releaseStatus`: `CANDIDATE → VERIFIED → PUBLISHED → YANKED`.
YANKED blocks **new** DRM lease issuance and is the current rollback
primitive ([../05-operations/release-channels.md](../05-operations/release-channels.md)).
Existing leases keep their natural expiry (ADR-001).

## Test evidence

- `tests/block7.test.ts` — "rejects publication when a declared dependency is
  missing/circular", "verify advances CANDIDATE -> VERIFIED; yank blocks new
  lease issuance".
- `tests/sandbox-static.test.ts` — 33 static-validation tests.
- `tests/publication-pipeline.test.ts` — full gate wiring.
