# 🎯 MTA Market — Анализ Задач для Дальнейшей Разработки
## На основе документации mta-market-document

**Дата анализа:** 2026-09-07  
**Источники:** status.md, production-readiness.md, audit-2025-01.md  
**Метод:** Анализ без PROMNT.md, только актуальная документация проекта

---

## 📊 Текущий Статус Проекта

### Общая Оценка
- **Фаза:** MVP / Pre-Production (НЕ Production Ready!)
- **Production Readiness:** 1% (1 из 75 gates verified)
- **Backend Implementation:** 30%
- **Feature Completion:** 15%
- **Security:** 5.5/10 (8 из 17 P0 issues resolved)

### ✅ Что Работает
- Backend API skeleton
- Frontend pages (Next.js 15)
- Docker + CI/CD
- Discord OAuth2 (partial)
- Basic CRUD для resources
- DRM v1 prototype (symmetric, insecure)

### ❌ Критические Проблемы
- **17 P0 security issues** не решены
- Платежи небезопасны (обход оплаты)
- DRM v1 использует symmetric crypto (небезопасно)
- Нет sandbox для uploads
- Нет observability
- Financial ledger неполный

---

## 🔴 КРИТИЧЕСКИЕ ЗАДАЧИ (P0) — Must Fix Before Launch

### Категория: Security Fixes (9 оставшихся задач)

#### TASK-SEC-10: Input Validation для Всех Endpoints
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED (30% partial)  
**Оценка:** 10-12 часов  
**Риск:** Injection attacks, data corruption

**Описание:**
- Добавить Zod schemas для всех API endpoints
- Валидация req.body, req.params, req.query
- Rate limiting per endpoint
- CORS configuration

**Acceptance Criteria:**
- [ ] Zod schema для каждого endpoint
- [ ] Injection test suite (SQL, XSS, command injection)
- [ ] Input sanitization
- [ ] Type validation

---

#### TASK-SEC-15: State Machine Validation
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов  
**Риск:** Invalid state changes

**Описание:**
Создать transition functions для всех stateful entities:
```typescript
canTransitionPurchase(from, to)
canTransitionPayment(from, to)
canTransitionResource(from, to)
canTransitionLicense(from, to)
canTransitionDispute(from, to)
```

**Entities требующие validation:**
- Purchase (PENDING → COMPLETED → REFUNDED)
- Payment (CREATED → PENDING → SUCCESS/FAILED)
- Resource (DRAFT → SUBMITTED → UNDER_REVIEW → PUBLISHED)
- License (ACTIVE → SUSPENDED → REVOKED)
- Dispute (OPEN → IN_PROGRESS → RESOLVED)

**Acceptance Criteria:**
- [ ] canTransition* functions для всех entities
- [ ] Invalid transition test для каждого entity
- [ ] Audit log для transitions
- [ ] State machine diagrams в documentation

---

#### TASK-SEC-17: Rate Limiting
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов  
**Риск:** API abuse, DDoS

**Описание:**
Redis-backed rate limiting:
- Per user (100 req/min)
- Per IP (300 req/min)
- Per endpoint (auth: 5 req/min, upload: 10 req/hour)
- Webhook endpoints: separate limits

**Implementation:**
```typescript
// Use express-rate-limit + rate-limit-redis
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

const limiter = rateLimit({
  store: new RedisStore({
    client: redis
  }),
  windowMs: 60 * 1000,
  max: 100
});
```

**Acceptance Criteria:**
- [ ] Redis integration
- [ ] Per-user limits
- [ ] Per-IP limits
- [ ] Per-endpoint custom limits
- [ ] Rate limit bypass test
- [ ] 429 Too Many Requests response

---

### Категория: Domain Model (11 оставшихся задач)

#### TASK-DOM-01: Product Entity
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
Создать Product entity как абстракцию над Resource/Service:

```prisma
enum ProductType {
  RESOURCE
  SERVICE
}

model Product {
  id          String      @id @default(cuid())
  type        ProductType
  title       String
  description String
  slug        String      @unique
  pricing     Json        // { type: FREE|FIXED, amount: number }
  sellerId    String
  
  resource Resource?
  service  Service?
}
```

**Acceptance Criteria:**
- [ ] Product model в Prisma
- [ ] Migration script
- [ ] CRUD endpoints
- [ ] Tests

---

#### TASK-DOM-02 & DOM-03: Resource & Service Specialization
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Описание:**
Resource и Service как специализации Product:

```prisma
model Resource {
  id              String @id @default(cuid())
  productId       String @unique
  resourceType    ResourceType
  fileUrl         String?
  compatibility   Json
  
  product Product @relation(...)
}

model Service {
  id               String @id @default(cuid())
  productId        String @unique
  deliveryTime     Int    // days
  requirementsForm Json
  
  product Product @relation(...)
}
```

**Acceptance Criteria:**
- [ ] Resource/Service models
- [ ] Migration from current Resource
- [ ] Relationship with Product
- [ ] Tests

---

#### TASK-DOM-04: Identity Provider Model
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
Provider-neutral authentication:

```prisma
model Identity {
  id              String @id @default(cuid())
  userId          String
  provider        String // DISCORD, TELEGRAM, YANDEX, VK, GOOGLE
  providerSubject String // Discord ID, Telegram ID, etc.
  accessToken     String?
  refreshToken    String?
  email           String?
  
  user User @relation(...)
  
  @@unique([provider, providerSubject])
}
```

**Acceptance Criteria:**
- [ ] Identity model
- [ ] Multi-provider auth test
- [ ] Account linking test
- [ ] Migration from current User.discordId

---

#### TASK-DOM-05: DiscountCampaign
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Описание:**
```prisma
model DiscountCampaign {
  id          String   @id @default(cuid())
  code        String   @unique
  type        DiscountType
  value       Int
  minAmount   Int?
  maxUses     Int?
  currentUses Int      @default(0)
  validFrom   DateTime
  validUntil  DateTime
  sellerId    String?
  
  @@index([code])
}
```

**Acceptance Criteria:**
- [ ] Discount model
- [ ] Discount application logic
- [ ] Validation (min amount, usage limits)
- [ ] Tests

---

#### TASK-DOM-06: Order/OrderItem
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Описание:**
Immutable order snapshot:

```prisma
model Order {
  id            String @id @default(cuid())
  userId        String
  totalAmount   Int
  discountCode  String?
  discountAmount Int @default(0)
  finalAmount   Int
  status        OrderStatus
  createdAt     DateTime @default(now())
  
  items OrderItem[]
  payment Payment?
}

model OrderItem {
  id           String @id @default(cuid())
  orderId      String
  productId    String
  productTitle String
  unitPrice    Int
  quantity     Int
  subtotal     Int
  
  order Order @relation(...)
}
```

**Acceptance Criteria:**
- [ ] Order/OrderItem models
- [ ] Immutable snapshot на checkout
- [ ] Multi-item support
- [ ] Price calculation service
- [ ] Tests

---

#### TASK-DOM-07: Payment Abstraction
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
```prisma
enum PaymentProvider {
  YOOKASSA
  TBANK
  ALFABANK
  CRYPTO
  TEST
}

model Payment {
  id              String @id @default(cuid())
  orderId         String
  provider        PaymentProvider
  providerPaymentId String
  amount          Int
  currency        String @default("RUB")
  status          PaymentStatus
  metadata        Json
  
  order Order @relation(...)
}
```

**Acceptance Criteria:**
- [ ] Payment model с provider field
- [ ] Provider interface
- [ ] Provider swap test
- [ ] Migration

---

#### TASK-DOM-09: Entitlement
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
```prisma
model Entitlement {
  id          String @id @default(cuid())
  userId      String
  productId   String
  purchaseId  String
  grantedAt   DateTime @default(now())
  expiresAt   DateTime?
  scope       Json // { versions: 'all' | ['v1.0.0'], updates: boolean }
  
  @@unique([userId, productId])
}
```

**Acceptance Criteria:**
- [ ] Entitlement model
- [ ] Scope validation
- [ ] Version access check
- [ ] Tests

---

#### TASK-DOM-11: Ledger Accounts
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 12-16 часов

**Описание:**
Double-entry bookkeeping:

```prisma
model LedgerAccount {
  id       String @id @default(cuid())
  type     AccountType
  name     String
  balance  Int @default(0)
  currency String @default("RUB")
}

model LedgerEntry {
  id          String @id @default(cuid())
  accountId   String
  amount      Int
  type        EntryType // DEBIT | CREDIT
  referenceType String
  referenceId String
  description String
  createdAt   DateTime @default(now())
}
```

**Acceptance Criteria:**
- [ ] Ledger models
- [ ] Double-entry enforcement
- [ ] Invariant: sum(debits) = sum(credits)
- [ ] Transaction support
- [ ] Tests

---

#### TASK-DOM-12: Payout
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Описание:**
```prisma
model Payout {
  id          String @id @default(cuid())
  sellerId    String
  amount      Int
  currency    String @default("RUB")
  status      PayoutStatus
  method      PayoutMethod
  details     Json
  requestedAt DateTime @default(now())
  approvedAt  DateTime?
  paidAt      DateTime?
}
```

**Acceptance Criteria:**
- [ ] Payout model
- [ ] Payout creation logic
- [ ] Approval workflow
- [ ] Platform fee calculation
- [ ] Tests

---

### Категория: Authentication (8 задач)

#### TASK-AUTH-02: Telegram Login
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Описание:**
Telegram Login Widget integration

**Resources:**
- https://core.telegram.org/bots/telegram-login

**Acceptance Criteria:**
- [ ] Telegram OAuth flow
- [ ] Identity creation
- [ ] Login test
- [ ] Account linking (if user already has Discord)

---

#### TASK-AUTH-03: Yandex ID
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Resources:**
- https://yandex.ru/dev/id/doc/ru/

**Acceptance Criteria:**
- [ ] Yandex OAuth 2.0 flow
- [ ] Identity creation
- [ ] Login test

---

#### TASK-AUTH-04: VK ID
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Acceptance Criteria:**
- [ ] VK OAuth flow
- [ ] Identity creation
- [ ] Login test

---

#### TASK-AUTH-05: Google OpenID Connect
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Resources:**
- https://developers.google.com/identity/openid-connect/openid-connect

---

#### TASK-AUTH-06: Apple Sign In
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Resources:**
- https://developer.apple.com/documentation/signinwithapple

---

#### TASK-AUTH-07: Account Linking
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
Позволить пользователю связать несколько провайдеров с одним аккаунтом

**Acceptance Criteria:**
- [ ] Link new provider to existing user
- [ ] Unlink provider
- [ ] Primary provider selection
- [ ] Tests

---

#### TASK-AUTH-08: Account Recovery
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
- Email recovery flow
- Forgot password (if email auth added)
- Account access via alternative provider

**Acceptance Criteria:**
- [ ] Recovery mechanism
- [ ] Email verification
- [ ] Recovery test

---

### Категория: Payments (10 задач)

#### TASK-PAY-01: PaymentProvider Interface
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Описание:**
```typescript
interface IPaymentProvider {
  createPayment(order: Order): Promise<ProviderPayment>;
  getPayment(paymentId: string): Promise<ProviderPayment>;
  cancelPayment(paymentId: string): Promise<void>;
  refundPayment(paymentId: string, amount?: number): Promise<void>;
  handleWebhook(req: Request): Promise<WebhookResult>;
}
```

**Implementations:**
- YooKassaProvider
- TBankProvider
- TestProvider

**Acceptance Criteria:**
- [ ] Interface definition
- [ ] YooKassa adapter
- [ ] Provider swap test (mock)
- [ ] Tests

---

#### TASK-PAY-02 & PAY-03: YooKassa Production Implementation
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Описание:**
Полная интеграция YooKassa по официальной документации:

**Features:**
- Payment creation
- Webhook handling (IP whitelist + Basic Auth)
- Event idempotency (provider_payment_events table)
- Signature verification
- Status reconciliation (GET payment object)
- Refund support

**Resources:**
- https://yookassa.ru/developers/using-api/webhooks

**Acceptance Criteria:**
- [ ] Production-ready webhook handler
- [ ] Event idempotency table
- [ ] IP whitelist validation
- [ ] Amount/currency verification
- [ ] Webhook replay test
- [ ] Duplicate webhook test
- [ ] Tests

---

#### TASK-PAY-04: T-Bank Adapter
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Resources:**
- https://www.tbank.ru/business/online-payments/internet-acquiring/

**Acceptance Criteria:**
- [ ] T-Bank API integration
- [ ] PaymentProvider implementation
- [ ] Webhook handler
- [ ] Tests

---

#### TASK-PAY-07: Provider Event Persistence
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Описание:**
```prisma
model ProviderPaymentEvent {
  id              String @id @default(cuid())
  provider        PaymentProvider
  providerEventId String
  objectType      String
  objectId        String
  eventType       String
  payloadHash     String
  payload         Json
  receivedAt      DateTime @default(now())
  processedAt     DateTime?
  status          EventStatus
  attempts        Int @default(0)
  lastError       String?
  
  @@unique([provider, providerEventId])
}
```

**Acceptance Criteria:**
- [ ] Event persistence
- [ ] Idempotency enforcement
- [ ] Event replay detection
- [ ] Tests

---

#### TASK-PAY-08: Amount Verification
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Описание:**
Сравнение суммы в Order vs Payment provider

**Acceptance Criteria:**
- [ ] Amount comparison
- [ ] Currency validation
- [ ] Mismatch alert
- [ ] Amount mismatch test

---

#### TASK-PAY-10: Refund Handling
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Acceptance Criteria:**
- [ ] Refund model
- [ ] Provider refund API
- [ ] Ledger entries для refund
- [ ] Entitlement revocation
- [ ] Tests

---

### Категория: Features (8 задач)

#### TASK-FEAT-01: Free Resources
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
```typescript
enum PricingType {
  FREE
  FIXED
}

interface Pricing {
  type: PricingType;
  amount?: number; // null for FREE
}
```

**Flow для FREE:**
1. User clicks "Get Free"
2. Create Purchase (no Payment)
3. Create Entitlement
4. Create License (if required)
5. User can download

**Acceptance Criteria:**
- [ ] PricingType enum
- [ ] Free purchase flow (no payment)
- [ ] Entitlement creation
- [ ] Free purchase test
- [ ] Download test

---

#### TASK-FEAT-02: Fixed-Price Services
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Описание:**
Service product type (см. TASK-DOM-03)

**Acceptance Criteria:**
- [ ] Service CRUD endpoints
- [ ] Service order flow
- [ ] Tests

---

#### TASK-FEAT-03: Service Fulfillment
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Описание:**
```prisma
model ServiceOrder {
  id            String @id @default(cuid())
  orderId       String
  serviceId     String
  requirements  Json
  status        ServiceOrderStatus
  startedAt     DateTime?
  deliveredAt   DateTime?
  acceptedAt    DateTime?
  
  deliverables ServiceDeliverable[]
}

model ServiceDeliverable {
  id             String @id @default(cuid())
  serviceOrderId String
  fileUrl        String
  description    String
  deliveredAt    DateTime @default(now())
}
```

**States:**
- PENDING
- IN_PROGRESS
- DELIVERED
- ACCEPTED
- COMPLETED
- DISPUTED

**Acceptance Criteria:**
- [ ] Service order model
- [ ] State machine
- [ ] Seller delivery UI
- [ ] Buyer acceptance UI
- [ ] Tests

---

#### TASK-FEAT-06: Multi-Item Checkout
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Описание:**
Cart → Order с несколькими items

**Models:**
```prisma
model Cart {
  id        String @id @default(cuid())
  userId    String @unique
  items     CartItem[]
  updatedAt DateTime @updatedAt
}

model CartItem {
  id        String @id @default(cuid())
  cartId    String
  productId String
  quantity  Int @default(1)
  addedAt   DateTime @default(now())
}
```

**Flow:**
1. Add to cart
2. View cart
3. Apply discount
4. Checkout → Create Order
5. Payment
6. Order completion

**Acceptance Criteria:**
- [ ] Cart models
- [ ] POST /cart/add
- [ ] GET /cart
- [ ] DELETE /cart/items/:id
- [ ] POST /cart/checkout
- [ ] Discount application
- [ ] Multi-item test

---

#### TASK-FEAT-05: Price Calculation Service
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
```typescript
interface PriceCalculation {
  subtotal: number;
  discountAmount: number;
  discountCode?: string;
  total: number;
  items: {
    productId: string;
    unitPrice: number;
    quantity: number;
    subtotal: number;
  }[];
}

async function calculateOrderPrice(
  items: CartItem[],
  discountCode?: string
): Promise<PriceCalculation>;
```

**Acceptance Criteria:**
- [ ] Price calculation service
- [ ] Discount validation
- [ ] Min amount check
- [ ] Usage limit check
- [ ] Immutable snapshot в Order
- [ ] Tests

---

### Категория: DRM (10 задач)

**Примечание:** TASK-019 (Artifact Signing) и TASK-020 (DRM v2) уже выполнены сегодня (server-side)

#### TASK-DRM-01: DRM Protocol v2 Documentation
**Статус:** ✅ SPECIFIED (уже есть)  
**File:** `mta-market-document/03-features/drm/protocol-v2.md`

---

#### TASK-DRM-02-04: Module Integration (C++)
**Приоритет:** 🔴 CRITICAL  
**Статус:** BLOCKED (no repo access)  
**Оценка:** 10-14 часов

**Описание:**
- Installation keypair generation (module-side)
- Challenge signing
- Lease verification
- Artifact hash checking

**Blocker:** Нет доступа к mta-market-module repository

---

#### TASK-DRM-05: Artifact Manifest
**Статус:** ✅ DONE (завершено сегодня)

---

#### TASK-DRM-06: Artifact Signing
**Статус:** ✅ DONE (завершено сегодня)

---

#### TASK-DRM-07: Hash Verification (Module)
**Статус:** BLOCKED (module access)

---

#### TASK-DRM-08: Revocation
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Acceptance Criteria:**
- [ ] License revocation endpoint
- [ ] Installation revocation
- [ ] Lease invalidation
- [ ] Module check для revoked status
- [ ] Tests

---

#### TASK-DRM-09: Offline Grace Period
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Acceptance Criteria:**
- [ ] Grace period configuration (7 days default)
- [ ] Module offline mode
- [ ] Controlled degradation
- [ ] Tests

---

#### TASK-DRM-10: Compatibility Matrix
**Статус:** ✅ DONE (завершено сегодня)

---

### Категория: Financial (5 задач)

#### TASK-FIN-01 & FIN-02: Double-Entry Ledger
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED (20% basic logging exists)  
**Оценка:** 16-20 часов

**Описание:**
См. TASK-DOM-11 (Ledger Accounts)

**Accounts:**
- Assets:Cash (platform wallet)
- Assets:Escrow (seller pending)
- Revenue:Sales
- Revenue:Fees
- Liabilities:SellerPayable

**Acceptance Criteria:**
- [ ] Ledger implementation
- [ ] Invariant: Σdebits = Σcredits
- [ ] Transaction support
- [ ] Rollback support
- [ ] Ledger balance test
- [ ] Invariant test

---

#### TASK-FIN-03: Seller Payout
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Описание:**
См. TASK-DOM-12 (Payout)

**Flow:**
1. Seller requests payout
2. Admin reviews
3. Platform fee deduction
4. Payout creation
5. External payment (bank transfer, etc.)
6. Ledger entries

**Acceptance Criteria:**
- [ ] Payout request
- [ ] Approval workflow
- [ ] Platform fee calculation
- [ ] Ledger entries
- [ ] Tests

---

#### TASK-FIN-04: Platform Fee
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Описание:**
```typescript
const PLATFORM_FEE_PERCENT = 15; // 15%

function calculatePlatformFee(amount: number): number {
  return Math.floor(amount * PLATFORM_FEE_PERCENT / 100);
}
```

**Acceptance Criteria:**
- [ ] Fee calculation
- [ ] Fee accounting в ledger
- [ ] Fee deduction on payout
- [ ] Tests

---

#### TASK-FIN-05: Reconciliation
**Статус:** ✅ DONE (завершено сегодня)

---

### Категория: Operations (5 задач)

#### TASK-OPS-01: OpenTelemetry Tracing
**Приоритет:** 🟡 P1  
**Статус:** PLANNED (5% console logs only)  
**Оценка:** 10-12 часов

**Описание:**
- Distributed tracing
- Request correlation IDs
- Performance metrics
- Error tracking

**Acceptance Criteria:**
- [ ] OTEL SDK integration
- [ ] Trace propagation
- [ ] Span creation
- [ ] Trace export (Jaeger/Zipkin)
- [ ] Tests

---

#### TASK-OPS-02: Prometheus Metrics
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Acceptance Criteria:**
- [ ] Metrics endpoint /metrics
- [ ] Request duration histogram
- [ ] Request count counter
- [ ] Active connections gauge
- [ ] Business metrics (purchases, revenue)
- [ ] Metrics scrape test

---

#### TASK-OPS-03: Structured Logging
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 4-6 часов

**Acceptance Criteria:**
- [ ] JSON logs
- [ ] Log levels (ERROR, WARN, INFO, DEBUG)
- [ ] Context fields (userId, requestId)
- [ ] Log aggregation (ELK/Loki)
- [ ] Log parsing test

---

#### TASK-OPS-04: Error Tracking (Sentry)
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 3-4 часа

**Acceptance Criteria:**
- [ ] Sentry integration
- [ ] Error capture (backend)
- [ ] Error capture (frontend)
- [ ] User context
- [ ] Release tracking
- [ ] Error capture test

---

#### TASK-OPS-05: Backup/Restore
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 6-8 часов

**Acceptance Criteria:**
- [ ] Automated DB backups (daily)
- [ ] S3 bucket backups
- [ ] Restore procedure
- [ ] Backup retention policy
- [ ] Restore test

---

### Категория: Testing (5 задач)

#### TASK-TEST-01: Integration Tests (Payment Flow)
**Приоритет:** 🔴 CRITICAL  
**Статус:** PLANNED (10% partial)  
**Оценка:** 12-16 часов

**Scenarios:**
- [ ] Complete purchase flow
- [ ] Payment webhook handling
- [ ] Refund flow
- [ ] Multi-item cart
- [ ] Discount application
- [ ] Free resource claim

---

#### TASK-TEST-02: E2E Tests
**Статус:** ✅ DONE (завершено сегодня — 87 unit tests)

**Примечание:** 87 unit tests написаны, но full E2E suite нужен

---

#### TASK-TEST-03: Compatibility Tests (Module ↔ Site)
**Статус:** BLOCKED (module access)

---

#### TASK-TEST-04: Load Tests
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 8-10 часов

**Scenarios:**
- [ ] Webhook load (1000 req/s)
- [ ] Concurrent purchases
- [ ] Cart operations
- [ ] Download pressure

**Tools:** k6 или Artillery

---

#### TASK-TEST-05: Security Tests (OWASP Top 10)
**Приоритет:** 🟡 P1  
**Статус:** PLANNED  
**Оценка:** 10-12 часов

**Tests:**
- [ ] SQL Injection
- [ ] XSS
- [ ] CSRF
- [ ] Authentication bypass
- [ ] Authorization bypass
- [ ] Rate limiting bypass
- [ ] File upload exploits

---

## 📊 Сводная Статистика

### По Статусу

| Статус | Количество |
|--------|----------:|
| **PLANNED** | 58 |
| **IMPLEMENTED** (not verified) | 1 |
| **VERIFIED** | 9 |
| **BLOCKED** | 3 |
| **DONE (сегодня)** | 7 |
| **TOTAL** | **78** |

### По Приоритетам

| Приоритет | Количество |
|-----------|----------:|
| 🔴 P0 CRITICAL | 52 |
| 🟡 P1 Important | 20 |
| 🟢 P2 Nice to have | 6 |

### По Категориям

| Категория | Всего | Готово | Осталось |
|-----------|------:|-------:|---------:|
| Security | 17 | 8 | 9 |
| Domain Model | 12 | 1 | 11 |
| Authentication | 8 | 0 | 8 |
| Payments | 10 | 0 | 10 |
| Features | 8 | 0 | 8 |
| DRM | 10 | 3 | 4 (3 blocked) |
| Financial | 5 | 1 | 4 |
| Operations | 5 | 0 | 5 |
| Testing | 5 | 1 | 4 |

---

## 🎯 Рекомендованный План Выполнения

### Phase 1: Security & Domain Model (30-40 часов)

**Цель:** Закрыть критические P0 security issues и завершить domain model

**Приоритетные задачи:**
1. TASK-SEC-10: Input Validation (10-12ч)
2. TASK-SEC-15: State Machines (8-10ч)
3. TASK-SEC-17: Rate Limiting (6-8ч)
4. TASK-DOM-01-03: Product/Resource/Service (14-18ч)

**Результат:** Security hardened, domain model complete

---

### Phase 2: Cart & Checkout (28-36 часов)

**Цель:** Полностью работающая корзина и checkout flow

**Задачи:**
1. TASK-DOM-06: Order/OrderItem (10-12ч)
2. TASK-FEAT-06: Multi-Item Checkout (10-12ч)
3. TASK-FEAT-05: Price Calculation (6-8ч)
4. Frontend: Cart UI (не в списке, 6-8ч)

**Результат:** Users can add multiple items to cart and checkout

---

### Phase 3: Payment Integration (28-36 часов)

**Цель:** Production-ready платежи

**Задачи:**
1. TASK-PAY-01: PaymentProvider Interface (8-10ч)
2. TASK-PAY-02-03: YooKassa Full (10-12ч)
3. TASK-PAY-07: Event Persistence (4-6ч)
4. TASK-PAY-08: Amount Verification (4-6ч)
5. TASK-DOM-07: Payment Abstraction (6-8ч)

**Результат:** Real payments work securely

---

### Phase 4: Authentication (24-32 часа)

**Цель:** Multi-provider auth

**Задачи:**
1. TASK-DOM-04: Identity Model (6-8ч)
2. TASK-AUTH-02: Telegram (4-6ч)
3. TASK-AUTH-03: Yandex (4-6ч)
4. TASK-AUTH-07: Account Linking (6-8ч)
5. TASK-AUTH-08: Recovery (6-8ч)

**Результат:** Users can login via Telegram, Yandex, Discord

---

### Phase 5: Financial & Discounts (32-42 часа)

**Цель:** Complete financial system

**Задачи:**
1. TASK-DOM-11: Ledger (12-16ч)
2. TASK-FIN-03: Seller Payout (10-12ч)
3. TASK-FIN-04: Platform Fee (6-8ч)
4. TASK-DOM-05: Discounts (8-10ч)

**Результат:** Financial system production-ready

---

### Phase 6: Services & Free Resources (24-30 часов)

**Задачи:**
1. TASK-FEAT-01: Free Resources (6-8ч)
2. TASK-FEAT-02: Service Product Type (8-10ч)
3. TASK-FEAT-03: Service Fulfillment (10-12ч)

**Результат:** Marketplace supports both resources and services

---

### Phase 7: Operations & Testing (40-50 часов)

**Задачи:**
1. TASK-OPS-01-04: Observability (23-30ч)
2. TASK-TEST-01: Integration Tests (12-16ч)
3. TASK-TEST-04-05: Load & Security Tests (18-22ч)

**Результат:** Production monitoring and comprehensive tests

---

## ⚠️ Критические Блокеры

### Блокер #1: Module Repository Access
**Затронутые задачи:**
- TASK-DRM-02-04: Module Integration (10-14ч)
- TASK-DRM-07: Hash Verification (4-6ч)
- TASK-TEST-03: Compatibility Tests (6-8ч)

**Total blocked:** 20-28 часов

**Решение:** Получить доступ к mta-market-module repo

---

### Блокер #2: Payment Provider Credentials
**Затронутые задачи:**
- TASK-PAY-02-03: YooKassa Production (нельзя полностью протестировать)
- TASK-PAY-04: T-Bank (нельзя начать)

**Решение:** 
- Зарегистрировать мерчанта в YooKassa
- Получить API credentials

---

## 💡 Моя Рекомендация

**Начать с Phase 1: Security & Domain Model**

**Почему:**
1. Закрывает критические P0 security issues
2. Нет внешних блокеров
3. 30-40 часов работы = 1-2 недели
4. Критично для production readiness
5. Создаст фундамент для остальных фич

**Первые задачи:**
1. TASK-SEC-10: Input Validation (10-12ч)
2. TASK-DOM-01-03: Product/Resource/Service (14-18ч)
3. TASK-SEC-15: State Machines (8-10ч)

---

## 📈 Прогноз до Production

**Минимальный путь (только P0):** 200-260 часов (~2-3 месяца при 30ч/неделю)

**Production-ready (P0 + P1):** 280-350 часов (~3-4 месяца)

**С учетом блокеров:** +20-28 часов после получения доступа к module

---

## ✅ Следующий Шаг

**Хотите, чтобы я начал с Phase 1?**

Я могу автономно выполнить:
1. TASK-SEC-10: Input Validation
2. TASK-DOM-01-03: Product/Resource/Service models
3. TASK-SEC-15: State Machine validation
4. TASK-SEC-17: Rate Limiting

**Или выберите другой приоритет — скажите с чего начать!**

---

**Дата создания:** 2026-09-07  
**Источник:** mta-market-document (status.md, production-readiness.md, audit-2025-01.md)  
**Статус:** Готов к выполнению  
**Следующий шаг:** Ваше решение
