status: current
version: 1.0
last_verified: 2026-09-09

# Artifact signing

Source of truth: `mta-market-site/apps/server/src/lib/artifact/signing.ts`
(pipeline + storage), `apps/server/src/lib/artifact/crypto.ts` (Ed25519 +
canonical JSON). Related: [format.md](format.md).

## Key model (MVP, documented honestly)

The **platform signs artifacts on behalf of the publisher** with a
server-held Ed25519 key stored in `ARTIFACT_SIGNING_PRIVATE_KEY` env
(base64 PKCS8 DER). The manifest records `publisherId = sellerId`; the
`PublisherKey` row binds the seller to the platform signing key. Per-seller
keys can replace this later without changing the verification contract.

`ensurePublisherKey(sellerId)` derives the public key (SPKI DER, base64) from
the configured private key, upserts an ACTIVE `PublisherKey` row per seller,
and revokes stale rows when the platform key is rotated. The private key is
never stored in the database.

## Signing pipeline (`signVersionArtifact(versionId, artifactBuffer)`)

```
artifact bytes
  → sha256 (artifactHash = manifest.sha256)
  → generateManifest(...)          # formatVersion 1, see format.md
  → canonical signing payload      # canonicalJSON(manifest fields + artifactHash)
  → Ed25519 sign (PKCS8 DER key)
  → attachSignatureMetadata(...)   # keyId, publicKey, signedAt
  → persist ArtifactSignature      # signature, manifestHash, artifactHash, manifest JSON
```

Signing payload rules (`createSigningPayload` in `artifact/crypto.ts`):

- canonical JSON: recursively sorted keys, no whitespace, UTF-8;
- the manifest's own hash (`hashManifest`) is bound into the payload —
  manifest integrity and artifact integrity are separate inputs;
- Ed25519 (pure, no prehash): `crypto.sign(null, payload, key)`.

## Verification (`verifyStoredArtifact(versionId, artifactBuffer)`)

1. Load the stored `ArtifactSignature` for the version — none → invalid.
2. The signing `PublisherKey` must exist and be `ACTIVE` — otherwise invalid
   (revoked key ⇒ verification failure, no grace window).
3. Recompute `sha256(artifactBuffer)` and verify against the signature over
   the canonical payload; `artifactHash !== manifest.sha256` fails first.
4. Result: `{valid, errors[], warnings[], signature{algorithm,keyId,verified},
   manifest{valid,formatVersion}}`.

Verification is the publication gate and the DRM lease chain: a lease's
`artifactHash` comes from the stored `ArtifactSignature`, so an unsigned or
re-signed-under-different-key version cannot issue leases for the old hash
(`activateLicense` requires a signature row — "Resource version not signed").

## Publication gate integration

- `hasValidSignature(versionId)` blocks admin publish of unsigned versions
  (`apps/server/src/routes/admin.ts`; test: "publish gate blocks an unsigned
  version (409)").
- Dependency-graph validation (`validateResourceDependencies`) runs at the
  same gate — see [compatibility.md](compatibility.md).
- Pipeline tested end-to-end in `tests/publication-pipeline.test.ts`
  ("creates a version from a stored artifact: validates, signs, records
  sandbox run").

## Rotation

Platform signing key rotation = set a new `ARTIFACT_SIGNING_PRIVATE_KEY`;
`ensurePublisherKey` then revokes the previous per-seller `PublisherKey` rows
and creates new ones. Old signatures stop verifying (key not ACTIVE) — re-sign
(`signVersionArtifact` is idempotent per version, it updates the existing
`ArtifactSignature` row) when old versions must remain downloadable.

## Test evidence

- `tests/artifact-crypto.test.ts` (20 tests) — keypair generation, sign,
  verify, tampered-message rejection, public key derivation.
- `tests/artifact-manifest.test.ts` (18 tests) — manifest generation,
  `attachSignatureMetadata`, structural validation.
- `tests/publication-pipeline.test.ts`, `tests/block7.test.ts` — gate wiring.
