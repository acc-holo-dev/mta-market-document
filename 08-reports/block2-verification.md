# Block 2 Verification Report — PLAN.md Tasks A-008..A-012, B-001, B-002

**Дата:** 2026-09-08
**Статус:** VERIFIED (с оговорками, см. § Ограничения)
**Репозиторий:** mta-market-site (база: коммит 08a1843 от Block 1)
**Evidence:** 162/162 тестов (13 файлов), typecheck EXIT 0, build EXIT 0

---

## 1. Статусы задач

| Task | Что сделано | Статус | Evidence |
|---|---|---|---|
| A-008 Seller moderation authorization | Матрица переходов `lib/moderation.ts` (single source of truth): продавец — только DRAFT→PENDING_REVIEW (submit) и PENDING_REVIEW→DRAFT (withdraw); publish/suspend/unsuspend/unpublish — только модератор. Admin-эндпоинт валидирует значение статуса и переход (J-001 state machine). Закрыта дыра: продавец мог «unsuspend» через SUSPENDED→DRAFT и «unpublish» через PUBLISHED→DRAFT | VERIFIED | `tests/moderation.test.ts` 15/15: все seller-переходы (2 разрешённых, 5 запрещённых), все moderator-переходы, garbage-статус 400, не-владелец 403, аноним 401, продавец у admin-эндпоинта 403 |
| A-009 Artifact download authorization | `upload.ts` хранит OBJECT KEY (S3) / локальную ссылку (не публичный URL) для платных артефактов; download-роут: S3 → short-lived signed GetObject (TTL cap 15 мин), local → файл стримится через авторизованный эндпоинт с confinement'ом (`resolveLocalUploadPath` — только bare filename, traversal/absolute отвергаются); публичный static `/uploads` отсутствует в `app.ts` | VERIFIED | `tests/download-auth.test.ts` 9/9: аноним 401, не-покупатель 403, покупатель 200 (контент совпадает), PENDING 403, REFUNDED 403, чужая версия 403; unit: traversal/absolute → null |
| A-010 Provider webhook hardening | КРИТИЧНО: при `YOOKASSA_ENABLED=false` webhook был полностью открытым (ни IP, ни auth, ни re-fetch) — bypass покупки. Теперь: 503 когда провайдер не сконфигурирован; IP-allowlist через `req.ip` (trust proxy), а не спуфабельный первый XFF; timing-safe Basic Auth; дедуп событий; компенсация лицензии при повторной обработке | VERIFIED | `tests/payments-webhook.test.ts` 11/11: не-allowlist IP 403, forged auth 401, replay ×10 → один бизнес-эффект, не-succeeded 409, не-succeeded event type → ack без эффекта, unknown purchase 404, missing metadata 400; `app-security.test.ts`: disabled → 503 |
| A-011 Payment amount invariant | Провайдеру выставляется `finalPrice` (не priceSnapshot — перезаряд при скидке); webhook верифицирует amount/currency против **finalPrice**; проверка привязки provider reference к покупке (Payment.purchaseId) с quarantine; mismatch → событие FAILED + 409, entitlement не выдаётся | VERIFIED | Тесты: wrong amount → 409 + PENDING + нет лицензии; wrong currency 409; payment bound to другой purchase → 409 quarantine |
| A-012 Secret startup policy | Production-требования расширены: `DRM_SERVER_PRIVATE_KEY` и `ARTIFACT_SIGNING_PRIVATE_KEY` обязательны (fail closed на старте, не на первом использовании); публичные fallback-секреты удалены из docker-compose; DRM/artifact ключи добавлены в compose и .env.example | VERIFIED | `tests/startup-policy.test.ts` 7/7: unit (5 негативов + позитив) + acceptance — реальный child process с отсутствующим JWT_SECRET завершается non-zero |
| B-001 Sandbox integration | Sandbox возвращён из attic, переписан на контрактный ORM (SandboxRun); `static.ts` переведён с `Extract({path})` (реальная запись на диск!) на `Parse` (in-memory, autodrain) — «no host filesystem» соблюдён; pipeline: version upload → static validation → (sandbox при Docker) → подпись; FAILED-валидация откатывает версию (422); publish gate блокирует версии с FAILED-валидацией | VERIFIED | `tests/publication-pipeline.test.ts` 5/5: malicious fixture (zip-slip) → 422 + версия не создана; good zip → 201 + SandboxRun записан; external URL → 400; publish gate 409 на FAILED |
| B-002 Artifact signing integration | `signing.ts` восстановлен и переписан на контрактный ORM; единая library implementation (`lib/artifact/signing.ts`) используется web-pipeline; платформенный ключ `ARTIFACT_SIGNING_PRIVATE_KEY` (private только в env, PublisherKey хранит public); publish gate требует валидную подпись всех версий | VERIFIED | Тесты: версия создаётся подписанной (artifactHash в ответе, ArtifactSignature в БД); unsigned версия → 409 «Version is not signed»; после подписи library → 200 |

## 2. Попутно исправленные дефекты (вскрыты тестами)

1. **Ledger invariant был сломан для скидок**: `platformFee + sellerRevenue !== priceSnapshot` — при скидке сумма равна finalPrice, поэтому ВСЕ скидочные покупки падали с 500 на webhook и никогда не завершались. Инвариант переведён на finalPrice (согласовано с A-011).
2. **zod-enum типов ресурса не совпадал с контрактом** (`SCRIPTS/MAPS/...` vs `SCRIPT/MAP/...`) — валидные по API запросы падали на DB CHECK constraint.
3. **`static.ts` реально распаковывал архивы на диск** в фиксированный путь `/tmp/sandbox-extract` под видом «статического анализа» — заменено на `Parse` (in-memory).
4. **`upload.ts`**: checksum читался после удаления файла (краш в S3-ветке); fileUrl для ресурсов был публичным URL.
5. **ORM-нюанс**: `where().delete()` удаляет одну строку — учтено в cleanup-утилитах.

## 3. Инфраструктура тестов

- `tests/helpers/db-reset.ts` — FK-safe сброс всех тестовых сущностей (префикс ID `550e8400-...44`), устойчив к падениям предыдущих прогонов.
- `tests/helpers/zip.ts` — минимальный in-memory ZIP-билдер (stored + CRC32) для fixtures sandbox-валидации.
- `vitest.config.ts`: `fileParallelism: false` — сьюты делят одну БД, параллельный запуск разрушал фикстуры.
- Мок провайдера (`vi.mock('../src/lib/yookassa')`) для webhook-тестов без внешних вызовов.

## 4. Ограничения (честно)

- **Sandbox execution (Docker) недоступен локально** — выполняется только static-валидация; sandbox-этап помечается PENDING (manual review) и не блокирует публикацию. Требование «malicious fixture cannot access host secret» покрыто static-валидацией + in-memory Parse; полный Docker-sandbox — при наличии Docker-среды (staging).
- **S3-ветки не тестируются вживую** (нет S3/R2): signed URL логика покрыта код-ревью и cap'ом TTL; контрактные тесты S3 — на staging.
- **CLI artifact:sign остаётся в attic** — единая library implementation существует (`lib/artifact/signing.ts`), CLI будет восстановлен в фазе артефактов (I) поверх той же библиотеки.
- Аудит-трейл FAILED-валидаций каскадно удаляется вместе с откатом версии (SandboxRun привязан к versionId) — evidence уходит в 422-ответ загрузчику; персистентный журнал неудачных загрузок — кандидат для Phase J (J-003).
- Browser E2E по-прежнему на staging (нет Discord-кредов).

## 5. Команды воспроизведения

```text
pnpm install
pnpm db:emit
pnpm --filter @mta-market/server type-check   # EXIT 0
pnpm --filter @mta-market/server test         # 162/162, EXIT 0
pnpm run type-check                           # EXIT 0
pnpm run build                                # EXIT 0
```

Тестовая БД: PostgreSQL 16 (embedded, порт 5433), схема по контракту.