# CURRENT — состояние проекта

Обновлено: 2026-09-10 (после выполнения PLAN-004).

## Активный план

Нет. PLAN-004 (Production Readiness & Operational Hardening) выполнен и
зафиксирован (см. [COMPLETED/PLAN-004.md](COMPLETED/PLAN-004.md)) со статусом
**IMPLEMENTATION COMPLETE — PRODUCTION NOT VERIFIED**: код и процедуры готовы,
реальные production-проверки (live webhook, Windows DRM, боевой домен,
restore/rollback drill) требуют deployment-окружения.

## Состояние продукта

MTA Market — полнофункциональный marketplace, подготовленный к production
deployment (см. [PROJECT.md](../PROJECT.md)). После PLAN-004 дополнительно:

- **Media contract**: S3 и локальное хранилище работают по одному контракту
  (opaque `media-` имена → объект в `media/` префиксе → контролируемая
  раздача `/media/<name>`; опциональный 302 на CDN через
  `MEDIA_PUBLIC_BASE_URL`).
- **Migration formal path**: Prisma 8 migration-пакеты существуют и в git
  (baseline 215 ops + media 5 ops), `deploy.sh` применяет `db migrate`
  (formal path) — dev-only `db update` для production запрещён;
  connection limits задокументированы (пул 10/процесс дефолт).
- **Production env matrix**: `.env.example` документирует все переменные с
  разделением dev/staging/prod; startup validation fail-fast (включая
  `DRM_MASTER_KEY`, который раньше проверялся только в runtime).
- **Rate limits**: production defaults зафиксированы в prod-compose
  (AUTH 100/15мин, LOGIN 10/мин/аккаунт и т.д.); dev/E2E значения изолированы.
- **Redis semantics**: security-critical лимитеры (auth/strict/per-account)
  fail-closed при outage Redis (503), bulk-limiter — fail-open (решение M-002
  зафиксировано); Redis содержит только rate-limit счётчики — persistence не
  требуется (M-003).
- **Storage**: uploads volume в Dockerfile chown-нут под non-root (P0-фикс —
  раньше все загрузки в проде падали EACCES); `/uploads/`-локация в nginx
  удалена (латентный bypass артефактов).
- **Deploy pipeline**: backup → migrate (`prisma db update`) → deploy exact
  `IMAGE_TAG` → health-gate (`compose up --wait`) → auto-rollback на
  предыдущий тег; предыдущая версия образа сохраняется для отката.
- **CI**: публикация образов гейтится на test+security+lint; audit-gate.sh
  блокирует high/critical (waivers только с датой ревью); 23 уязвимости
  закрыты (next 15.5.24, overrides hono/lodash/postcss/sharp и др.); SHA-теги
  образов `format=long`.
- **Финансовая целостность**: знак кэша баланса при refund исправлен;
  settlement идемпотентен (детерминированный transactionId
  `settle:purchase:<id>`) и дозапускается в alreadyCompleted-ветке;
  `payment.canceled` закрывает PENDING-заказы (FAILED), поздний succeeded
  чинит и завершает (provider truth wins); маппинг canceled в reconciliation
  единый.
- **Надёжность процесса**: глобальный error-middleware, unhandledRejection
  handler, graceful shutdown закрывает HTTP/Redis/DB.
- **nginx**: строгий CSP (`default-src 'self'`, без unsafe-eval), HSTS в
  443-блоке, Permissions-Policy, security headers не теряются в location,
  Connection upgrade через map, server_tokens off.
- **DRM client**: подпись challenge над декодированными байтами (протокольный
  P0), interop-тест в `tests_drm/main.cpp`.
- **Документация**: production runbook, database-migrations, backup-restore
  (RPO 24h/RTO 4h) в `mta-market-site/docs/operations/`.

## Приёмка PLAN-004 (2026-09-10)

- Backend-тесты: **256/256** (было 253/254; `startup-policy` acceptance больше
  не падает — причина была в dotenv-заполнении JWT_SECRET из dev-.env).
- Playwright browser E2E: **25/25** на живых dev-серверах
  (PostgreSQL/Redis в Docker; для запуска Chromium в окружении потребовались
  локальные системные библиотеки — среда, не продукт).
- Prisma 8 migration formal path: `migration plan` (baseline 215 ops +
  media 5 ops) → `db migrate` → `migration status` "Up to date" на живой БД;
  пакеты и snapshots в git.
- `tsc --noEmit` чист у server и web; production build server и web проходит;
  lint чистый.
- Модуль (C++): `make -f source/drm/Makefile test` — ALL TESTS PASSED, включая
  новый challenge interop-тест.
- Security: audit-gate 0 high/critical (23 закрыты обновлениями).
- Статус плана: **IMPLEMENTATION COMPLETE — PRODUCTION NOT VERIFIED**.

## Blockers

Нет блокеров кода. Перед реальным production-запуском нужно окружение для:

1. **YooKassa sandbox→real** (D-007): sandbox-магазин, зарегистрированный
   webhook URL, тестовый платёж → webhook → license; потом отдельный
   real-money прогон. Runbook: `docs/operations/production-runbook.md` §12.
2. **DRM live + Windows** (F-001/F-003): сервер + собранный модуль против
   staging-лицензионного сервера; Windows-сборка (MSVC/MinGW) и DPAPI
   key store ни разу не компилировались.
3. **Restore/rollback drill** (C-005, K-005): прогон на staging-клоне по
   `docs/operations/backup-restore.md`.
4. **Домен/TLS/DNS** (S-001..S-006) и внешний uptime-мониторинг (I-004/I-005)
   — операции на стороне инфраструктуры.

Известные непокрытые темы (кандидаты в следующий план):

- Payout flow (замкнуть «начислено → выведено», SELLER_PAYOUT) — до реальных
  выплат.
- Password reset + смена пароля с инвалидацией сессий (O-001/O-003).
- SQL-агрегация rating/popular вместо in-memory капа 1000 (R-002) — перед
  ростом каталога.
- Шифрование OAuth-токенов в БД (G-01 аудита) или отказ от их хранения.
- Account deletion/anonymization (P-003).
- Reconciliation: ledger-аудит (дубли LedgerEntry), retry/backoff шагов,
  distributed lock при >1 реплики.
- pg_trgm GIN для ILIKE-поиска + индексы под сортировки (R-003/G-14).

## Следующий шаг

Сформировать следующий план отдельным решением (автоматически не создаётся).
Логичные кандидаты: production verification plan (пункты blockers выше),
payout flow, password lifecycle. Наличие темы в списке не является
обязательством её реализовать.
