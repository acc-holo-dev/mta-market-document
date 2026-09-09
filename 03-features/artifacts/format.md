status: current
version: 1.0
last_verified: 2026-09-09

# Artifact format — manifest v1

Source of truth: `mta-market-site/apps/server/src/lib/artifact/types.ts` and
`apps/server/src/lib/artifact/manifest.ts`. This document mirrors them (P-009).
Related: [signing.md](signing.md), [compatibility.md](compatibility.md),
[../drm/protocol-v2.md](../drm/protocol-v2.md).

## Manifest (formatVersion 1)

```jsonc
{
  "formatVersion": 1,
  "productId": "<resourceId, uuid>",
  "versionId": "<resourceVersionId, uuid>",
  "artifactId": "<resourceId>-<version string>",
  "sha256": "<64 hex — sha256 of the ARTIFACT file>",
  "publisherId": "<sellerId, uuid>",
  "publishedAt": "<ISO 8601>",
  "dependencies": [
    {
      "resourceName": "<slug>",
      "resourceId": "<uuid, optional>",
      "minVersion": "<dotted numeric, optional>",
      "maxVersion": "<optional>",
      "optional": false
    }
  ],
  "compatibility": {
    "mta": { "min": "1.5.0", "max": "1.6.0", "tested": ["1.5.9", "1.6.0"] },
    "os": ["linux", "windows"],
    "architecture": ["x64", "x86"],
    "requiredModules": ["string"],
    "conflicts": [{ "resourceName": "slug", "resourceId": "uuid?", "reason": "..." }]
  },
  "signature": {
    "algorithm": "EdDSA",
    "keyId": "<PublisherKey id>",
    "publicKey": "<base64 SPKI DER>",
    "signedAt": "<ISO 8601>"
  },
  "drm": {
    "enabled": true,
    "version": 2,
    "encryptionAlgorithm": "AES-256-GCM"
  }
}
```

Field notes (from `manifest.ts`):

- `sha256` is the artifact file hash. The manifest's own integrity is bound
  separately via `hashManifest(manifest)` inside the signing payload — do not
  conflate the two (explicit comment in `artifact/crypto.ts`).
- `signature.keyId` / `signature.publicKey` / `signature.signedAt` are filled
  after signing (`attachSignatureMetadata`).
- `drm` is present only when `drmEnabled` (server default `true`).
- Defaults applied by `buildCompatibility`: `os = ["linux","windows"]`,
  `architecture = ["x64"]`, `requiredModules = []`, `conflicts = []`,
  `mta.tested = []`.

## Structural validation

`validateManifest()` accepts a manifest iff:

- `formatVersion === 1`;
- required fields present: `productId`, `versionId`, `artifactId`, `sha256`,
  `publisherId`, `publishedAt`;
- `sha256` matches `^[a-f0-9]{64}$` (case-insensitive);
- `publishedAt` parses as a date;
- `dependencies` (if present) is an array;
- `compatibility` object present.

Serialization for storage/transport is `JSON.stringify(manifest, null, 2)`
(`serializeManifest`); the **signature input** is canonical JSON
(sorted keys, no whitespace, see [signing.md](signing.md)).

## Storage model

| Table | Content |
|---|---|
| `ArtifactSignature` | one row per ResourceVersion (`versionId` unique): `keyId`, `signature`, `algorithm`, `manifestHash`, `artifactHash`, `manifest` (JSON), `signedAt` |
| `ArtifactEncryption` | one row per encrypted version: `dekId`, `algorithm` (`aes-256-gcm`), `wrappedDek`, `wrapNonce`, `wrapTag` |
| `PublisherKey` | signing keys (`sellerId`, `keyType ED25519`, `status ACTIVE/REVOKED`) |

## Encryption envelope

Per version (G-005): unique 32-byte DEK; payload encrypted with AES-256-GCM
(fresh 12-byte nonce, `{nonce, tag, ciphertext}`); DEK wrapped under the
server master key (`DRM_MASTER_KEY`). The unwrapped DEK is released only
through `POST /drm/v2/versions/:versionId/dek` to a lease holder that proves
possession of its installation key. See
[../drm/protocol-v2.md](../drm/protocol-v2.md).

## Test evidence

- `mta-market-site/tests/artifact-manifest.test.ts` (18 tests) — generation,
  validation, round-trip.
- `mta-market-site/tests/artifact-crypto.test.ts` (20 tests) — hashing,
  canonical JSON, sign/verify, tamper rejection.
- `mta-market-site/tests/drm-g6.test.ts` — envelope wrap/release/decrypt,
  tampering fails.
