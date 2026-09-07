# DRM Protocol v2 Specification

**Version:** 2.0  
**Status:** Implementation Complete  
**Last Updated:** 2026-09-07

---

## Overview

DRM Protocol v2 replaces symmetric key distribution with asymmetric cryptography using Ed25519 digital signatures. Each installation generates its own keypair, and the server issues signed time-limited leases that bind licenses to specific installations and artifacts.

---

## Key Differences from v1

| Feature | v1 (Symmetric) | v2 (Asymmetric) |
|---------|----------------|-----------------|
| **Identity** | Hardware fingerprint | Ed25519 keypair |
| **Key Distribution** | Server sends symmetric key | Client generates keypair |
| **Authentication** | None | Challenge/response |
| **License Format** | Encrypted artifact | Signed lease |
| **Revocation** | Difficult | Revoke installation |
| **Replay Protection** | None | Nonce-based |
| **Artifact Binding** | None | SHA-256 hash in lease |

---

## Protocol Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Installation Registration                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Client generates Ed25519 keypair (client-side)                 │
│         ↓                                                        │
│  Client → POST /drm/v2/installations                            │
│         {                                                        │
│           "publicKey": "MCowBQYDK2VwAyEA...",                   │
│           "mtaVersion": "1.5.9",                                │
│           "moduleVersion": "0.5.0"                              │
│         }                                                        │
│         ↓                                                        │
│  Server ← 201 Created                                           │
│         {                                                        │
│           "installationId": "clx...",                           │
│           "challenge": "base64_encoded_random_32_bytes"         │
│         }                                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 2. Challenge Verification                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Client signs challenge with privateKey (Ed25519 signature)     │
│         ↓                                                        │
│  Client → POST /drm/v2/installations/:id/verify                 │
│         {                                                        │
│           "challengeResponse": "base64_encoded_signature"       │
│         }                                                        │
│         ↓                                                        │
│  Server verifies signature with stored publicKey                │
│         ↓                                                        │
│  Server ← 200 OK                                                │
│         {                                                        │
│           "verified": true,                                     │
│           "installationId": "clx..."                            │
│         }                                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 3. License Activation                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Client generates random nonce                                  │
│         ↓                                                        │
│  Client → POST /drm/v2/activate                                 │
│         {                                                        │
│           "licenseId": "clx...",                                │
│           "installationId": "clx...",                           │
│           "nonce": "64_hex_chars"                               │
│         }                                                        │
│         ↓                                                        │
│  Server verifies ownership                                      │
│  Server generates lease                                         │
│  Server signs lease with server's privateKey                    │
│         ↓                                                        │
│  Server ← 200 OK (SignedLease)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 4. Lease Verification (Client-Side)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Client fetches server's publicKey (once, cached)               │
│  Client verifies lease signature                                │
│  Client checks:                                                 │
│    ✓ Signature valid                                            │
│    ✓ Not expired                                                │
│    ✓ Nonce matches                                              │
│    ✓ Artifact hash matches                                      │
│    ✓ Resource/version matches                                   │
│         ↓                                                        │
│  If valid: Run resource                                         │
│  If invalid: Deny execution                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 5. Heartbeat (Optional)                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Client → POST /drm/v2/heartbeat (periodic)                     │
│         {                                                        │
│           "installationId": "clx...",                           │
│           "resourceId": "clx...",                               │
│           "uptime": 3600                                        │
│         }                                                        │
│         ↓                                                        │
│  Server ← 200 OK                                                │
│         {                                                        │
│           "acknowledged": true,                                 │
│           "leaseValid": true,                                   │
│           "shouldUpdate": false                                 │
│         }                                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## API Endpoints

### GET /drm/v2/public-key

Get server's public Ed25519 key for lease verification.

**Response:**
```json
{
  "publicKey": "MCowBQYDK2VwAyEA...",
  "algorithm": "EdDSA",
  "keyType": "ED25519"
}
```

**Caching:** Client should cache this key.

---

### POST /drm/v2/installations

Register new installation with public key.

**Request:**
```json
{
  "publicKey": "MCowBQYDK2VwAyEA...",
  "mtaVersion": "1.5.9",
  "moduleVersion": "0.5.0",
  "serverSerial": "optional",
  "serverName": "optional"
}
```

**Response (201):**
```json
{
  "installationId": "clx3u7o1q0003qzrm8j7m2n5s",
  "challenge": "base64_encoded_32_bytes"
}
```

**Errors:**
- `400 INVALID_REQUEST` - Missing required fields
- `409 INSTALLATION_EXISTS` - Public key already registered
- `500 SERVER_ERROR` - Internal error

---

### POST /drm/v2/installations/:id/verify

Verify installation with signed challenge response.

**Request:**
```json
{
  "challengeResponse": "base64_encoded_signature"
}
```

**Response (200):**
```json
{
  "verified": true,
  "installationId": "clx3u7o1q0003qzrm8j7m2n5s"
}
```

**Errors:**
- `400 INVALID_REQUEST` - Missing challengeResponse
- `401 DRM_INVALID_CHALLENGE_RESPONSE` - Invalid signature
- `404 DRM_INSTALLATION_NOT_FOUND` - Installation not found
- `500 SERVER_ERROR` - Internal error

---

### POST /drm/v2/activate

Activate license and receive signed lease.

**Request:**
```json
{
  "licenseId": "clx3s4m9p0001qzrm6h5k0l3q",
  "installationId": "clx3u7o1q0003qzrm8j7m2n5s",
  "nonce": "64_hex_characters_random"
}
```

**Response (200):**
```json
{
  "protocolVersion": 2,
  "licenseId": "clx3s4m9p0001qzrm6h5k0l3q",
  "installationId": "clx3u7o1q0003qzrm8j7m2n5s",
  "resourceId": "clx3r2k8n0000qzrm5g4j9k2p",
  "resourceVersionId": "clx3s4m9p0001qzrm6h5k0l3q",
  "artifactHash": "a3f2c1b9e4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1",
  "issuedAt": "2026-09-07T12:00:00.000Z",
  "expiresAt": "2026-09-14T12:00:00.000Z",
  "nonce": "64_hex_characters_random",
  "serverKeyId": "clx3t6n0p0002qzrm7i6l1m4r",
  "capabilities": ["run", "update"],
  "signature": "base64_encoded_Ed25519_signature"
}
```

**Errors:**
- `400 INVALID_REQUEST` - Missing fields or invalid nonce format
- `403 DRM_INSTALLATION_NOT_VERIFIED` - Installation not verified
- `404 DRM_INSTALLATION_NOT_FOUND` - Installation not found
- `404 DRM_INVALID_LICENSE` - License not found
- `409 DRM_NONCE_ALREADY_USED` - Nonce already used (replay)
- `500 SERVER_ERROR` / `500 SERVER_MISCONFIGURED` - Internal error

---

### POST /drm/v2/heartbeat

Record installation heartbeat.

**Request:**
```json
{
  "installationId": "clx3u7o1q0003qzrm8j7m2n5s",
  "resourceId": "clx3r2k8n0000qzrm5g4j9k2p",
  "uptime": 3600,
  "lastError": "optional_error_message"
}
```

**Response (200):**
```json
{
  "acknowledged": true,
  "leaseValid": true,
  "shouldUpdate": false,
  "updateVersionId": null
}
```

---

### GET /drm/v2/leases/:installationId/:resourceId

Get active lease for installation and resource.

**Response (200):**
```json
{
  "protocolVersion": 2,
  "licenseId": "...",
  "installationId": "...",
  ...
}
```

**Errors:**
- `404 LEASE_NOT_FOUND` - No active lease found
- `500 SERVER_ERROR` - Internal error

---

## Signed Lease Format

```typescript
interface SignedLease {
  protocolVersion: 2;
  licenseId: string;
  installationId: string;
  resourceId: string;
  resourceVersionId: string;
  artifactHash: string;        // SHA-256 of artifact file
  issuedAt: string;            // ISO 8601 timestamp
  expiresAt: string;           // ISO 8601 timestamp
  nonce: string;               // 64 hex characters
  serverKeyId: string;
  capabilities: Capability[];  // ["run", "update", "debug", "export"]
  signature: string;           // Base64 Ed25519 signature
}
```

**Signature Covers:**
- Canonical JSON of lease (excluding signature field)
- SHA-256 hash of canonical JSON

**Verification Steps:**
1. Parse lease JSON
2. Extract signature
3. Create canonical JSON (sorted keys, no whitespace, no signature field)
4. Hash canonical JSON with SHA-256
5. Verify Ed25519 signature with server's public key
6. Check `expiresAt > now()`
7. Check `artifactHash` matches downloaded artifact
8. Check `resourceId` and `resourceVersionId` match expected

---

## Nonce Requirements

**Format:** 64 hexadecimal characters (32 bytes)

**Generation:** `crypto.randomBytes(32).toString('hex')`

**Usage:** Each nonce can only be used once. Server maintains nonce table to prevent replay attacks.

**Lifetime:** Nonces expire after successful lease generation or 1 hour (whichever comes first).

---

## Security Considerations

### Private Key Security

**Installation Private Key:**
- Generated client-side
- Never transmitted to server
- Stored securely in MTA module memory
- Used only for signing challenges

**Server Private Key:**
- Generated once during setup
- Stored in ENV, KMS, or Vault
- Never exposed to clients
- Used only for signing leases

### Replay Protection

1. **Nonce:** Each activation requires unique nonce
2. **Nonce Table:** Server tracks used nonces
3. **Expiry:** Old nonces are periodically cleaned

### Revocation

**Installation Revocation:**
```sql
UPDATE installations 
SET status = 'REVOKED', 
    revokedAt = now(), 
    revokedBy = 'admin-id',
    revocationReason = 'Security incident'
WHERE id = 'installation-id';
```

**Effect:** Future lease requests will be denied.

**Existing Leases:** Continue working until expiry (grace period).

### Grace Period

When server is temporarily unavailable:
1. Client continues with valid lease until expiry
2. Client logs warning
3. Client retries with exponential backoff
4. If lease expires during outage, controlled degradation

---

## Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `DRM_INVALID_LICENSE` | 404 | License not found or invalid |
| `DRM_INSTALLATION_NOT_FOUND` | 404 | Installation not found |
| `DRM_INSTALLATION_NOT_VERIFIED` | 403 | Challenge not verified |
| `DRM_INSTALLATION_REVOKED` | 403 | Installation revoked |
| `DRM_INVALID_CHALLENGE_RESPONSE` | 401 | Challenge signature invalid |
| `DRM_NONCE_ALREADY_USED` | 409 | Nonce already used (replay) |
| `DRM_NONCE_EXPIRED` | 400 | Nonce expired |
| `DRM_LEASE_EXPIRED` | 403 | Lease expired |
| `DRM_INVALID_SIGNATURE` | 401 | Lease signature invalid |
| `DRM_PROTOCOL_VERSION_MISMATCH` | 400 | Unsupported protocol version |
| `DRM_ARTIFACT_HASH_MISMATCH` | 400 | Artifact hash mismatch |
| `DRM_INSUFFICIENT_CAPABILITIES` | 403 | Capability not granted |
| `DRM_SERVER_KEY_REVOKED` | 500 | Server key revoked |

---

## Compatibility Matrix

| Site Version | API | DRM Protocol | Module Version | Artifact Format |
|--------------|-----|--------------|----------------|-----------------|
| 0.1.x        | v1  | v1           | 0.1.x - 0.3.x  | v1              |
| 0.2.x        | v1  | v2           | >= 0.4.x       | v1              |
| 1.0.x        | v2  | v2           | >= 0.5.x       | v2              |

---

## Migration from v1 to v2

### Server-Side

1. Deploy v2 endpoints alongside v1
2. Keep v1 endpoints for backward compatibility
3. Generate server signing keypair
4. Configure `DRM_SERVER_PRIVATE_KEY` environment variable

### Client-Side

1. Update module to v0.5.0+
2. Generate installation keypair on first run
3. Register installation with server
4. Verify challenge
5. Request lease instead of symmetric key
6. Verify lease signature
7. Cache server public key

### Gradual Rollout

1. Deploy v2 server (supports both v1 and v2)
2. Release module update (optional for users)
3. Monitor v2 adoption
4. After 90% adoption, deprecate v1
5. After 180 days, remove v1 endpoints

---

## Testing

### Unit Tests

```bash
pnpm test tests/drm-crypto.test.ts
```

### Integration Tests

```bash
# Generate server key
pnpm drm:keygen

# Generate test installation
pnpm drm:test-installation

# Start server
pnpm dev

# Test registration
curl -X POST http://localhost:3001/drm/v2/installations \
  -H "Content-Type: application/json" \
  -d '{"publicKey":"...","mtaVersion":"1.5.9","moduleVersion":"0.5.0"}'

# Sign challenge (client-side)
# ...

# Verify challenge
curl -X POST http://localhost:3001/drm/v2/installations/:id/verify \
  -H "Content-Type: application/json" \
  -d '{"challengeResponse":"..."}'

# Activate license
curl -X POST http://localhost:3001/drm/v2/activate \
  -H "Content-Type: application/json" \
  -d '{"licenseId":"...","installationId":"...","nonce":"..."}'
```

---

## Performance

| Operation | Time |
|-----------|------|
| Keypair generation | ~1ms |
| Challenge generation | <1ms |
| Challenge signing | 2-5ms |
| Challenge verification | 3-7ms |
| Lease signing | 2-5ms |
| Lease verification | 3-7ms |
| Nonce generation | <1ms |

**Network Overhead:** +2 roundtrips vs v1 (registration + verification)

**Storage:** ~1KB per installation, ~2KB per lease

---

## References

- [Ed25519 Specification](https://ed25519.cr.yp.to/)
- [PROMNT.md Section 14: DRM v2](../../../PROMNT.md#14-drm-v2)
- [TASK-020 Report](../../08-reports/task-020-drm-v2.md)

---

**Status:** ✅ IMPLEMENTATION COMPLETE  
**Last Updated:** 2026-09-07  
**Next:** Module integration (C++)
