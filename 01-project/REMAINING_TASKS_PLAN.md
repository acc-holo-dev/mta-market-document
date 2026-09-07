# 🎯 MTA Market — Детальный план оставшихся задач

**Дата:** 2026-09-07  
**Базовый документ:** [PROMNT.md](../../../PROMNT.md)  
**Статус проекта:** [status.md](status.md)

---

## 📊 Статус выполнения задач из PROMNT.md

### ✅ Выполнено (TASK-001 — TASK-017)

| Задача | Статус | Отчет | Описание |
|---|:---:|---|---|
| **TASK-001** | ❌ PARTIAL | - | Синхронизация имен репозиториев (частично) |
| **TASK-002** | ❌ PARTIAL | - | Canonical status (status.md обновлен, но есть устаревшие claims) |
| **TASK-003** | ✅ DONE | [task-003-completed.md](../08-reports/task-003-completed.md) | ID migration: Int → String CUID |
| **TASK-004** | ✅ DONE | [task-004-completed.md](../08-reports/task-004-completed.md) | Payment bypass removal |
| **TASK-005** | ✅ DONE | [task-005-completed.md](../08-reports/task-005-completed.md) | YooKassa webhook security |
| **TASK-006** | ✅ DONE | [task-006-completed.md](../08-reports/task-006-completed.md) | Download protection (presigned URLs) |
| **TASK-007** | ✅ DONE | [task-007-completed.md](../08-reports/task-007-completed.md) | DRM activation ownership |
| **TASK-008** | ✅ DONE | [task-008-completed.md](../08-reports/task-008-completed.md) | Seller moderation bypass blocked |
| **TASK-009** | ✅ DONE | [task-009-completed.md](../08-reports/task-009-completed.md) | Auth token security (httpOnly cookies) |
| **TASK-010** | ✅ DONE | [task-010-completed.md](../08-reports/task-010-completed.md) | Startup secret checks |
| **TASK-011** | ✅ DONE | [task-011-completed.md](../08-reports/task-011-completed.md) | Identity provider model |
| **TASK-012** | ✅ DONE | [task-012-completed.md](../08-reports/task-012-completed.md) | PaymentProvider interface |
| **TASK-013** | ✅ DONE | [task-013-completed.md](../08-reports/task-013-completed.md) | Free resources (pricing_type = FREE) |
| **TASK-014** | ✅ DONE | [task-014-completed.md](../08-reports/task-014-completed.md) | Discount campaigns |
| **TASK-015** | ✅ DONE | [task-015-completed.md](../08-reports/task-015-completed.md) | Order/OrderItem model |
| **TASK-016** | ✅ DONE | [task-016-completed.md](../08-reports/task-016-completed.md) | Service product type (schema) |
| **TASK-017** | ✅ DONE | [task-016-017-completed.md](../08-reports/task-016-017-completed.md) | Financial ledger review |

**Итого выполнено:** 15 из 25 задач PROMNT.md (60%)

---

## 🚧 Оставшиеся задачи (TASK-018 — TASK-025)

### ⚠️ Важное замечание

**Задачи из PROMNT.md выполнены только на уровне SCHEMA:**
- ✅ Prisma models созданы
- ❌ Routes/endpoints НЕ реализованы
- ❌ Business logic НЕ реализована
- ❌ Integration tests НЕ написаны

**Следующий этап: IMPLEMENTATION**

---

## 📋 TASK-018: Reconciliation Worker

**Приоритет:** 🟡 MEDIUM  
**Зависимости:** TASK-017 (ledger review)  
**Оценка:** 8-10 часов  
**Статус:** ❌ NOT STARTED

### Описание

Создать scheduled job для сверки платежей между internal ledger и payment providers.

### Требования из PROMNT.md

```text
# 23. Reconciliation

Add scheduled reconciliation job.

Compare:
- provider payments vs internal payments
- provider refunds vs internal refunds
- provider payouts vs internal payouts

Mismatches create alert and reconciliation record.
Never silently auto-correct money.
```

### Acceptance Criteria

- [ ] Создать модель `ReconciliationReport`
- [ ] Scheduled job (cron/worker)
- [ ] Сравнение YooKassa payments с internal
- [ ] Обнаружение расхождений
- [ ] Alert system для mismatches
- [ ] Reconciliation report UI (admin)
- [ ] Документация reconciliation процесса

### Детальный план

#### 1. Database Schema (1 час)

```prisma
model ReconciliationReport {
  id                String   @id @default(cuid())
  provider          String   // YOOKASSA, TBANK, etc.
  reportType        String   // PAYMENT, REFUND, PAYOUT
  periodStart       TimestamptzString
  periodEnd         TimestamptzString
  internalCount     Int
  internalTotal     Int
  providerCount     Int
  providerTotal     Int
  mismatchCount     Int
  status            ReconciliationStatus
  createdAt         TimestamptzString @default(now())
  completedAt       TimestamptzString?

  mismatches ReconciliationMismatch[]
}

model ReconciliationMismatch {
  id               String @id @default(cuid())
  reportId         String
  type             String // MISSING_INTERNAL, MISSING_PROVIDER, AMOUNT_MISMATCH
  internalId       String?
  providerId       String?
  expectedAmount   Int?
  actualAmount     Int?
  description      String
  resolved         Boolean @default(false)
  resolvedAt       TimestamptzString?
  resolvedBy       String?
  resolution       String?
  createdAt        TimestamptzString @default(now())

  report ReconciliationReport @relation(...)
}

enum ReconciliationStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
  FAILED
}
```

#### 2. Reconciliation Service (4 часа)

**File:** `apps/server/src/lib/reconciliation.ts`

```typescript
interface ReconciliationInput {
  provider: PaymentProvider;
  periodStart: Date;
  periodEnd: Date;
}

interface ReconciliationResult {
  reportId: string;
  internalCount: number;
  internalTotal: number;
  providerCount: number;
  providerTotal: number;
  mismatches: Mismatch[];
  status: 'ok' | 'mismatches_found';
}

// Основные функции
export async function reconcilePayments(input: ReconciliationInput): Promise<ReconciliationResult>;
export async function reconcileRefunds(input: ReconciliationInput): Promise<ReconciliationResult>;
export async function fetchProviderTransactions(provider, period): Promise<ProviderTransaction[]>;
export async function compareTransactions(internal, provider): Promise<Mismatch[]>;
export async function createReconciliationReport(result): Promise<void>;
export async function alertOnMismatch(mismatch): Promise<void>;
```

#### 3. Scheduled Job (2 часа)

**File:** `apps/server/src/jobs/reconciliation.ts`

```typescript
// Cron: каждый день в 03:00
// Compare yesterday's transactions

export async function runDailyReconciliation() {
  const yesterday = {
    start: startOfDay(subDays(new Date(), 1)),
    end: endOfDay(subDays(new Date(), 1))
  };

  await reconcilePayments({
    provider: 'YOOKASSA',
    periodStart: yesterday.start,
    periodEnd: yesterday.end
  });
}
```

#### 4. Admin API (1 час)

```typescript
// GET /admin/reconciliation/reports
// GET /admin/reconciliation/reports/:id
// POST /admin/reconciliation/run (manual trigger)
// PATCH /admin/reconciliation/mismatches/:id/resolve
```

#### 5. Tests (2 часа)

```typescript
describe('Reconciliation', () => {
  it('should detect missing internal payment');
  it('should detect missing provider payment');
  it('should detect amount mismatch');
  it('should create alert on mismatch');
  it('should not auto-correct money');
  it('should handle provider API unavailability');
});
```

### Блокеры

- ❌ YooKassa API client для fetching transactions (нужен wrapper)
- ❌ Alert system (email/webhook)
- ❌ Cron scheduler (node-cron или bull)

---

## 📋 TASK-019: Artifact Signing + Manifest

**Приоритет:** 🔴 HIGH (P0 Security)  
**Зависимости:** None  
**Оценка:** 12-16 часов  
**Статус:** ❌ NOT STARTED

### Описание

Реализовать подписание artifacts с помощью asymmetric cryptography и создание artifact manifest.

### Требования из PROMNT.md

```text
# 11. Artifact architecture

Artifact manifest:
{
  "formatVersion": 1,
  "productId": "...",
  "versionId": "...",
  "artifactId": "...",
  "sha256": "...",
  "publisherId": "...",
  "dependencies": [],
  "compatibility": {},
  "signature": {},
  "drm": {}
}

Artifact signature must cover canonical manifest + artifact hash.
```

### Acceptance Criteria

- [ ] Generate publisher keypair (Ed25519 or RSA)
- [ ] Sign artifact + manifest
- [ ] Verify signature on download
- [ ] Store public keys in DB
- [ ] Manifest schema validation
- [ ] Signature verification service
- [ ] CLI tool для key generation
- [ ] Documentation

### Детальный план

#### 1. Key Management Schema (2 часа)

```prisma
model PublisherKey {
  id          String @id @default(cuid())
  sellerId    String
  keyType     String // ED25519, RSA
  publicKey   String
  algorithm   String
  createdAt   TimestamptzString @default(now())
  revokedAt   TimestamptzString?
  status      KeyStatus @default(ACTIVE)

  seller User @relation(...)
  artifacts ArtifactSignature[]
}

model ArtifactSignature {
  id           String @id @default(cuid())
  versionId    String @unique
  keyId        String
  signature    String  // Base64 encoded
  algorithm    String
  manifestHash String  // SHA-256 of canonical manifest JSON
  artifactHash String  // SHA-256 of artifact file
  signedAt     TimestamptzString @default(now())

  version ResourceVersion @relation(...)
  key     PublisherKey    @relation(...)
}

enum KeyStatus {
  ACTIVE
  REVOKED
  EXPIRED
}
```

#### 2. Manifest Generator (3 часа)

**File:** `apps/server/src/lib/artifact/manifest.ts`

```typescript
interface ArtifactManifest {
  formatVersion: 1;
  productId: string;
  versionId: string;
  artifactId: string;
  sha256: string;
  publisherId: string;
  publishedAt: string;
  dependencies: Dependency[];
  compatibility: Compatibility;
  signature: Signature;
  drm: DRMMetadata;
}

export async function generateManifest(version: ResourceVersion): Promise<ArtifactManifest>;
export function canonicalJSON(manifest: ArtifactManifest): string;
export function hashManifest(manifest: ArtifactManifest): string;
```

#### 3. Signing Service (4 часа)

**File:** `apps/server/src/lib/artifact/signing.ts`

```typescript
// Use Node.js crypto or @noble/ed25519

export async function generatePublisherKeypair(): Promise<{
  publicKey: string;
  privateKey: string;
}>;

export async function signArtifact(input: {
  manifest: ArtifactManifest;
  artifactHash: string;
  privateKey: string;
}): Promise<string>;

export async function verifyArtifactSignature(input: {
  manifest: ArtifactManifest;
  signature: string;
  publicKey: string;
}): Promise<boolean>;
```

#### 4. Upload Pipeline Integration (3 часа)

```typescript
// apps/server/src/lib/upload.ts

async function processUpload(file, seller) {
  // 1. Upload to storage
  const url = await uploadToS3(file);
  
  // 2. Calculate hash
  const artifactHash = await sha256(file);
  
  // 3. Generate manifest
  const manifest = await generateManifest({ artifactHash, seller, ... });
  
  // 4. Sign manifest + artifact
  const privateKey = await getSellerPrivateKey(seller);
  const signature = await signArtifact({ manifest, artifactHash, privateKey });
  
  // 5. Store signature
  await prisma.artifactSignature.create({ ... });
  
  return { url, manifest, signature };
}
```

#### 5. Verification Middleware (2 часа)

```typescript
// Verify signature before download
export async function verifyDownload(versionId: string): Promise<boolean> {
  const signature = await prisma.artifactSignature.findUnique({ versionId });
  const publicKey = await prisma.publisherKey.findUnique({ id: signature.keyId });
  
  return verifyArtifactSignature({
    manifest: signature.manifest,
    signature: signature.signature,
    publicKey: publicKey.publicKey
  });
}
```

#### 6. CLI Tool (2 часа)

```bash
# Generate keypair
pnpm artifact:keygen --seller-id <id>

# Sign artifact manually
pnpm artifact:sign --file <path> --key <private-key>

# Verify signature
pnpm artifact:verify --file <path> --signature <sig> --public-key <key>
```

### Блокеры

- ❌ Decision: Ed25519 vs RSA (рекомендую Ed25519)
- ❌ Private key storage strategy (ENV? AWS KMS? HashiCorp Vault?)
- ❌ Key rotation policy

---

## 📋 TASK-020: DRM Protocol v2

**Приоритет:** 🔴 HIGH (P0 Security)  
**Зависимости:** TASK-019 (artifact signing)  
**Оценка:** 16-20 часов  
**Статус:** ❌ NOT STARTED

### Описание

Заменить DRM v1 (symmetric) на v2 (asymmetric) с installation keypairs и signed leases.

### Требования из PROMNT.md

```text
# 14. DRM v2

Production DRM:
installation keypair
    |
    | signed challenge
    v
license API
    |
    | signed lease
    v
MTA Market Module

Server never receives installation private key.

Lease must bind:
- protocolVersion
- licenseId
- installationId
- resourceId
- resourceVersionId
- artifactHash
- issuedAt
- expiresAt
- nonce
- keyId
- capabilities
```

### Acceptance Criteria

- [ ] Installation keypair generation (module side)
- [ ] Signed challenge/response
- [ ] Signed lease format
- [ ] Lease verification (module side)
- [ ] Nonce/replay protection
- [ ] Server signing key management
- [ ] Protocol versioning
- [ ] Backward compatibility with v1 (optional)
- [ ] Module integration tests
- [ ] Protocol documentation

### Детальный план

#### 1. Protocol Specification (2 часа)

**File:** `mta-market-document/03-features/drm/protocol-v2.md`

```text
DRM Protocol v2 Specification

1. Installation Registration
   Module → Server: POST /drm/v2/installations
   {
     "publicKey": "base64...",
     "mtaVersion": "1.5.9",
     "moduleVersion": "0.5.0"
   }
   
   Server → Module: 
   {
     "installationId": "cuid...",
     "challenge": "base64..."
   }

2. Challenge Response
   Module signs challenge with private key
   
   Module → Server: POST /drm/v2/installations/:id/verify
   {
     "challengeResponse": "base64..."
   }

3. License Activation
   Module → Server: POST /drm/v2/activate
   {
     "licenseId": "cuid...",
     "installationId": "cuid...",
     "nonce": "random..."
   }
   
   Server → Module: Signed Lease
   {
     "protocolVersion": 2,
     "licenseId": "...",
     "installationId": "...",
     "resourceId": "...",
     "resourceVersionId": "...",
     "artifactHash": "sha256:...",
     "issuedAt": "2026-09-07T...",
     "expiresAt": "2026-09-14T...",
     "nonce": "...",
     "serverKeyId": "...",
     "capabilities": ["run", "update"],
     "signature": "base64..."
   }

4. Lease Verification (module side)
   - Verify server signature
   - Check expiry
   - Check nonce (no replay)
   - Check artifact hash
   - Check resource binding
```

#### 2. Database Schema (1 час)

```prisma
model Installation {
  id            String             @id @default(cuid())
  licenseId     String
  publicKey     String             @unique
  serverSerial  String?
  serverName    String?
  mtaVersion    String?
  moduleVersion String?
  status        InstallationStatus @default(ACTIVE)
  challenge     String?            // Pending verification
  verifiedAt    TimestamptzString?
  installedAt   TimestamptzString @default(now())
  lastHeartbeat TimestamptzString?
  revokedAt     TimestamptzString?

  license License @relation(...)
  leases  Lease[]
}

model Lease {
  id               String @id @default(cuid())
  installationId   String
  licenseId        String
  resourceVersionId String
  artifactHash     String
  nonce            String @unique
  issuedAt         TimestamptzString @default(now())
  expiresAt        TimestamptzString
  serverKeyId      String
  signature        String
  capabilities     Json
  lastVerifiedAt   TimestamptzString?
  
  installation Installation @relation(...)
  
  @@index([installationId])
  @@index([nonce])
  @@index([expiresAt])
}

model ServerSigningKey {
  id         String @id @default(cuid())
  keyType    String // ED25519
  publicKey  String
  algorithm  String
  createdAt  TimestamptzString @default(now())
  revokedAt  TimestamptzString?
  status     KeyStatus @default(ACTIVE)
}
```

#### 3. Server-side Implementation (6 часов)

**File:** `apps/server/src/lib/drm/v2.ts`

```typescript
// 1. Installation registration
export async function registerInstallation(input: {
  publicKey: string;
  mtaVersion: string;
  moduleVersion: string;
}): Promise<{
  installationId: string;
  challenge: string;
}>;

// 2. Challenge verification
export async function verifyInstallation(input: {
  installationId: string;
  challengeResponse: string;
}): Promise<{ verified: boolean }>;

// 3. Lease generation
export async function generateLease(input: {
  licenseId: string;
  installationId: string;
  nonce: string;
}): Promise<SignedLease>;

// 4. Lease signing
export async function signLease(lease: Lease): Promise<string>;

// 5. Nonce validation
export async function validateNonce(nonce: string): Promise<boolean>;
```

**File:** `apps/server/src/routes/drm/v2.ts`

```typescript
router.post('/v2/installations', registerInstallationHandler);
router.post('/v2/installations/:id/verify', verifyInstallationHandler);
router.post('/v2/activate', activateLicenseV2Handler);
router.post('/v2/verify', verifyLeaseHandler);
router.post('/v2/heartbeat', heartbeatHandler);
```

#### 4. Module-side Implementation (6 часов)

**Repository:** `mta-market-module`

**File:** `src/drm/v2/installation.cpp`

```cpp
// 1. Generate keypair
KeyPair generateInstallationKeypair();

// 2. Register installation
InstallationResponse registerInstallation(
  const std::string& publicKey,
  const std::string& mtaVersion
);

// 3. Sign challenge
std::string signChallenge(
  const std::string& challenge,
  const PrivateKey& privateKey
);

// 4. Verify lease signature
bool verifyLease(
  const SignedLease& lease,
  const std::string& serverPublicKey
);

// 5. Store lease locally
void storeLease(const SignedLease& lease);

// 6. Load lease on resource start
SignedLease loadLease(const std::string& resourceId);
```

#### 5. Protocol Tests (3 часа)

**File:** `apps/server/tests/drm-v2.test.ts`

```typescript
describe('DRM Protocol v2', () => {
  it('should register installation with public key');
  it('should verify challenge response');
  it('should reject invalid challenge response');
  it('should generate signed lease');
  it('should verify lease signature');
  it('should reject expired lease');
  it('should reject replayed nonce');
  it('should reject wrong installation');
  it('should reject wrong resource');
  it('should reject tampered artifact hash');
});
```

**File:** `mta-market-module/tests/drm_v2_test.cpp`

```cpp
TEST(DRMv2, GenerateKeypair);
TEST(DRMv2, SignChallenge);
TEST(DRMv2, VerifyLease);
TEST(DRMv2, RejectInvalidSignature);
TEST(DRMv2, RejectExpiredLease);
TEST(DRMv2, RejectWrongArtifactHash);
```

#### 6. Key Management (2 часа)

```bash
# Generate server signing key
pnpm drm:keygen --type server

# Rotate server key
pnpm drm:key-rotate --old-key-id <id>

# Revoke compromised key
pnpm drm:key-revoke --key-id <id>
```

### Блокеры

- ❌ C++ module development (нужен доступ к mta-market-module repo)
- ❌ MTA testing environment
- ❌ Key storage strategy (ENV? KMS?)
- ❌ Migration plan v1 → v2

---

## 📋 TASK-021: Module ↔ Site Compatibility Tests

**Приоритет:** 🟡 MEDIUM  
**Зависимости:** TASK-020 (DRM v2)  
**Оценка:** 6-8 часов  
**Статус:** ❌ NOT STARTED

### Описание

Создать cross-repo integration tests для проверки совместимости между module и site.

### Acceptance Criteria

- [ ] CI job для compatibility tests
- [ ] Protocol version compatibility matrix
- [ ] Backward compatibility tests
- [ ] Breaking change detection
- [ ] Documentation

### Детальный план

#### 1. Compatibility Matrix (1 час)

**File:** `mta-market-document/03-features/drm/compatibility.md`

```markdown
# DRM Protocol Compatibility Matrix

| Site Version | API Version | DRM Protocol | Module Version | Artifact Format |
|--------------|-------------|--------------|----------------|-----------------|
| 0.1.x        | v1          | v1           | 0.1.x - 0.3.x  | v1              |
| 0.2.x        | v1          | v2           | >= 0.4.x       | v1              |
| 1.0.x        | v2          | v2           | >= 0.5.x       | v2              |
```

#### 2. CI Integration (3 часа)

**File:** `.github/workflows/compatibility.yml`

```yaml
name: Protocol Compatibility

on:
  push:
    paths:
      - 'apps/server/src/routes/drm/**'
      - 'apps/server/src/lib/drm/**'

jobs:
  compatibility:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout site
        uses: actions/checkout@v4
      
      - name: Checkout module
        uses: actions/checkout@v4
        with:
          repository: acc-holo-dev/mta-market-module
          path: module
      
      - name: Build site
        run: pnpm install && pnpm build
      
      - name: Build module
        run: cd module && cmake . && make
      
      - name: Run compatibility tests
        run: pnpm test:compatibility
```

#### 3. Test Suite (4 часа)

**File:** `apps/server/tests/compatibility/drm-protocol.test.ts`

```typescript
describe('DRM Protocol Compatibility', () => {
  describe('v1 → v2 migration', () => {
    it('should support v1 module with v2 server');
    it('should reject unsupported protocol version');
  });
  
  describe('Breaking change detection', () => {
    it('should detect lease format changes');
    it('should detect signature algorithm changes');
    it('should detect artifact format changes');
  });
  
  describe('Backward compatibility', () => {
    it('should accept old module version');
    it('should accept old artifact format');
  });
});
```

### Блокеры

- ❌ Access to mta-market-module CI
- ❌ Test infrastructure setup

---

## 📋 TASK-022: Upload Sandbox

**Приоритет:** 🔴 HIGH (P0 Security)  
**Зависимости:** None  
**Оценка:** 12-16 часов  
**Статус:** ❌ NOT STARTED

### Описание

Создать isolated sandbox для безопасного выполнения seller uploads перед публикацией.

### Требования из PROMNT.md

```text
# 27. File upload security
# 28. Sandbox service

Implement:
- file size limit
- archive nesting limit
- compression ratio limit
- path traversal protection
- symlink rejection
- MIME/content validation
- malware/static scan hook
- sandbox execution

Never execute seller uploaded content on the main backend host.

Sandbox isolation minimum:
- non-root
- no host mounts
- no Docker socket
- no production secrets
- no production DB
- CPU/RAM/process limits
- ephemeral filesystem
```

### Acceptance Criteria

- [ ] Docker-based sandbox runner
- [ ] Static analysis (file validation)
- [ ] Dynamic execution (runtime test)
- [ ] Resource limits (CPU, RAM, network)
- [ ] Timeout enforcement
- [ ] Compatibility report generation
- [ ] Integration with upload pipeline
- [ ] Security tests

### Детальный план

#### 1. Sandbox Schema (1 час)

```prisma
model SandboxRun {
  id               String @id @default(cuid())
  versionId        String
  status           SandboxStatus
  startedAt        TimestamptzString @default(now())
  completedAt      TimestamptzString?
  timeoutSeconds   Int
  cpuLimit         Int
  memoryLimitMb    Int
  exitCode         Int?
  stdout           String?
  stderr           String?
  compatibilityReport Json?
  securityReport   Json?
  
  version ResourceVersion @relation(...)
}

enum SandboxStatus {
  PENDING
  RUNNING
  SUCCESS
  FAILED
  TIMEOUT
  SECURITY_VIOLATION
}
```

#### 2. Static Validation (3 часа)

**File:** `apps/server/src/lib/sandbox/static.ts`

```typescript
interface StaticValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
  fileCount: number;
  totalSize: number;
  suspiciousFiles: string[];
}

export async function validateArchive(file: Buffer): Promise<StaticValidationResult> {
  // 1. Size check
  if (file.length > MAX_UPLOAD_SIZE) {
    return { valid: false, errors: ['File too large'] };
  }
  
  // 2. Archive structure
  const entries = await extractArchive(file);
  
  // 3. Path traversal check
  for (const entry of entries) {
    if (entry.path.includes('..')) {
      return { valid: false, errors: ['Path traversal detected'] };
    }
  }
  
  // 4. Symlink check
  const symlinks = entries.filter(e => e.type === 'symlink');
  if (symlinks.length > 0) {
    return { valid: false, errors: ['Symlinks not allowed'] };
  }
  
  // 5. Compression ratio (bomb detection)
  const ratio = entries.reduce((sum, e) => sum + e.uncompressedSize, 0) / file.length;
  if (ratio > MAX_COMPRESSION_RATIO) {
    return { valid: false, errors: ['Compression bomb detected'] };
  }
  
  // 6. File count
  if (entries.length > MAX_FILE_COUNT) {
    return { valid: false, errors: ['Too many files'] };
  }
  
  // 7. Suspicious patterns
  const suspicious = detectSuspiciousFiles(entries);
  
  return {
    valid: true,
    errors: [],
    warnings: suspicious,
    fileCount: entries.length,
    totalSize: file.length,
    suspiciousFiles: suspicious
  };
}
```

#### 3. Sandbox Runner (6 часов)

**File:** `apps/server/src/lib/sandbox/runner.ts`

```typescript
interface SandboxRunOptions {
  artifact: Buffer;
  timeoutSeconds: number;
  cpuLimit: number;
  memoryLimitMb: number;
  networkAllowed: boolean;
}

interface SandboxRunResult {
  status: 'success' | 'failed' | 'timeout' | 'security_violation';
  exitCode?: number;
  stdout: string;
  stderr: string;
  duration: number;
  compatibilityReport: CompatibilityReport;
  securityIssues: SecurityIssue[];
}

export async function runSandbox(options: SandboxRunOptions): Promise<SandboxRunResult> {
  // 1. Create ephemeral Docker container
  const containerId = await createSandboxContainer({
    image: 'mta-sandbox:latest',
    cpus: options.cpuLimit,
    memory: `${options.memoryLimitMb}m`,
    network: options.networkAllowed ? 'bridge' : 'none',
    user: 'sandbox', // non-root
    readOnly: true,
    tmpfs: {
      '/tmp': 'rw,noexec,nosuid,size=100m'
    }
  });
  
  // 2. Copy artifact to container
  await docker.putArchive(containerId, '/sandbox/artifact.zip', options.artifact);
  
  // 3. Execute with timeout
  const exec = await docker.exec(containerId, {
    Cmd: ['/sandbox/run.sh'],
    AttachStdout: true,
    AttachStderr: true
  });
  
  const result = await promiseWithTimeout(
    exec.start(),
    options.timeoutSeconds * 1000
  );
  
  // 4. Collect logs
  const stdout = result.stdout.toString();
  const stderr = result.stderr.toString();
  
  // 5. Parse compatibility report
  const compatibilityReport = parseCompatibilityReport(stdout);
  
  // 6. Cleanup
  await docker.removeContainer(containerId, { force: true });
  
  return {
    status: result.exitCode === 0 ? 'success' : 'failed',
    exitCode: result.exitCode,
    stdout,
    stderr,
    duration: result.duration,
    compatibilityReport,
    securityIssues: []
  };
}
```

#### 4. Docker Sandbox Image (4 часа)

**File:** `apps/server/sandbox/Dockerfile`

```dockerfile
FROM ubuntu:22.04

# Install MTA Server (headless mode)
RUN apt-get update && apt-get install -y \
    wget \
    unzip \
    libstdc++6

# Download MTA Server
RUN wget https://linux.mtasa.com/dl/multitheftauto_linux_x64.tar.gz \
    && tar -xzf multitheftauto_linux_x64.tar.gz \
    && rm multitheftauto_linux_x64.tar.gz

# Create sandbox user
RUN useradd -m -s /bin/bash sandbox

# Create sandbox directories
RUN mkdir -p /sandbox && chown sandbox:sandbox /sandbox

# Copy test script
COPY run.sh /sandbox/run.sh
RUN chmod +x /sandbox/run.sh

USER sandbox
WORKDIR /sandbox

CMD ["/bin/bash"]
```

**File:** `apps/server/sandbox/run.sh`

```bash
#!/bin/bash
set -e

# 1. Extract artifact
unzip -q /sandbox/artifact.zip -d /sandbox/resource

# 2. Start MTA server in test mode
timeout 30s /opt/mta-server/mta-server --test-resource /sandbox/resource

# 3. Output compatibility report
cat /sandbox/resource/compatibility.json
```

#### 5. Integration with Upload (2 часа)

```typescript
// apps/server/src/lib/upload.ts

async function processUpload(file, seller) {
  // 1. Static validation
  const staticResult = await validateArchive(file);
  if (!staticResult.valid) {
    throw new Error(`Validation failed: ${staticResult.errors.join(', ')}`);
  }
  
  // 2. Upload to storage
  const url = await uploadToS3(file);
  
  // 3. Run sandbox
  const sandboxResult = await runSandbox({
    artifact: file,
    timeoutSeconds: 60,
    cpuLimit: 1,
    memoryLimitMb: 512,
    networkAllowed: false
  });
  
  if (sandboxResult.status !== 'success') {
    throw new Error(`Sandbox failed: ${sandboxResult.stderr}`);
  }
  
  // 4. Store results
  await prisma.sandboxRun.create({
    versionId,
    status: sandboxResult.status,
    compatibilityReport: sandboxResult.compatibilityReport,
    ...
  });
  
  return { url, sandboxResult };
}
```

### Блокеры

- ❌ MTA Server installation в Docker
- ❌ MTA headless/test mode
- ❌ Docker host access (production deployment)

---

## 📋 TASK-023: Compatibility Matrix

**Приоритет:** 🟡 MEDIUM  
**Зависимости:** TASK-022 (sandbox)  
**Оценка:** 4-6 часов  
**Статус:** ❌ NOT STARTED

### Описание

Создать систему compatibility declaration и verification для resources.

### Acceptance Criteria

- [ ] Compatibility schema в manifest
- [ ] MTA version detection
- [ ] OS/architecture support
- [ ] Dependency checking
- [ ] Compatibility badge UI
- [ ] Documentation

### Детальный план

#### 1. Compatibility Schema (1 час)

```typescript
interface Compatibility {
  mta: {
    min?: string;      // "1.5.0"
    max?: string;      // "1.6.0"
    tested: string[];  // ["1.5.9", "1.6.0"]
  };
  os: ('linux' | 'windows')[];
  architecture: ('x64' | 'x86')[];
  dependencies: Dependency[];
  requiredModules: string[];
  conflicts: Conflict[];
}

interface Dependency {
  resourceName: string;
  minVersion?: string;
  maxVersion?: string;
  optional: boolean;
}
```

#### 2. Compatibility Service (2 часа)

```typescript
export async function checkCompatibility(input: {
  resource: Resource;
  mtaVersion: string;
  os: string;
  architecture: string;
}): Promise<CompatibilityResult>;

export async function detectConflicts(resources: Resource[]): Promise<Conflict[]>;

export async function suggestCompatibleVersion(
  resource: Resource,
  constraints: Constraints
): Promise<ResourceVersion | null>;
```

#### 3. UI Components (1 час)

```tsx
// apps/web/components/CompatibilityBadge.tsx
export function CompatibilityBadge({ compatibility }) {
  return (
    <div>
      <Badge variant="success">MTA 1.5.9+</Badge>
      <Badge variant="info">Linux & Windows</Badge>
      <Badge variant="warning">Requires: map-editor</Badge>
    </div>
  );
}
```

#### 4. Tests (1 час)

```typescript
describe('Compatibility', () => {
  it('should detect compatible MTA version');
  it('should detect incompatible MTA version');
  it('should detect missing dependencies');
  it('should detect conflicts');
  it('should suggest alternative version');
});
```

---

## 📋 TASK-024: Update Signature + Rollback

**Приоритет:** 🟡 MEDIUM  
**Зависимости:** TASK-019 (artifact signing)  
**Оценка:** 8-10 часов  
**Статус:** ❌ NOT STARTED

### Описание

Реализовать signed updates и механизм rollback для безопасного обновления resources.

### Acceptance Criteria

- [ ] Update manifest signature
- [ ] Version downgrade protection
- [ ] Rollback mechanism
- [ ] Update verification (module side)
- [ ] Update policy (auto/manual)
- [ ] Health check post-update
- [ ] Documentation

### Детальный план

#### 1. Update Schema (1 час)

```prisma
model UpdatePolicy {
  id                String @id @default(cuid())
  resourceId        String @unique
  autoUpdate        Boolean @default(false)
  updateChannel     String // STABLE, BETA, ALPHA
  allowDowngrade    Boolean @default(false)
  healthCheckUrl    String?
  rollbackOnFailure Boolean @default(true)
  
  resource Resource @relation(...)
}

model UpdateHistory {
  id                 String @id @default(cuid())
  installationId     String
  fromVersionId      String
  toVersionId        String
  status             UpdateStatus
  startedAt          TimestamptzString @default(now())
  completedAt        TimestamptzString?
  rolledBackAt       TimestamptzString?
  healthCheckPassed  Boolean?
  errorMessage       String?
  
  installation Installation @relation(...)
}

enum UpdateStatus {
  PENDING
  DOWNLOADING
  INSTALLING
  HEALTH_CHECK
  SUCCESS
  FAILED
  ROLLED_BACK
}
```

#### 2. Update Service (3 часа)

```typescript
export async function checkForUpdates(installationId: string): Promise<UpdateInfo[]>;

export async function downloadUpdate(versionId: string): Promise<{
  manifest: ArtifactManifest;
  signature: string;
  artifact: Buffer;
}>;

export async function verifyUpdate(update: {
  manifest: ArtifactManifest;
  signature: string;
  publicKey: string;
}): Promise<boolean>;

export async function installUpdate(installationId: string, versionId: string): Promise<void>;

export async function rollbackUpdate(installationId: string): Promise<void>;
```

#### 3. Module Integration (3 часа)

```cpp
// mta-market-module/src/update/manager.cpp

class UpdateManager {
public:
  UpdateInfo checkForUpdates();
  bool downloadUpdate(const std::string& versionId);
  bool verifyUpdate(const UpdatePackage& package);
  bool installUpdate(const UpdatePackage& package);
  bool rollback();
  
private:
  std::string lastKnownGoodVersion_;
};
```

#### 4. Health Check (1 час)

```typescript
export async function runHealthCheck(installationId: string): Promise<{
  healthy: boolean;
  checks: HealthCheck[];
}>;

interface HealthCheck {
  name: string;
  passed: boolean;
  message?: string;
}
```

#### 5. API Endpoints (1 час)

```typescript
// GET /drm/v2/updates/:installationId
router.get('/v2/updates/:installationId', checkForUpdatesHandler);

// POST /drm/v2/updates/:installationId/download
router.post('/v2/updates/:installationId/download', downloadUpdateHandler);

// POST /drm/v2/updates/:installationId/install
router.post('/v2/updates/:installationId/install', installUpdateHandler);

// POST /drm/v2/updates/:installationId/rollback
router.post('/v2/updates/:installationId/rollback', rollbackHandler);

// POST /drm/v2/updates/:installationId/health
router.post('/v2/updates/:installationId/health', healthCheckHandler);
```

#### 6. Tests (1 час)

```typescript
describe('Updates', () => {
  it('should detect available update');
  it('should verify update signature');
  it('should install update');
  it('should run health check');
  it('should rollback on failure');
  it('should prevent downgrade (if policy)');
  it('should keep last known good version');
});
```

---

## 📋 TASK-025: E2E Test Suites

**Приоритет:** 🔴 HIGH  
**Зависимости:** All above  
**Оценка:** 12-16 часов  
**Статус:** ❌ NOT STARTED

### Описание

Создать comprehensive E2E test suites для всех user flows.

### Acceptance Criteria

- [ ] Buyer journey (register → purchase → download → install)
- [ ] Seller journey (register → upload → moderation → publish)
- [ ] Free resource flow
- [ ] Discount flow
- [ ] Service flow
- [ ] Multi-item cart
- [ ] Refund flow
- [ ] Dispute flow
- [ ] Update flow
- [ ] CI integration

### Детальный план

#### 1. Test Infrastructure (2 часа)

```typescript
// Setup Playwright or Cypress
// Seed database with test data
// Mock payment provider responses
// Mock MTA server environment
```

#### 2. Buyer E2E Tests (3 часа)

```typescript
describe('Buyer Journey', () => {
  it('should complete full purchase flow', async () => {
    // 1. Register via Discord
    await page.goto('/auth/discord');
    await loginWithDiscord(testUser);
    
    // 2. Browse catalog
    await page.goto('/catalog');
    await page.click('[data-testid="resource-card-1"]');
    
    // 3. Add to cart
    await page.click('[data-testid="add-to-cart"]');
    expect(await page.locator('[data-testid="cart-count"]').textContent()).toBe('1');
    
    // 4. Apply discount
    await page.goto('/cart');
    await page.fill('[data-testid="discount-code"]', 'TEST20');
    await page.click('[data-testid="apply-discount"]');
    
    // 5. Checkout
    await page.click('[data-testid="checkout"]');
    
    // 6. Payment (mocked)
    await mockPaymentSuccess();
    
    // 7. Verify purchase
    await page.goto('/dashboard');
    expect(await page.locator('[data-testid="purchase-1"]')).toBeVisible();
    
    // 8. Download
    await page.click('[data-testid="download-btn"]');
    
    // 9. Activate license (API call)
    const license = await activateLicense(testInstallation);
    expect(license.status).toBe('ACTIVE');
  });
});
```

#### 3. Seller E2E Tests (3 часа)

```typescript
describe('Seller Journey', () => {
  it('should upload and publish resource', async () => {
    // 1. Register as seller
    await registerSeller(testSeller);
    
    // 2. Create resource
    await page.goto('/dashboard/resources/new');
    await page.fill('[data-testid="title"]', 'Test Resource');
    await page.fill('[data-testid="description"]', 'Test description');
    await page.selectOption('[data-testid="type"]', 'GAMEMODE');
    await page.fill('[data-testid="price"]', '1000');
    
    // 3. Upload file
    await page.setInputFiles('[data-testid="file"]', 'test-resource.zip');
    
    // 4. Wait for validation
    await page.waitForSelector('[data-testid="validation-success"]');
    
    // 5. Submit for moderation
    await page.click('[data-testid="submit"]');
    
    // 6. Moderator approves (API)
    await moderateResource(resourceId, 'APPROVED');
    
    // 7. Verify published
    await page.goto(`/resources/${resourceSlug}`);
    expect(await page.locator('[data-testid="status"]').textContent()).toBe('Published');
  });
});
```

#### 4. Free Resource E2E (1 час)

```typescript
describe('Free Resource Flow', () => {
  it('should claim free resource without payment', async () => {
    // 1. Browse catalog
    await page.goto('/catalog?filter=free');
    
    // 2. View free resource
    await page.click('[data-testid="free-resource"]');
    expect(await page.locator('[data-testid="price"]').textContent()).toBe('Free');
    
    // 3. Claim
    await page.click('[data-testid="claim-btn"]');
    
    // 4. Verify entitlement (no payment)
    await page.goto('/dashboard');
    expect(await page.locator('[data-testid="free-resource-entitlement"]')).toBeVisible();
    
    // 5. Download
    const downloadBtn = await page.locator('[data-testid="download-btn"]');
    expect(downloadBtn).toBeEnabled();
  });
});
```

#### 5. Service E2E (2 часа)

```typescript
describe('Service Flow', () => {
  it('should complete service order and delivery', async () => {
    // 1. Browse services
    await page.goto('/services');
    
    // 2. Select service
    await page.click('[data-testid="service-card"]');
    
    // 3. Fill requirements
    await page.fill('[data-testid="requirements"]', 'Custom gamemode with racing');
    
    // 4. Add to cart
    await page.click('[data-testid="add-to-cart"]');
    
    // 5. Checkout + payment
    await checkoutAndPay();
    
    // 6. Seller marks in progress (API)
    await updateServiceStatus(orderId, 'IN_PROGRESS');
    
    // 7. Seller delivers
    await deliverService(orderId, {
      files: ['gamemode.zip'],
      notes: 'Delivered as requested'
    });
    
    // 8. Buyer accepts
    await page.goto('/dashboard/orders');
    await page.click('[data-testid="accept-delivery"]');
    
    // 9. Verify completed
    expect(await getServiceStatus(orderId)).toBe('COMPLETED');
  });
});
```

#### 6. CI Integration (1 час)

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
      redis:
        image: redis:7
      
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Build application
        run: pnpm build
      
      - name: Seed database
        run: pnpm db:seed:test
      
      - name: Run E2E tests
        run: pnpm test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-results
          path: test-results/
```

---

## 📊 Итоговая сводка оставшихся задач

### Распределение по приоритетам

**🔴 HIGH Priority (P0 Security):**
- TASK-019: Artifact Signing (12-16h)
- TASK-020: DRM Protocol v2 (16-20h)
- TASK-022: Upload Sandbox (12-16h)
- TASK-025: E2E Tests (12-16h)

**Подитог HIGH:** 52-68 часов (~2-3 недели)

**🟡 MEDIUM Priority:**
- TASK-018: Reconciliation Worker (8-10h)
- TASK-021: Compatibility Tests (6-8h)
- TASK-023: Compatibility Matrix (4-6h)
- TASK-024: Update + Rollback (8-10h)

**Подитог MEDIUM:** 26-34 часов (~1-1.5 недели)

**Итого:** 78-102 часов (~4-5 недель при 20ч/неделю)

---

## 🎯 Рекомендуемый порядок выполнения

### Phase 1: Security Foundations (2 недели)
1. **TASK-019** — Artifact Signing (12-16h)
2. **TASK-022** — Upload Sandbox (12-16h)
3. **TASK-020** — DRM Protocol v2 (16-20h)

**Результат:** Core security infrastructure ready

### Phase 2: Operations (1 неделя)
4. **TASK-018** — Reconciliation Worker (8-10h)
5. **TASK-023** — Compatibility Matrix (4-6h)
6. **TASK-021** — Compatibility Tests (6-8h)

**Результат:** Operational monitoring + compatibility system

### Phase 3: Updates & Testing (1 неделя)
7. **TASK-024** — Update + Rollback (8-10h)
8. **TASK-025** — E2E Tests (12-16h)

**Результат:** Update mechanism + comprehensive test coverage

---

## ⚠️ Критические блокеры

### Технические блокеры

1. **MTA Server integration**
   - Нужен headless/test mode для sandbox
   - Или mock MTA environment

2. **C++ Module development**
   - Доступ к mta-market-module repo
   - C++ build environment
   - MTA SDK/headers

3. **Key Management**
   - Decision: где хранить private keys (ENV? KMS? Vault?)
   - Key rotation policy
   - Backup strategy

4. **Docker in Production**
   - Sandbox требует Docker host access
   - Security implications
   - Resource limits

### Организационные блокеры

1. **Cross-repo coordination**
   - Module + Site compatibility testing
   - Breaking change communication
   - Version synchronization

2. **Legal/Compliance**
   - Payment provider agreements (T-Bank, Alfa-Bank)
   - Crypto payment policy approval
   - Terms of Service finalization

3. **Infrastructure**
   - Production environment setup
   - CI/CD pipeline expansion
   - Monitoring/alerting infrastructure

---

## 📈 Метрики прогресса

### Выполнено
- ✅ 17 задач (TASK-001 до TASK-017)
- ✅ 60% от PROMNT.md tasks
- ✅ 8/17 P0 security issues resolved

### Осталось
- ❌ 8 задач (TASK-018 до TASK-025)
- ❌ 40% от PROMNT.md tasks
- ❌ 9/17 P0 security issues remain

### До Production Ready
- ❌ Все 8 задач выше
- ❌ Infrastructure setup
- ❌ Legal compliance
- ❌ Security audit
- ❌ Load testing
- ❌ Documentation finalization

**Оценка до MVP:** 78-102 часа (4-5 недель)  
**Оценка до Production:** +40-60 часов (2-3 недели)  
**Итого:** 120-160 часов (6-8 недель)

---

## 🎯 Следующий шаг

**Рекомендую начать с:**

1. **TASK-019: Artifact Signing** — фундамент для безопасности
2. Параллельно: Решить блокеры (MTA integration, key management)
3. Затем: TASK-022 (Sandbox) → TASK-020 (DRM v2)

**Хотите, чтобы я начал с TASK-019?**

---

**Дата создания:** 2026-09-07  
**Следующее обновление:** После завершения TASK-018 или TASK-019
