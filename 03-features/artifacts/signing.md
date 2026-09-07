# Artifact Signing System Documentation

**TASK-019: Artifact Signing Implementation**

---

## Overview

The Artifact Signing System provides cryptographic integrity and authenticity verification for MTA Market resources using Ed25519 digital signatures.

### Key Features

- **Ed25519 Cryptography**: Fast, secure asymmetric signing
- **Artifact Manifests**: Complete metadata with compatibility info
- **Signature Verification**: Automatic verification on download
- **Key Management**: Publisher key lifecycle management
- **CLI Tools**: Command-line interface for key operations

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Upload Pipeline                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Seller uploads artifact (ZIP file)                  │
│                   ↓                                      │
│  2. Generate manifest (metadata + compatibility)        │
│                   ↓                                      │
│  3. Calculate artifact SHA-256 hash                     │
│                   ↓                                      │
│  4. Sign manifest + hash with Ed25519                   │
│                   ↓                                      │
│  5. Store signature in database                         │
│                   ↓                                      │
│  6. Upload signed artifact to S3/R2                     │
│                                                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   Download Pipeline                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Buyer requests download                             │
│                   ↓                                      │
│  2. Check entitlement                                   │
│                   ↓                                      │
│  3. Retrieve signature from database                    │
│                   ↓                                      │
│  4. Verify signature with public key                    │
│                   ↓                                      │
│  5. Generate presigned S3 URL (if valid)                │
│                   ↓                                      │
│  6. Buyer downloads artifact                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Database Schema

### PublisherKey

Stores seller's public key for artifact signing.

```prisma
model PublisherKey {
  id          String    @id @default(cuid())
  sellerId    String
  keyType     String    @default("ED25519")
  publicKey   String    // Base64 encoded
  algorithm   String    @default("EdDSA")
  status      KeyStatus @default(ACTIVE)
  createdAt   TimestamptzString
  revokedAt   TimestamptzString?
  revokedBy   String?
  revocationReason String?
}
```

**Key Points:**
- One active key per seller
- Public key stored in database
- Private key NEVER stored (seller's responsibility)
- Revocable for security incidents

### ArtifactSignature

Stores cryptographic signature for each artifact version.

```prisma
model ArtifactSignature {
  id           String @id @default(cuid())
  versionId    String @unique
  keyId        String
  signature    String  // Base64 encoded Ed25519 signature
  algorithm    String  // "EdDSA"
  manifestHash String  // SHA-256 of manifest
  artifactHash String  // SHA-256 of artifact file
  manifest     Json    // Complete manifest
  signedAt     TimestamptzString
  verifiedAt   TimestamptzString?
}
```

**Key Points:**
- One signature per resource version
- Immutable after creation
- Contains complete manifest
- Verification timestamp updated on each check

---

## Artifact Manifest

Complete metadata describing the artifact.

### Structure

```typescript
interface ArtifactManifest {
  formatVersion: 1;
  productId: string;           // Resource ID
  versionId: string;           // Version ID
  artifactId: string;          // Unique artifact identifier
  sha256: string;              // Artifact file hash
  publisherId: string;         // Seller ID
  publishedAt: string;         // ISO 8601 timestamp
  dependencies: Dependency[];
  compatibility: Compatibility;
  signature: SignatureMetadata;
  drm?: DRMMetadata;
}
```

### Example

```json
{
  "formatVersion": 1,
  "productId": "clx3r2k8n0000qzrm5g4j9k2p",
  "versionId": "clx3s4m9p0001qzrm6h5k0l3q",
  "artifactId": "clx3r2k8n0000qzrm5g4j9k2p-1.0.0",
  "sha256": "a3f2c1b9e4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1",
  "publisherId": "clx3t6n0p0002qzrm7i6l1m4r",
  "publishedAt": "2026-09-07T12:00:00.000Z",
  "dependencies": [
    {
      "resourceName": "base-library",
      "minVersion": "1.0.0",
      "optional": false
    }
  ],
  "compatibility": {
    "mta": {
      "min": "1.5.0",
      "tested": ["1.5.9", "1.6.0"]
    },
    "os": ["linux", "windows"],
    "architecture": ["x64"],
    "requiredModules": ["mta-market-module"]
  },
  "signature": {
    "algorithm": "EdDSA",
    "keyId": "clx3u7o1q0003qzrm8j7m2n5s",
    "publicKey": "MCowBQYDK2VwAyEA...",
    "signedAt": "2026-09-07T12:00:00.000Z"
  },
  "drm": {
    "enabled": true,
    "version": 2,
    "encryptionAlgorithm": "AES-256-GCM"
  }
}
```

---

## API Reference

### High-Level Service API

#### `createPublisherKey(sellerId: string)`

Generate Ed25519 keypair for seller.

**Parameters:**
- `sellerId` - User ID of the seller

**Returns:**
```typescript
{
  keyId: string;
  publicKey: string;    // Store in database
  privateKey: string;   // MUST store securely (ENV/KMS/Vault)
}
```

**Important:** Private key is returned once and NEVER stored in database.

**Example:**
```typescript
const { keyId, publicKey, privateKey } = await createPublisherKey('seller-123');

// Store private key securely
process.env[`SELLER_KEY_${sellerId}`] = privateKey;
// Or use KMS, Vault, etc.
```

#### `signAndStoreArtifact(versionId, artifactBuffer, privateKey)`

Sign artifact and store signature in database.

**Parameters:**
- `versionId` - ResourceVersion ID
- `artifactBuffer` - Raw artifact file (Buffer)
- `privateKey` - Seller's private key (Base64)

**Returns:**
```typescript
{
  manifest: ArtifactManifest;
  signature: string;
  manifestHash: string;
  artifactHash: string;
}
```

**Example:**
```typescript
const artifactBuffer = await readFile('gamemode.zip');
const privateKey = process.env.SELLER_KEY_seller123;

const result = await signAndStoreArtifact(
  'version-456',
  artifactBuffer,
  privateKey
);

console.log('Signed:', result.artifactHash);
```

#### `verifyStoredArtifact(versionId, artifactBuffer)`

Verify artifact signature from database.

**Parameters:**
- `versionId` - ResourceVersion ID
- `artifactBuffer` - Artifact file to verify (Buffer)

**Returns:**
```typescript
{
  valid: boolean;
  errors: string[];
  warnings: string[];
  signature?: {
    algorithm: string;
    keyId: string;
    verified: boolean;
  };
}
```

**Example:**
```typescript
const artifactBuffer = await downloadFromS3(key);

const result = await verifyStoredArtifact('version-456', artifactBuffer);

if (!result.valid) {
  throw new Error(`Invalid signature: ${result.errors.join(', ')}`);
}
```

---

## CLI Usage

### Generate Keypair

```bash
cd apps/server

# Generate new keypair for seller
pnpm artifact:keygen --seller-id clx3t6n0p0002qzrm7i6l1m4r

# Output:
# 🔑 Generating Ed25519 keypair...
# ✅ Keypair generated successfully!
# 📋 Key ID: clx3u7o1q0003qzrm8j7m2n5s
# 🔓 Public Key: MCowBQYDK2VwAyEA...
# 🔐 Private key saved to: .keys/seller-123.key
#
# ⚠️  IMPORTANT: Store private key securely!
```

### Sign Artifact

```bash
# Sign artifact with private key
pnpm artifact:sign \
  --file ./uploads/gamemode.zip \
  --version-id clx3s4m9p0001qzrm6h5k0l3q \
  --private-key .keys/seller-123.key

# Or with raw key
pnpm artifact:sign \
  --file ./uploads/gamemode.zip \
  --version-id clx3s4m9p0001qzrm6h5k0l3q \
  --private-key "MC4CAQAwBQYDK2VwBCIEIE..."
```

### Verify Artifact

```bash
# Verify artifact signature
pnpm artifact:verify \
  --file ./uploads/gamemode.zip \
  --version-id clx3s4m9p0001qzrm6h5k0l3q

# Output:
# 🔍 Verifying artifact...
# ✅ Signature valid!
# ✓ Algorithm: EdDSA
# ✓ Key ID: clx3u7o1q0003qzrm8j7m2n5s
# ✓ Manifest format: v1
```

### List Keys

```bash
# List all keys for seller
pnpm artifact:keys --seller-id clx3t6n0p0002qzrm7i6l1m4r
```

### Revoke Key

```bash
# Revoke compromised key
pnpm artifact:revoke \
  --key-id clx3u7o1q0003qzrm8j7m2n5s \
  --revoked-by admin-user-id \
  --reason "Private key compromised"
```

---

## Integration with Upload Pipeline

### Before (Without Signing)

```typescript
async function uploadResource(file: Buffer, versionId: string) {
  // 1. Upload to S3
  const url = await uploadToS3(file);
  
  // 2. Update version record
  await prisma.resourceVersion.update({
    where: { id: versionId },
    data: { fileUrl: url }
  });
}
```

### After (With Signing)

```typescript
async function uploadResource(file: Buffer, versionId: string, sellerId: string) {
  // 1. Get seller's private key
  const privateKey = await getSellerPrivateKey(sellerId);
  
  // 2. Sign artifact and store signature
  const signed = await signAndStoreArtifact(versionId, file, privateKey);
  
  // 3. Upload to S3
  const url = await uploadToS3(file);
  
  // 4. Update version record
  await prisma.resourceVersion.update({
    where: { id: versionId },
    data: { 
      fileUrl: url,
      fileChecksum: signed.artifactHash
    }
  });
  
  return signed;
}
```

---

## Integration with Download Pipeline

### Download Verification

```typescript
async function downloadResource(versionId: string, userId: string) {
  // 1. Check entitlement
  const hasAccess = await checkEntitlement(userId, versionId);
  if (!hasAccess) {
    throw new Error('No entitlement');
  }
  
  // 2. Get artifact from S3
  const artifactBuffer = await downloadFromS3(versionId);
  
  // 3. Verify signature
  const verification = await verifyStoredArtifact(versionId, artifactBuffer);
  
  if (!verification.valid) {
    throw new Error(`Invalid signature: ${verification.errors.join(', ')}`);
  }
  
  // 4. Return verified artifact
  return artifactBuffer;
}
```

---

## Security Considerations

### Private Key Storage

**❌ NEVER:**
- Store private keys in database
- Commit private keys to git
- Expose private keys to clients
- Log private keys

**✅ RECOMMENDED:**

**Option 1: Environment Variables (Development)**
```bash
# .env (never commit)
SELLER_KEY_seller123="MC4CAQAwBQYDK2VwBCIEIE..."
```

**Option 2: AWS KMS (Production)**
```typescript
import { KMSClient, DecryptCommand } from '@aws-sdk/client-kms';

async function getPrivateKey(sellerId: string) {
  const kms = new KMSClient({});
  const result = await kms.send(new DecryptCommand({
    CiphertextBlob: encryptedKey
  }));
  return result.Plaintext;
}
```

**Option 3: HashiCorp Vault (Enterprise)**
```typescript
import vault from 'node-vault';

async function getPrivateKey(sellerId: string) {
  const client = vault({ endpoint: 'https://vault.internal' });
  const secret = await client.read(`secret/sellers/${sellerId}/key`);
  return secret.data.privateKey;
}
```

### Key Rotation

When rotating keys:

1. Generate new keypair
2. Sign new artifacts with new key
3. Keep old key active for existing artifacts
4. Gradually phase out old key
5. Revoke old key after migration

### Signature Verification

Always verify signatures:
- On download
- Before DRM activation
- During updates
- After rollback

---

## Testing

### Run Tests

```bash
cd apps/server

# Run all artifact signing tests
pnpm test tests/artifact-*.test.ts

# Run specific test suite
pnpm test tests/artifact-crypto.test.ts
pnpm test tests/artifact-manifest.test.ts
```

### Test Coverage

- ✅ Keypair generation
- ✅ Signing
- ✅ Verification
- ✅ Manifest generation
- ✅ Manifest validation
- ✅ Canonical JSON serialization
- ✅ Hash computation
- ✅ Key format validation
- ✅ Signature format validation
- ✅ Tamper detection
- ✅ Hash mismatch detection

---

## Troubleshooting

### "Seller already has an active publisher key"

Each seller can only have one active key at a time. Revoke the old key first:

```bash
pnpm artifact:revoke \
  --key-id <old-key-id> \
  --revoked-by <admin-id> \
  --reason "Key rotation"
```

### "Invalid signature"

Possible causes:
- Artifact file modified after signing
- Wrong private key used
- Corrupted signature in database
- Public key revoked

Solution: Re-sign the artifact with correct private key.

### "No signature found for this artifact"

Artifact was uploaded before signing system was implemented. Sign it:

```bash
pnpm artifact:sign \
  --file <path-to-artifact> \
  --version-id <version-id> \
  --private-key <key>
```

---

## Performance

### Signing Performance

- **Keypair generation:** ~1ms
- **Signing:** ~2-5ms
- **Verification:** ~3-7ms
- **Hash calculation:** Depends on file size (~100MB/s)

### Database Impact

- **Storage:** ~2KB per signature
- **Queries:** 1 SELECT per download verification
- **Indexes:** Optimized for versionId and keyId lookups

---

## Future Enhancements

### Planned Features

1. **Batch Signing**: Sign multiple artifacts in one operation
2. **Key Expiration**: Automatic key rotation based on time
3. **Hardware Security Module (HSM)**: Support for HSM key storage
4. **Multi-Signature**: Require multiple signatures (platform + seller)
5. **Signature Timestamp**: RFC 3161 trusted timestamps
6. **Revocation List**: Public key revocation list (CRL)

---

## References

- [Ed25519 Specification](https://ed25519.cr.yp.to/)
- [Node.js Crypto Documentation](https://nodejs.org/api/crypto.html)
- [PROMNT.md Section 11: Artifact Architecture](../../../PROMNT.md#11-artifact-architecture)
- [TASK-019 Report](../08-reports/task-019-artifact-signing.md)

---

**Implementation Status:** ✅ COMPLETE  
**Last Updated:** 2026-09-07  
**Next Task:** TASK-020 (DRM Protocol v2)
