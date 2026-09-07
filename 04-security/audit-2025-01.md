# MTA Market — полный технический аудит и целевой план проекта

**Дата:** 7 сентября 2026

**Объект:** `acc-holo-dev/mta-market`

**Дополнительный материал:** внешний аудит `mta-market-audit.md`

**Статус документа:** сводный аудит + целевая архитектура + backlog + roadmap

---

## 0. Executive summary

MTA Market уже нельзя считать просто концептом: в репозитории есть monorepo, backend/frontend, Prisma schema, Docker/CI, а также отдельный `mta-guard-module` с реальной интеграцией с MTA. При этом публичное описание проекта завышает степень готовности: в README проект помечен как `v1.0.0 / production-ready`, тогда как код содержит dev/simulate paths, незавершённые security-механизмы и существенные расхождения с целевой архитектурой.

Главная задача сейчас — не добавлять новые фичи, а сделать один непротиворечивый production flow:

```text
Seller
  -> upload
  -> static validation
  -> sandbox validation
  -> artifact hash/signature
  -> moderation
  -> publication

Buyer
  -> compatibility check
  -> checkout
  -> real payment
  -> verified provider event
  -> purchase
  -> license
  -> installation key
  -> signed lease
  -> authorized artifact download
  -> verify
  -> run
  -> update / rollback
```

### Итоговая оценка

| Область | Оценка | Вывод |
|---|---:|---|
| Продуктовая идея | 8.5/10 | Ниша и ценность понятны |
| Общая архитектура | 7.5/10 | Modular monolith — правильный фундамент |
| Backend implementation | 6/10 | Есть рабочая база, но много MVP-заглушек |
| БД / доменная модель | 7.5/10 | Значительно лучше первоначальной версии |
| DRM feasibility | 8/10 | Proof-of-feasibility есть |
| Production DRM | 4/10 | Cryptography/lease pipeline ещё не доведён |
| Payments | 4/10 | До реальных денег не готово |
| Security | 5.5/10 | Есть хорошие решения, но несколько critical issues |
| Ops / observability | 4/10 | Недостаточно для production |
| Documentation consistency | 4/10 | Несколько поколений архитектуры сосуществуют |
| MVP scope | 7/10 | После сокращения roadmap стал разумнее |

### Главный вывод

Проект находится в состоянии **advanced MVP / pre-production**, а не production-ready.

---

# 1. Что использовано для аудита

Сводка объединяет:

1. фактическое состояние репозитория `mta-market`;
2. отдельный `mta-guard-module`;
3. проектную документацию в `document/`;
4. предоставленный дополнительный статический аудит;
5. официальную документацию MTA;
6. актуальную документацию ЮKassa по webhook, split payments, safe deal и финансовой сверке.

Предоставленный аудит особенно важен тем, что в нём указаны конкретные файлы и строки текущей реализации. В нём зафиксированы критические проблемы с `POST /purchases/:id/complete`, webhook ЮKassa, `parseInt()` над CUID, DRM activation, moderation, токенами, скачиванием файлов, JWT fallback secret, rate limiting, upload validation, ledger и pagination. fileciteturn0file0L15-L38 fileciteturn0file0L40-L66

Отдельный внешний аудит также отмечает, что целевая модель с Ed25519, encrypted blobs, installation DEK, 2FA, escrow и forensic watermarking существенно опережает текущую реализацию. fileciteturn0file0L9-L12

---

# 2. Первое обязательное исправление: определить реальный статус проекта

## Проблема

README и CHANGELOG позиционируют проект как `v1.0.0 production-ready`, но код содержит:

- payment simulation;
- purchase completion без проверки оплаты;
- неполную DRM security model;
- незавершённый avatar flow;
- заглушку ledger;
- публичную раздачу upload-файлов;
- несогласованные ID-типы;
- документацию, описывающую более зрелую систему, чем реализация.

Дополнительный аудит прямо фиксирует это как главный разрыв между заявленным и фактическим состоянием. fileciteturn0file0L9-L12 fileciteturn0file0L135-L153

## Решение

Заменить статус:

```text
v1.0.0 — Production-ready
```

на:

```text
MVP / Pre-production
```

или:

```text
Private Beta
```

После прохождения production gates статус можно вернуть в `Production-ready`.

---

# 3. P0 — критические уязвимости и функциональные блокеры

## 3.1. Полный обход оплаты через `/purchases/:id/complete`

Файл: `apps/server/src/routes/purchases.ts`.

По предоставленному аудиту endpoint позволяет аутентифицированному покупателю завершить purchase без проверки реального платежа. Получается:

```text
create purchase
    -> complete purchase
    -> active license
```

без YooKassa.

Это означает прямой обход оплаты.

Предоставленный аудит классифицирует это как критическую дыру и отмечает, что в отличие от `/payments/:id/simulate`, endpoint не закрыт environment flag. fileciteturn0file0L15-L25

### Требование

Production flow должен быть только:

```text
payment created
 -> provider payment
 -> provider confirms success
 -> webhook / verified GET
 -> transactional settlement
 -> purchase completion
 -> license creation
```

`POST /purchases/:id/complete`:

- удалить из production;
- либо оставить только в отдельном dev/test router;
- либо сделать невозможным при `NODE_ENV=production`.

### Тест

Обязательный integration test:

```text
create purchase
POST /purchases/:id/complete
without successful payment
=> 403/409
license count unchanged
```

---

## 3.2. Webhook ЮKassa реализован не по реальному протоколу

Предоставленный аудит указывает, что код ожидает `x-yookassa-signature` + HMAC-SHA256, тогда как официальная документация ЮKassa для HTTP Basic Auth webhook flow описывает проверку подлинности через текущий статус объекта и/или IP-адрес отправителя; отдельного HMAC header для этого сценария документация не требует. ЮKassa также требует отвечать HTTP 200 на валидное уведомление, иначе продолжает доставку. fileciteturn0file0L27-L38 citeturn164063search4

### Правильный pipeline

```text
POST /webhooks/yookassa
        |
        +-- validate request source / trusted proxy setup
        |
        +-- persist event idempotently
        |
        +-- return 200 quickly
        |
        v
queue / worker
        |
        +-- GET provider object by payment_id
        +-- verify current status
        +-- verify amount
        +-- verify currency
        +-- verify metadata/order id
        +-- verify expected seller/purchase
        |
        v
transaction
        |
        +-- payment state
        +-- ledger
        +-- purchase
        +-- entitlement/license
```

ЮKassa прямо рекомендует при получении уведомления проверить текущий статус объекта и/или IP отправителя. Для HTTP Basic Auth уведомления настраиваются в личном кабинете. citeturn164063search4

### Дополнительное требование

Обязательно хранить:

```text
payment_provider_events
- id
- provider
- provider_event_id / object_id + event
- object_type
- event_type
- payload_hash
- received_at
- processed_at
- status
- attempts
- last_error
```

И уникальный constraint на событие, чтобы 10 одинаковых webhook'ов давали ровно один business effect.

---

## 3.3. Типы ID сломаны после перехода на CUID

Предоставленный аудит нашёл системное противоречие:

```text
Prisma:
User.id / Purchase.id / License.id / Resource.id = String CUID

Code:
JWT userId = number
parseInt(req.params.id)
```

и перечисляет 12+ мест, включая `payments.ts`, `admin.ts`, `drm.ts`, `purchases.ts`. fileciteturn0file0L52-L66

### Решение

Во всём backend:

```ts
userId: string
purchaseId: string
licenseId: string
resourceId: string
reviewId: string
```

Удалить `parseInt()` для database IDs.

Использовать runtime validation:

```ts
z.string().min(1)
```

или route-specific schema.

### Обязательный test

Каждый route должен тестироваться с реальным CUID из test DB.

Не только TypeScript compilation.

---

## 3.4. DRM activation нельзя строить вокруг ID лицензии

Текущий `/drm/activate` по дополнительному аудиту:

- не требует authentication;
- пытается трактовать license ID как число;
- связывает activation с `serverSerial`;
- сам код помечает подход как simplified. fileciteturn0file0L40-L50

### Целевая модель

```text
license
  |
  +-- license_installation
          |
          +-- installation_id
          +-- public_key
          +-- status
          +-- created_at
```

Первая активация:

```text
client generates Ed25519 keypair
private key stays local
public key -> server
```

Server выдаёт activation/installation record.

Все последующие запросы:

```text
challenge
    -> sign with installation private key
    -> verify public key
```

### Важно

`licenseKey` не должен быть одновременно:

```text
license database primary key
```

и:

```text
secret authentication credential
```

Нужен отдельный opaque activation credential/cryptographic protocol.

---

# 4. P0 — управление ресурсами и supply chain

## 4.1. Seller может выставить `PUBLISHED`

Дополнительный аудит отмечает, что `PATCH /resources/:slug` позволяет владельцу ресурса изменить статус, потенциально минуя moderation. fileciteturn0file0L72-L80

### Должно быть

Seller:

```text
DRAFT -> SUBMITTED
```

Moderator:

```text
SUBMITTED -> UNDER_REVIEW -> APPROVED -> PUBLISHED
```

Seller никогда не должен иметь право самостоятельно устанавливать:

```text
PUBLISHED
APPROVED
```

Также желательно запретить seller менять:

```text
moderation fields
security scan result
compatibility verification result
artifact hash
publisher signature
```

---

## 4.2. Сделать immutable artifact pipeline

После публикации версия ресурса должна быть неизменяемой.

```text
upload
 -> normalize
 -> hash
 -> scan
 -> validate
 -> sign
 -> moderation
 -> publish
```

После `PUBLISHED` нельзя перезаписать blob.

Новая модификация = новая `resource_version`.

---

## 4.3. Artifact manifest

Каждая версия должна иметь manifest:

```json
{
  "resource_id": "...",
  "version": "1.4.2",
  "artifact_sha256": "...",
  "publisher_id": "...",
  "build_id": "...",
  "mta": {
    "min": "1.6.0",
    "max": "1.7.x"
  },
  "os": ["windows-x64", "linux-x64"],
  "dependencies": [],
  "features": [],
  "key_id": "...",
  "signature": "..."
}
```

Это станет основанием для:

- DRM;
- compatibility;
- updates;
- rollback;
- moderation;
- forensic analysis.

---

# 5. P0 — authentication/session security

## 5.1. Токены нельзя передавать в URL

Дополнительный аудит указывает на flow:

```text
/auth/callback?access_token=...&refresh_token=...
```

и последующее хранение обоих токенов в `localStorage`. Это создаёт лишние места утечки через browser history, proxy logs, referrer/logging и XSS-доступ к `localStorage`. Также отмечено хранение refresh token в БД в открытом виде и отсутствие явного типа access/refresh token. fileciteturn0file0L82-L91

### Целевая схема

Для web:

```text
Access token:
short lived
in memory / controlled client storage

Refresh token:
HttpOnly
Secure
SameSite
cookie
```

Refresh token в БД:

```text
SHA-256/HMAC hash only
```

и:

```text
session_id
user_id
token_family
issued_at
expires_at
rotated_at
revoked_at
last_used_at
```

### Token type

JWT access:

```json
{
  "sub": "user_cuid",
  "type": "access",
  "iat": 0,
  "exp": 0
}
```

Refresh:

```json
{
  "sub": "user_cuid",
  "type": "refresh",
  "sid": "...",
  "iat": 0,
  "exp": 0
}
```

Каждая verifier-функция обязана проверять `type`.

---

## 5.2. JWT secret fallback нельзя оставлять

Дополнительный аудит отмечает:

```ts
process.env.JWT_SECRET || "dev_secret_change_in_production"
```

Это означает, что отсутствие environment variable не приводит к отказу старта. fileciteturn0file0L107-L108

### Решение

В production:

```text
JWT_SECRET missing
=> process exits immediately
```

То же правило для:

```text
DRM signing keys
cookie secrets
OAuth secrets
S3 credentials
YooKassa credentials
```

---

# 6. P0 — file delivery и DRM downloads

Дополнительный аудит выявляет две проблемы:

1. `getS3PublicUrl()` даёт постоянную публичную ссылку;
2. `getS3DownloadUrl()` использует `PutObjectCommand` вместо `GetObjectCommand`;
3. при `S3_ENABLED=false` `/uploads` раздаётся через `express.static()` без authentication. fileciteturn0file0L93-L101

### Целевая модель

Никогда не делать:

```text
GET /uploads/secret-resource.zip
```

через public static.

Вместо этого:

```text
GET /resources/:version/download
```

Flow:

```text
authenticate
 -> check purchase/license
 -> check entitlement
 -> create short-lived signed URL
 -> return URL
```

TTL:

```text
1–5 minutes
```

Объект storage:

```text
private bucket
```

Не public bucket.

---

# 7. DRM architecture v2

## 7.1. Что уже правильно

Текущая архитектура уже ушла от примитивного HWID к installation identity и рассматривает:

```text
Ed25519
AES-GCM
DEK
lease
installation public key
```

Отдельный `mta-guard-module` также имеет реальный integration layer, что является большим плюсом.

## 7.2. Что надо довести

### Resource version key

Лучший scope:

```text
1 resource version = 1 DEK
```

Например:

```text
resource A / 1.0 -> DEK-A1
resource A / 1.1 -> DEK-A2
```

### Lease

Lease должен включать минимум:

```text
license_id
installation_id
resource_id
resource_version_id
issued_at
expires_at
nonce
key_id
capabilities
signature
```

Это предотвращает перенос валидного lease на другой artifact.

### Cryptographic chain

```text
artifact hash
        |
        v
artifact signature
        |
        v
license entitlement
        |
        v
installation public key
        |
        v
signed lease
```

---

# 8. Почему DRM никогда не даст абсолютную защиту

Пользователь, на чьём сервере исполняется resource, контролирует машину.

Во время исполнения plaintext/decoded code в той или иной форме оказывается в memory.

Поэтому обещание должно быть:

```text
DRM предотвращает удобное массовое копирование,
контролирует entitlement,
усложняет reverse engineering,
помогает атрибутировать утечки.
```

Не:

```text
ресурс невозможно украсть.
```

Это важнейшее продуктовое уточнение.

---

# 9. Financial architecture

## 9.1. Purchase, payment, license и payout — разные сущности

Нельзя смешивать:

```text
Order
Payment
Purchase / Entitlement
License
Settlement
Payout
Refund
Dispute
```

Рекомендуемые определения:

```text
Order
= коммерческое намерение/корзина

Payment
= операция оплаты у payment provider

Purchase
= факт покупки товара / entitlement

License
= техническое право использования

Settlement
= распределение финансового результата

Payout
= фактическая выплата продавцу

Refund
= возврат средств
```

---

## 9.2. Выбрать ровно одну payment model

До production нужно подтвердить юридическую и техническую модель с ЮKassa.

### Split Payments

ЮKassa официально описывает marketplace-сценарий, в котором платформа принимает один платёж, передаёт распределение средств между магазинами и комиссию, а деньги распределяются между продавцами. Для продавцов требуется подключение к ЮKassa. citeturn164063search0turn164063search1

### Safe Deal

ЮKassa также описывает Безопасную сделку: деньги покупателя замораживаются до подтверждения/отмены сделки, после чего отправляются продавцу или возвращаются покупателю. citeturn164063search5

### Решение для MTA Market

Если продукт продаёт готовые цифровые ресурсы и хочет классический marketplace settlement, **Split Payments выглядит естественным вариантом**, но это должно быть подтверждено договором/онбордингом ЮKassa и вашей юридической моделью.

Если нужен именно escrow-подобный процесс с удержанием средств до подтверждения, Safe Deal ближе по семантике.

Нельзя проектировать финальную бизнес-логику вокруг собственного поля:

```text
escrow_status = HELD
```

без связи с реальным состоянием денег у провайдера.

---

# 10. Ledger: довести до бухгалтерски корректной модели

В текущем коде `balanceAfter` отмечен как TODO и остаётся нулём; дополнительный аудит также отмечает отсутствие полноценной double-entry модели, которая заявлена проектной документацией. fileciteturn0file0L119-L120

## Целевая модель

```text
ledger_accounts
ledger_transactions
ledger_entries
```

Например accounts:

```text
platform_cash
seller_pending
seller_available
platform_fee
provider_fee
refunds
adjustments
```

Каждая финансовая операция:

```text
transaction
  -> entries
```

Идея:

```text
SUM(debits) == SUM(credits)
```

`balance_after` можно оставить только как materialized/cache value.

### Обязательные инварианты

```text
captured payment
= seller share + platform fee + provider fee adjustments
```

```text
refund <= captured amount
```

```text
payout <= seller available balance
```

```text
one provider event
=> one financial effect
```

---

# 11. Financial reconciliation

Для marketplace reconciliation должен быть не только manual admin endpoint.

ЮKassa для split payments прямо предусматривает ежедневные реестры успешных платежей и возвратов для сверки расчётов. citeturn164063search3

Нужен nightly job:

```text
provider data
     |
     v
reconciliation
     |
     +-- payments
     +-- refunds
     +-- payouts / transfers
     +-- fees
     +-- receipts
```

Любое расхождение:

```text
ALERT
+ reconciliation case
```

---

# 12. Seller / buyer / admin model

## Ошибка

`USER / ADMIN / MODERATOR` слишком грубо для дальнейшего роста.

### Рекомендация

Разделить:

```text
User
SellerProfile
Roles
Permissions
```

Пример permissions:

```text
resource.create
resource.edit
resource.submit
resource.publish
resource.moderate
license.revoke
refund.create
payout.view
payout.manage
finance.reconcile
user.manage
```

Роли:

```text
USER
SELLER
MODERATOR
SUPPORT
FINANCE
ADMIN
SUPERADMIN
```

Не нужно реализовывать все сразу. Но доменная модель должна позволять это.

---

# 13. Moderation architecture

## Resource state machine

```text
DRAFT
  -> SUBMITTED
  -> UNDER_REVIEW
  -> APPROVED
  -> PUBLISHED
```

Отдельные terminal/exception states:

```text
REJECTED
SUSPENDED
ARCHIVED
YANKED
```

## Moderation event log

```text
resource_moderation_events
- resource_id
- moderator_id
- from_status
- to_status
- reason
- metadata
- created_at
```

Текущий `status` показывает состояние.

История объясняет, как оно появилось.

---

# 14. Upload security

Allowlist расширений — полезная первая защита, но недостаточная. Дополнительный аудит отмечает, что сейчас проверка в основном ориентирована на extension, без полноценной content validation/static scanning. fileciteturn0file0L116-L118

## Pipeline

```text
Upload
 -> max size
 -> max file count
 -> archive parser
 -> path traversal check
 -> symlink check
 -> nested archive limits
 -> compression ratio check
 -> content/MIME validation
 -> static scan
 -> meta.xml parse
 -> sandbox
 -> artifact hash
```

Защита от:

```text
ZIP Slip
ZIP bomb
nested archive bomb
symlink traversal
huge file count
malformed archive
unexpected binary
runtime downloader
```

---

# 15. Sandbox

Это один из главных security boundaries проекта.

Seller resource — потенциально враждебный код.

Нельзя запускать его в той же среде, где находятся:

```text
production secrets
PostgreSQL
Redis
S3 credentials
Docker socket
host filesystem
```

## Минимум

```text
non-root
CPU limit
RAM limit
PID limit
network deny/default deny
no host mounts
no secrets
read-only base FS
ephemeral writable FS
timeout
hard cleanup
```

Для высокорискового hostile execution позже рассмотреть VM/stronger isolation вместо ставки только на container boundary.

---

# 16. Resource dependency graph

Добавить dependency manifest:

```text
Resource A
  ├─ mysql
  ├─ dx
  └─ my-lib >= 2.4
```

При установке:

```text
dependency resolution
 -> missing dependency
 -> version conflict
 -> incompatible MTA version
```

Это можно превратить в одну из основных фич marketplace.

---

# 17. Compatibility system

Текущий `min_mta_version` — хороший старт, но нужно расширить.

## Compatibility manifest

```json
{
  "mta": {
    "min": "1.6.0",
    "max": "1.7.x"
  },
  "os": ["windows-x64", "linux-x64"],
  "dependencies": [],
  "native_modules": [],
  "db": ["mysql"],
  "tested": true
}
```

## UI

```text
Compatibility

MTA 1.6      ✅
MTA 1.7      ✅
Windows      ✅
Linux        ✅
MySQL        ✅
Custom C++   ⚠️
```

---

# 18. Verified Resource

Это потенциально сильнее обычного рейтинга.

Показывать:

```text
VERIFIED

Structure         ✅
Static scan       ✅
MTA compatibility ✅
Windows           ✅
Linux             ✅
Dependencies      ✅
Artifact hash     ✅
Publisher         ✅
Last verification <date>
```

Это должно стать частью marketplace trust model.

---

# 19. Artifact signing и supply-chain security

Новая release pipeline:

```text
seller upload
 -> artifact hash
 -> build ID
 -> security scan
 -> moderation
 -> publisher signature
 -> publish
```

Client side:

```text
download
 -> hash verification
 -> publisher signature verification
 -> license verification
 -> install
```

Это защищает не только от пиратства, но и от компрометации seller account.

---

# 20. Самая опасная будущая атака: compromised seller account

Сценарий:

```text
seller account hacked
 -> attacker publishes malicious update
 -> auto-update sends it to hundreds of servers
```

Поэтому auto-update нельзя строить как:

```text
latest.zip
```

Нужно:

```text
version manifest
 + signed artifact
 + moderation approval
 + compatibility result
 + release state
```

И update system должен поддерживать rollback.

---

# 21. Release channels

Предлагается:

```text
STABLE
BETA
LEGACY
```

Новая версия:

```text
DRAFT
 -> CANDIDATE
 -> VERIFIED
 -> PUBLISHED
```

Критическая ошибка:

```text
PUBLISHED
 -> YANKED
```

Yanked version не устанавливается новым пользователям, но уже установленную версию можно не ломать автоматически.

---

# 22. Updates и rollback

Целевой процесс:

```text
manifest
 -> download
 -> verify hash/signature
 -> backup current
 -> install new
 -> start resource
 -> health check
```

Если health check fails:

```text
rollback
```

Это обязательная функция до production auto-update.

---

# 23. Purchase model

В purchase должен оставаться snapshot данных на момент покупки:

```text
product_title_snapshot
seller_id_snapshot
price_cents
currency
platform_fee_cents
provider_fee_cents
seller_amount_cents
resource_version_id
```

Текущая схема уже хорошо движется в эту сторону. Дополнительный аудит отдельно отмечает наличие `priceSnapshot` как правильное решение. fileciteturn0file0L148-L153

---

# 24. Refund / entitlement policy

Нужно отдельно решить продуктовое правило:

> что происходит с лицензией после refund?

Предлагаемая схема:

```text
refund initiated
 -> entitlement = REFUND_PENDING
 -> grace period / evidence processing
 -> REVOKED
```

Для partial refund нужен отдельный механизм.

Нельзя предполагать, что любой refund автоматически означает мгновенный revoke всех технических прав без формализованной политики.

---

# 25. Dispute system

Текущих `OPEN / RESOLVED` недостаточно.

Предлагается:

```text
OPEN
WAITING_BUYER
WAITING_SELLER
UNDER_REVIEW
RESOLVED_BUYER
RESOLVED_SELLER
PARTIAL_REFUND
CLOSED
```

И:

```text
dispute_messages
dispute_attachments
dispute_events
```

Evidence:

```text
MTA version
OS
resource version
logs
screenshots
installation ID
server diagnostics
```

Системная evidence model нужна для того, чтобы support/moderation могли принимать решения на основе фактов.

---

# 26. Reviews and fraud prevention

Хорошее решение уже есть: отзыв связан с purchase и ограничивается одним отзывом на покупку. Дополнительный аудит также считает ownership checks и price snapshot правильными решениями. fileciteturn0file0L146-L153

Но review eligibility должен опираться на:

```text
purchase settled
```

а не просто:

```text
purchase exists
```

Отдельно отслеживать:

```text
self-purchase
multiple accounts
refund after review
chargeback after review
```

---

# 27. Authentication / OAuth

Discord OAuth остаётся хорошим способом identity для этой аудитории.

Но security flow должен быть:

```text
Discord OAuth
 -> server verifies callback
 -> creates session
 -> sets refresh cookie
 -> redirects without tokens
```

Никогда:

```text
redirect?access_token=...
```

---

# 28. CORS / Helmet / proxy

Дополнительный аудит отмечает отсутствие:

- `helmet`;
- явной CORS policy;
- `trust proxy`. fileciteturn0file0L110-L114

## Требование

В production:

```text
helmet/security headers
explicit CORS allowlist
correct trust proxy config
secure cookie config
```

Но `trust proxy` нельзя выставлять слепо в `true` — конфигурация должна соответствовать реальной nginx/load-balancer topology.

---

# 29. Rate limiting

Текущая идея хорошая, но дополнительный аудит отмечает:

```text
Redis unavailable
 -> catch
 -> next()
```

то есть fail-open. Также ключ только по IP, что плохо для операций, где атаку можно проводить через разные IP. fileciteturn0file0L113-L114

## Для чувствительных операций

Использовать комбинированные keys:

```text
IP
+
account/user ID
+
installation ID
```

Для auth:

```text
login identity + IP
```

Для DRM:

```text
license + installation + IP
```

Fail-open допустим только как осознанный availability choice и должен сопровождаться alerting.

---

# 30. Admin statistics

`User.all()`, `Resource.all()`, `Purchase.all()`, `Review.all()` с загрузкой целых таблиц в память — плохой паттерн для production. Дополнительный аудит отмечает это как performance/DoS risk. fileciteturn0file0L122-L124

### Решение

Использовать DB-side aggregates:

```sql
COUNT(*)
SUM()
AVG()
GROUP BY
```

и materialized metrics, если потребуется.

---

# 31. Pagination bug

Дополнительный аудит отмечает неправильный расчёт:

```ts
const resources = ...limit(limitNum).offset(skip).all();
const total = resources.length;
```

Это возвращает размер текущей страницы, а не всего набора. fileciteturn0file0L125-L131

### Решение

Выполнять:

```text
SELECT ... LIMIT/OFFSET
SELECT COUNT(*) ...
```

или использовать transaction/parallel queries.

Ответ:

```json
{
  "items": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 137,
    "pages": 7
  }
}
```

---

# 32. Search

PostgreSQL FTS + GIN — правильный ранний выбор.

Не добавлять Elasticsearch/Meilisearch до реальной потребности.

Для marketplace полезны поля:

```text
title
short_description
description
tags
category
compatibility
```

Позже можно добавить:

```text
seller reputation
verified score
health score
```

как ranking signals.

---

# 33. 3D Asset Studio

Функциональность перспективная, но не MVP.

Отдельный asset service должен извлекать:

```text
polycount
textures
texture sizes
materials
bones
animations
collision
LOD
```

Эти данные можно использовать одновременно в:

```text
3D preview
moderation
product card
compatibility
quality score
```

---

# 34. Live Demo

Live Demo — одна из сильнейших продуктовых фич, но дорогостоящая.

Целевая архитектура:

```text
Demo Manager
 -> template server
 -> ephemeral MTA instance
 -> resource
 -> TTL
 -> cleanup
```

Sandbox policy должна быть не менее строгой, чем у moderation runners.

MVP: не делать.

---

# 35. Leak Radar

Не строить как «crawler нашёл пользователя — пользователь виноват».

Нужна forensic scoring system:

```text
artifact fingerprint
+
watermark
+
installation history
+
source similarity
+
external evidence
=
confidence score
```

Пример:

```text
Leak confidence: 98%
Likely artifact family: resource X
Evidence: 4 signals
```

Не утверждать обвинение при низкой confidence.

---

# 36. Telemetry и privacy

Telemetry должна быть разделена на:

```text
raw private telemetry
        |
        v
aggregated metrics
        |
        v
public health/compatibility stats
```

Нельзя выводить публично:

```text
IP
exact host information
raw HW identifiers
secrets
```

Для backend должны быть определены:

```text
retention period
purpose
access controls
anonymization
```

Если система работает с персональными данными, юридические требования надо согласовать с фактической моделью бизнеса и юрисдикцией.

---

# 37. Observability

До production нужны:

## Metrics

```text
request latency
5xx
DB latency
Redis latency
webhook backlog
payment success rate
license verify failures
download failures
sandbox failures
queue depth
```

## Logs

Structured JSON logs:

```text
request_id
user_id
route
status
latency
error_code
```

## Alerts

```text
payment webhook lag
payment/ledger mismatch
5xx spike
DRM verification spike
storage failures
queue backlog
```

---

# 38. Request ID / tracing

Каждый request получает:

```text
request_id
```

который проходит через:

```text
nginx
 -> API
 -> DB logs
 -> worker
 -> payment event
 -> email
```

Это резко уменьшит стоимость расследования проблем.

---

# 39. Email / background jobs

Email не должен быть частью synchronous transaction.

Плохо:

```text
purchase request
 -> database
 -> SMTP
 -> response
```

Хорошо:

```text
purchase event
 -> queue
 -> mail worker
```

То же для:

```text
analytics
webhook enrichment
reconciliation
license cleanup
telemetry aggregation
```

---

# 40. CI/CD

Текущий CI — хороший фундамент.

До production добавить gates:

```text
lint
unit
integration
E2E
migration test
build
dependency audit
secret scan
container scan
artifact scan
```

Pipeline:

```text
push
 -> CI
 -> image build
 -> staging
 -> migrations check
 -> smoke
 -> approval
 -> production
```

Не делать:

```text
git push main
 -> production
```

---

# 41. Database migrations

Prisma schema должна быть source of truth для структуры БД.

Документация не должна вручную содержать вторую «истинную» schema model, иначе она будет расходиться с кодом.

Правило:

```text
Prisma schema
= canonical DB contract
```

Документация — производная.

---

# 42. API contract

Сделать:

```text
OpenAPI
```

source of truth для публичного HTTP API.

С него можно генерировать frontend types и contract tests.

Отдельно документировать machine-to-machine DRM protocol.

---

# 43. Четыре источника истины

Предлагаемая модель:

```text
DB:
Prisma schema

HTTP API:
OpenAPI

Events:
JSON/TypeScript schemas

DRM:
versioned protocol spec
```

Всё остальное — explanatory documentation.

---

# 44. Документация: что исправить

Текущая проблема — несколько поколений архитектуры.

Примеры:

```text
old:
HWID = MAC + CPU + volume

new:
installation keypair
```

```text
old:
heartbeat 30 min / grace 48h

new:
lease-based model
```

```text
old:
subscription in core model

new:
subscription postponed from MVP
```

Именно такие расхождения нужно удалить, а не оставлять рядом как альтернативы.

Дополнительный аудит также фиксирует расхождение CHANGELOG с фактическими security features. fileciteturn0file0L135-L140

---

# 45. Документы, которые должны существовать

Рекомендуемый набор:

```text
README.md
STATUS.md
ARCHITECTURE.md
DOMAIN_MODEL.md
API.md
THREAT_MODEL.md
DRM_PROTOCOL.md
PAYMENTS.md
FINANCIAL_LEDGER.md
MODERATION.md
ARTIFACT_PIPELINE.md
COMPATIBILITY.md
DEPLOYMENT.md
DISASTER_RECOVERY.md
PRIVACY.md
SELLER_POLICY.md
REFUND_DISPUTE_POLICY.md
ROADMAP.md
```

---

# 46. Threat model

Отдельно описать attacker classes:

```text
A. malicious buyer
B. malicious seller
C. stolen seller account
D. stolen license credential
E. replay attacker
F. reverse engineer
G. leaked artifact
H. compromised admin
I. compromised DB
J. compromised storage
K. compromised CI
L. malicious uploaded archive
```

Для каждого:

```text
asset
threat
likelihood
impact
mitigation
residual risk
```

---

# 47. Admin security

Чувствительные actions:

```text
revoke license
refund
approve seller
publish resource
suspend seller
change role
payout override
```

должны попадать в immutable audit log.

Пример:

```text
actor
action
target
before
after
IP
user-agent
request_id
timestamp
```

Для особо опасных действий позже можно добавить step-up auth/2FA.

---

# 48. First admin bootstrap

Дополнительный аудит отмечает отсутствие нормального seed/CLI flow для создания первого ADMIN. fileciteturn0file0L137-L140

Сделать отдельный command:

```text
pnpm admin:create
```

или:

```text
pnpm db:bootstrap-admin
```

Команда должна работать только явно и не создавать администратора автоматически при каждом старте.

---

# 49. Cleanup / code quality

Убрать:

```text
TODO, которые маскируют production behavior
unused requireRole()
dead DRM helpers
legacy ID conversions
```

Например, отдельный аудит отмечает `requireRole()` как вероятно неиспользуемый и avatar upload как незавершённый. fileciteturn0file0L137-L140

Перед production:

```text
zero critical TODOs
zero known security TODOs
zero dead authentication code
```

---

# 50. Product positioning

Не позиционировать продукт прежде всего как:

> DRM marketplace.

Лучше:

> **Verified and licensed marketplace for MTA:SA resources.**

Основная ценность:

```text
Find
 -> Verify
 -> Buy
 -> Install
 -> Update
 -> Rollback
```

DRM — инфраструктурный механизм доверия.

---

# 51. Конкурентное преимущество

Самый сильный moat здесь не шифрование.

Он может выглядеть так:

```text
Verified resources
+
Compatibility data
+
Seller reputation
+
Installation health
+
Version history
+
Signed artifacts
+
Safe payments
+
Dispute evidence
```

Конкурент может сделать сайт для загрузки ZIP.

Сложнее скопировать сеть проверок и накопленные trust/compatibility data.

---

# 52. Recommended MVP

## Оставить в MVP

```text
Discord OAuth
User / Seller
Catalog
Search
Resource upload
Basic static validation
Moderation
Resource versions
One-time purchases
Real YooKassa integration
Purchase entitlement
Basic license server
Installation key
Signed artifact metadata
Authenticated downloads
Basic MTA Guard integration
Version updates
Rollback
Reviews
Admin panel
Audit log
```

## Не включать в MVP

```text
Subscriptions
Bundles
Chat
A/B testing
Influencer system
Seller API
Live Demo
3D Studio
Leak Radar
Deposits
Advanced analytics
Marketplace for custom development / bidding
```

Последнее важно: freelance marketplace и asset marketplace — разные продукты.

---

# 53. P0 backlog

| ID | Task | Причина |
|---|---|---|
| P0-01 | Удалить/изолировать `/purchases/:id/complete` | Обход оплаты |
| P0-02 | Исправить YooKassa webhook | Реальные платежи сейчас небезопасны/неподтверждаемы |
| P0-03 | Перевести все IDs на `string` | CUID/`parseInt` conflict |
| P0-04 | Закрыть DRM activation | Неавторизованная активация |
| P0-05 | Запретить seller `PUBLISHED` | Обход moderation |
| P0-06 | Убрать tokens из OAuth URL | Token leakage |
| P0-07 | HttpOnly refresh cookie | Session security |
| P0-08 | Hash refresh tokens | DB compromise impact |
| P0-09 | Disable JWT fallback secret | Auth bypass |
| P0-10 | Private storage + signed download | DRM/download bypass |
| P0-11 | Исправить `GetObjectCommand` | Broken download primitive |
| P0-12 | Реализовать ledger invariants | Реальные деньги |
| P0-13 | Webhook event deduplication | Idempotency |
| P0-14 | Sandbox upload runner | Arbitrary code execution |
| P0-15 | Artifact immutable/signature | Supply-chain security |
| P0-16 | Обновить README status | Trust / correctness |
| P0-17 | Удалить stale architecture docs | Prevent implementation drift |

---

# 54. P1 backlog

```text
OpenAPI
RBAC permissions
moderation events
payment reconciliation
refund state machine
dispute evidence
compatibility manifest
dependency graph
update channels
release manifest
rollback health check
structured logging
request IDs
metrics
alerts
email queues
admin bootstrap
security headers
CORS policy
proxy configuration
container scanning
secret scanning
backup restore tests
```

---

# 55. P2 backlog

```text
Live Demo
3D Asset Studio
Leak Radar
advanced seller analytics
subscriptions
bundles
promocodes
influencer links
seller API
A/B testing
```

---

# 56. Целевая архитектура

```text
                       ┌───────────────────────┐
                       │        Web App        │
                       │      Next.js 15       │
                       └───────────┬───────────┘
                                   │
                                   v
                       ┌───────────────────────┐
                       │       API Server      │
                       │    Modular Monolith   │
                       └───────────┬───────────┘
                                   │
             ┌─────────────────────┼─────────────────────┐
             v                     v                     v
        PostgreSQL               Redis               S3/R2
             │
             ├── users
             ├── seller_profiles
             ├── resources
             ├── resource_versions
             ├── artifacts
             ├── orders
             ├── payments
             ├── purchases
             ├── licenses
             ├── installations
             ├── ledger_accounts
             ├── ledger_transactions
             ├── ledger_entries
             ├── payouts
             ├── refunds
             ├── disputes
             ├── reviews
             ├── moderation_events
             └── audit_events

                       ┌───────────────────────┐
                       │    License Service    │
                       │ logical boundary/API │
                       └───────────┬───────────┘
                                   │
                            signed lease
                                   │
                                   v
                       ┌───────────────────────┐
                       │    MTA Guard Module   │
                       │         C++           │
                       └───────────┬───────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    v              v              v
                Resource A     Resource B     Resource C
```

---

# 57. Modular monolith structure

Backend постепенно привести к:

```text
src/
  modules/
    auth/
    users/
    sellers/
    catalog/
    resources/
    moderation/
    orders/
    payments/
    ledger/
    purchases/
    licenses/
    drm/
    reviews/
    disputes/
    admin/
  infrastructure/
    postgres/
    redis/
    storage/
    yookassa/
    email/
    queue/
  shared/
    errors/
    logging/
    validation/
    crypto/
```

Routes должны быть thin:

```text
HTTP
 -> schema validation
 -> application service
 -> transaction
 -> response
```

Не:

```text
route
 -> 200 lines business logic
 -> Prisma
 -> payment provider
```

---

# 58. State machines

## Payment

```text
CREATED
 -> PENDING
 -> WAITING_FOR_CAPTURE
 -> SUCCEEDED
 -> CANCELED
```

Отдельно:

```text
REFUND_PENDING
REFUNDED
```

## License

```text
PENDING
 -> ACTIVE
 -> EXPIRED
 -> REVOKED
```

## Resource

```text
DRAFT
 -> SUBMITTED
 -> UNDER_REVIEW
 -> APPROVED
 -> PUBLISHED
```

Exception:

```text
REJECTED
SUSPENDED
YANKED
ARCHIVED
```

Переходы должны быть централизованными функциями/службами, а не произвольным присваиванием string status.

---

# 59. Data invariants

В проекте должны существовать automated tests для следующих правил.

### Payment

```text
provider payment succeeded
=> one purchase
```

### Webhook

```text
same provider event x10
=> one financial effect
```

### License

```text
revoked license
=> no new valid lease
```

### Download

```text
no entitlement
=> no artifact URL
```

### Review

```text
not settled purchase
=> no verified review
```

### Ledger

```text
total debits == total credits
```

### Artifact

```text
artifact bytes changed
=> hash/signature mismatch
```

---

# 60. Database additions

Минимальный будущий набор:

```text
seller_profiles
license_installations
payment_provider_events
moderation_events
audit_events
ledger_accounts
ledger_transactions
ledger_entries
dispute_messages
dispute_events
dispute_attachments
resource_dependencies
resource_artifacts
release_manifests
compatibility_reports
sessions
```

Не нужно создавать все таблицы завтра. Но доменная модель должна быть зафиксирована.

---

# 61. Storage model

```text
private bucket
  /resources/{resource_id}/{version_id}/artifact
  /resources/{resource_id}/{version_id}/manifest
  /previews/...
  /avatars/...
```

Public web assets:

```text
CDN/cacheable
```

Paid artifacts:

```text
private
short-lived signed URL
```

---

# 62. Backup / disaster recovery

Нужны явно заданные:

```text
RPO
RTO
backup retention
restore test frequency
offsite backup
encrypted backup
key recovery
```

Хранить и восстанавливать нужно не только Postgres:

```text
Postgres
object storage
configuration
critical signing key metadata / recovery procedure
```

---

# 63. Secrets / key management

Production secrets:

```text
JWT signing
DRM signing
YooKassa
OAuth
storage
DB
Redis
email
```

не должны использовать default values.

Для signing keys нужна rotation strategy:

```text
key_id
active key
previous key
revoked key
```

Artifact/lease signature должен содержать `key_id`.

---

# 64. Product roadmap

## Phase 0 — Security & correctness

Сначала закрыть P0.

## Phase 1 — Marketplace core

```text
Auth
Catalog
Seller
Upload
Moderation
Product detail
```

## Phase 2 — Payments

```text
Order
YooKassa
Webhook
Ledger
Purchase
Refund
Reconciliation
```

## Phase 3 — Licensing

```text
Installation
Keypair
Lease
Artifact verification
MTA Guard
```

## Phase 4 — Production operations

```text
CI/CD
Observability
Backups
Security hardening
Rollback
```

## Phase 5 — Closed beta

```text
5–10 sellers
20–50 buyers
real resources
real MTA servers
real support/dispute flows
```

## Phase 6 — Growth features

```text
Live Demo
3D Studio
Leak Radar
Advanced analytics
```

---

# 65. Definition of Production Ready

Проект можно называть production-ready только когда:

```text
[ ] payment bypass impossible
[ ] provider webhook verified correctly
[ ] webhook idempotent
[ ] payment amount validated
[ ] reconciliation implemented
[ ] ledger balances reconcile
[ ] all CUID IDs treated as strings
[ ] seller cannot publish directly
[ ] upload sandbox exists
[ ] artifact is immutable
[ ] artifact hash/signature exists
[ ] downloads require entitlement
[ ] storage is private
[ ] access/refresh tokens are separated
[ ] refresh token is protected
[ ] default JWT secret removed
[ ] security headers configured
[ ] CORS configured
[ ] proxy/IP handling verified
[ ] rate limits tested
[ ] moderation audit works
[ ] admin bootstrap works
[ ] database migrations tested
[ ] backup restore tested
[ ] staging smoke tests pass
[ ] observability exists
[ ] critical TODOs are zero
[ ] documentation matches implementation
```

---

# 66. Definition of MVP

MVP не должен доказывать все идеи проекта.

Он должен доказать одну цепочку:

```text
seller
 -> publishes verified resource

buyer
 -> finds compatible resource
 -> pays
 -> receives license
 -> installs on MTA server
 -> resource runs
 -> update works
 -> rollback works
```

Если эта цепочка не является железобетонной, дополнительные фичи не увеличивают качество продукта.

---

# 67. Что оставить как сильные решения

Следующие решения уже выглядят правильными и их не стоит ломать без причины:

- modular monolith;
- PostgreSQL;
- PostgreSQL FTS + GIN для раннего поиска;
- money as integer cents;
- purchase price snapshot;
- review tied to purchase;
- installation keypair вместо primitive HWID;
- отдельный MTA native module;
- реальные integration tests для MTA Guard;
- queue-oriented future architecture;
- сокращённый MVP scope.

Предоставленный аудит также отдельно отмечает ownership checks, случайные имена файлов, upload size limit, rate limiting, price snapshot и отсутствие секретов в репозитории как хорошие решения. fileciteturn0file0L146-L154

---

# 68. Что не делать

Не делать сейчас:

```text
microservices
Elastic/Meilisearch
subscriptions
chat
A/B platform
influencer platform
full Leak Radar
full 3D Studio
complex bidding
```

Не пытаться решить:

```text
absolute DRM
```

Не выдавать:

```text
production-ready
```

пока не выполнен production checklist.

---

# 69. Главный продуктовый принцип

MTA Market должен продавать не ZIP-файл.

Он должен продавать:

```text
Verified software
+
Compatibility
+
License
+
Updates
+
Support
+
Safe transaction
```

Именно это создаёт ценность, которую сложно заменить бесплатным Telegram/форумом.

---

# 70. Финальный вердикт

Проект перспективный и заметно серьёзнее обычного раннего marketplace. Особенно сильны сочетание MTA-specific tooling, license infrastructure, compatibility verification и marketplace trust model.

Но сейчас основная проблема — не отсутствие фич, а **расхождение между целевой архитектурой и фактическим кодом** плюс несколько P0 security/business bugs.

В первую очередь необходимо сделать:

```text
1. payment cannot be bypassed
2. YooKassa webhook works according to real protocol
3. IDs are consistently CUID/string
4. DRM activation is cryptographic and owned
5. seller cannot bypass moderation
6. downloads are access-controlled
7. auth tokens are redesigned safely
8. uploads run in isolation
9. ledger is actually correct
10. docs / README / code describe one system
```

После этого можно считать платформу готовой к закрытой beta.

---

# 71. Внешние подтверждения по ключевым местам

ЮKassa документирует webhook flow через HTTP Basic Auth и рекомендует проверять актуальный статус объекта и/или IP-адрес отправителя; для webhook требуется HTTPS, а при не-200 доставка продолжается. citeturn164063search4

Для marketplace ЮKassa предлагает Split Payments, где платформа задаёт распределение средств и комиссию между магазинами продавцов. citeturn164063search0turn164063search1

ЮKassa также отдельно описывает Safe Deal, где деньги замораживаются до подтверждения/отмены сделки, а также требования к подготовке площадки. citeturn164063search5turn164063search2

Для Split Payments ЮKassa предусматривает ежедневные реестры успешных платежей и возвратов для финансовой сверки. citeturn164063search3

---

# 72. Один главный roadmap

```text
NOW
 |
 +--> P0 Security / payment correctness
 |
 +--> P0 Documentation consolidation
 |
 +--> P0 Artifact + download security
 |
 +--> P0 DRM protocol
 |
 +--> P1 Ledger / reconciliation
 |
 +--> P1 Moderation / sandbox
 |
 +--> P1 Observability / deployment
 |
 +--> Closed Beta
 |
 +--> Live Demo
 |
 +--> 3D Studio
 |
 +--> Leak Radar
 |
 +--> Growth features
```

**Цель первого production release:** не «сделать весь задуманный MTA Market», а доказать, что один цифровой ресурс можно безопасно проверить, продать, лицензировать, установить и обновить end-to-end.
