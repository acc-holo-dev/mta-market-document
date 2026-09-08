# Block 1 Verification Report — PLAN.md Tasks A-001..A-007

**Дата:** 2026-09-08
**Статус:** VERIFIED (с оговорками, см. § Ограничения)
**Репозиторий:** mta-market-site (коммит на момент работ: cd2b260)
**Метод:** DISCOVER → VERIFY → REUSE/INTEGRATE → TEST → DOCUMENT (политика PLAN §18)

---

## 1. Ключевое открытие: рассинхрон поколений контракта

Заявление reality-check (PLAN §5) «CUID/string ID migration уже отражена в актуальной
Prisma contract schema» оказалось верным **только для PSL-исходника**
(`contract.prisma`), но не для скомпилированного рантайм-артефакта:

- `contract.json` / `contract.d.ts` (то, чем реально работает `db.orm`) были
  до-CUID поколения: `User.id` — `int4` (number), autoincrement;
- в рантайм-контракте отсутствовали модели TASK-020 (`Lease`, `ServerSigningKey`,
  `ArtifactSignature`, `PublisherKey`, `SandboxRun`, `Reconciliation*`);
- `contract.prisma` не компилировался: `@default(cuid())` не поддержан диалектом
  «SQL PSL provider v1», плюс висячие backrelations на модели `Order`/`OrderItem`/
  `Discount`, которые никогда не существовали ни в одном компилировавшемся PSL;
- baseline `tsc --noEmit` падал на десятках ошибок number/string по всем роутам —
  репозиторий был закоммичен в некомпилируемом состоянии.

Это живое подтверждение центрального тезиса плана: `implemented != connected != verified`.

**Исправление (evidence: `pnpm db:emit` → ok, `prisma db init` → «Applied 127
operation(s), database signed»):**

- PSL отремонтирован: `cuid()` → `uuid()` (ADR-017), вырезаны висячие связи,
  `Review.resourceId Int` → `String`, `FinancialTransaction.relatedPurchaseId/
  relatedPayoutId Int?` → `String?`, `Payment.orderId` → `purchaseId` (ADR-018),
  `Purchase.orderItemId` → optional;
- контракт переэмитирован; схема применена к БД (`prisma db init`, 127 операций,
  23 таблицы, БД подписана контрактом);
- код приведён к строковым ID (A-004): numeric casts доменных ID в server
  отсутствовали и до работ (все `parseInt` — пагинация/env/IP); исправлены
  типы фронта (`id: number` → `string` в store/dashboard/resources) и
  `validateParam("id","int")` → `validateCuid("id")` в purchases.

## 2. Статусы задач

| Task | Что сделано | Статус | Evidence |
|---|---|---|---|
| A-001 Browser auth E2E | Сервер: убран hack `auth_callback_token` (access-токен больше не в cookie), refresh-cookie через централизованный хелпер; JWT access/refresh различаются claim'ом `type` (refresh не принимается как access и наоборот); фронт: токены только в памяти (zustand, без persist/localStorage), callback обменивает refresh-cookie на access через `POST /auth/refresh`, single-flight refresh (параллельные 401 ждут один промис), retry ровно один раз, `withCredentials` везде, `bootstrapSession` восстанавливает сессию после reload | VERIFIED (integration) | `tests/auth-flow.test.ts` 7/7: refresh без cookie → 401; ротация с новым HttpOnly cookie; replay старого токена → 401 + семья отозвана (включая новый токен); logout удаляет сессию и чистит cookie; `/auth/me` принимает access, отвергает refresh; `tests/jwt.test.ts` 6/6 |
| A-002 Cookie handling | `cookie-parser` добавлен и подключён до роутов; атрибуты cookie централизованы (`lib/cookies.ts`): HttpOnly, Secure/SameSite из topology (A-003) | VERIFIED | тесты шлют реальную cookie через HTTP (`supertest`), проверяют `HttpOnly`/`SameSite` флаги; refresh/logout видят cookie |
| A-003 CORS / origin model | Topology зафиксирована: production — same-origin за nginx (`/api/` → backend, см. nginx.conf); dev — cross-origin. `cors` middleware с явным allowlist (`CORS_ORIGINS`), `credentials: true`, wildcard с credentials запрещён; cookie `SameSite` конфигурируется (`COOKIE_SAMESITE`), `SameSite=none` форсирует Secure | VERIFIED (preflight) | `tests/app-security.test.ts`: preflight от allowlist-origin → ACAO=origin + credentials=true; не-allowlist origin не отражается; `*` никогда не возвращается. Browser E2E с реальным Discord — см. § Ограничения |
| A-004 ID audit | Numeric casts доменных ID в server отсутствуют (grep-аудит: все `parseInt` — пагинация/env/IP); `validateParam("id","int")` заменён на `validateCuid`; фронтовые типы исправлены на string; контракт теперь реально строковый | VERIFIED | typecheck (контракт text PK), grep-аудит чист, тесты используют UUID-идентификаторы |
| A-005 Production simulate paths | `/payments/:id/simulate` регистрируется только вне production (условная регистрация) — подтверждено; `/purchases/:id/complete` удалён ранее — подтверждено (комментарий в коде + отсутствие роута); добавлен явный JSON 404-обработчик | VERIFIED | `tests/app-security.test.ts`: production-like env → 404 на simulate; dev/test → 401 (роут существует, auth требуется) |
| A-006 DRM v2 wiring | v2 смонтирован в `app.ts` (`/drm/v2/*`); сервис переписан с несуществующего classic-Prisma API на контрактный ORM; добавлен ownership: регистрация installation требует authenticate + владение лицензией (license→purchase.buyerId), installation привязывается к лицензии, activate требует совпадения licenseId с привязанным (INV-007/INV-011); rate limits; карта ошибок | VERIFIED | `tests/drm-v2.test.ts` 9/9: полный цикл register→verify(challenge)→activate(signed lease)→heartbeat→lease lookup + негативы: неаутентифицированная регистрация 401, чужая лицензия 403 `DRM_LICENSE_NOT_OWNED`, mismatch лицензии 403, nonce reuse 409, активация неверифицированной установки 403, неверная подпись challenge 401 |
| A-007 DRM v1 decision | Выбран вариант A: v1 activation (`/drm/activate`, `/drm/verify`) → `410 Gone`; management (`my-licenses`, `revoke`) оставлен; ADR-016 записан | VERIFIED | `tests/drm-v2.test.ts`: 410 + `status: deprecated` на обоих эндпоинтах; ADR-016 в mta-market-document |

## 3. Попутно исправленные дефекты (вскрыты тестами/типизацией)

1. **Крипто DRM v2 был структурно сломан** (легасные тесты `drm-crypto.test.ts`,
   никогда не запускавшиеся, упали): challenge/lease подписи шли через
   artifact-manifest machinery с невыполнимым self-hash условием. Переписано на
   прямое Ed25519: challenge — над сырыми байтами challenge; lease — над
   каноническим JSON payload (serverKeyId включён в подпись). 16/16 тестов.
2. **`verifyArtifactSignature`**: bogus-проверка «manifest hash» путала хэш
   манифеста с хэшем артефакта — sign/verify не сходились. Исправлено
   (artifact-crypto 20/20).
3. **Reuse detection не отзывал семью**: контрактный ORM `where().delete()`
   удаляет только ОДНУ строку — при replay-атаке ротированный токен атакующего
   оставался валидным. Семья теперь дренируется циклом. (Найдено интеграционным
   тестом, не ревью.)
4. **Refresh-токены детерминированы** в пределах секунды (одинаковый payload →
   одинаковый JWT → коллизия hash). Добавлен `jti` в refresh-токены.
5. **Дублирующие/мёртвые реализации** перенесены в `src/attic/` с README
   (не удалены): `lib/order.ts` (Phase C), `lib/artifact/signing.ts` + CLI
   artifact (Phase B), `lib/providers/*` (дубль E-002), `lib/sandbox/` (Phase B),
   их тесты. Канонические реализации не тронуты.
6. **Reconciliation** (B-003 зона) переписан на контрактный ORM, чтобы
   компилировался; поведение сохранено.
7. Rate limit thresholds конфигурируемы через env (Q-002 уточнит размерности).

## 4. Ограничения (честно)

- **Browser E2E с реальным Discord OAuth не выполнялся**: нет Discord-приложения/
  кредов локально. Acceptance A-001 покрыт интеграционно (полный cookie-цикл
  через HTTP, включая ротацию/reuse/logout) + typecheck/build фронта. Реальный
  browser E2E — на staging с провайдерскими кредами (N-002).
- **Redis в тестах отсутствует** → rate limiter fail-open (это и есть его
  контрактное поведение при недоступности Redis); лимиты проверяются
  конфигурацией, но не поведением 429 в тестах.
- **Миграция существующей БД** (int4 → text PK) не выполнялась: локальной БД до
  этого блока не существовало; тестовая БД развёрнута с нуля по контракту
  (`prisma db init`). Для существующих окружений нужен `prisma db update`/
  migration plan — в Block 2 перед деплоем.
- `lib/drm/crypto.ts` `signLease` формат подписи изменён (канонический JSON
  вместо `sha256|sha256` через mock-manifest) — модуль-клиент (spike) ещё не
  реализовал верификацию, так что совместимость не нарушена; контракт
  фиксируется в Phase G (G-001).

## 5. Команды воспроизведения

```text
pnpm install
pnpm db:emit                      # контракт из contract.prisma
pnpm --filter @mta-market/server type-check   # EXIT 0
pnpm --filter @mta-market/server test         # 81/81, EXIT 0
pnpm run type-check               # EXIT 0 (server + web)
pnpm run build                    # см. BUILD EXIT в логе
```

Тестовая БД: PostgreSQL 16 (embedded, порт 5433), схема применена
`prisma db init` (127 операций, подписана контрактом).