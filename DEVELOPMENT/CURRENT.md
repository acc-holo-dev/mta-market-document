# CURRENT — состояние проекта

Обновлено: 2026-09-11 (PLAN-006 зарегистрирован; реализация не начата).

## Активный план

[PLAN-006 — Daily Experience Foundation](ACTIVE/PLAN-006.md)
(зарегистрирован 2026-09-11). Переход состояния: из «Home = витрина
маркетплейса» в «Home = живой вход в экосистему» — глобальные LIVE-агрегаты
из реальных heartbeat-сэмплов, derived activity read-layer (без новой
доменной сущности), пересборка Home («Сейчас в MTA» / Активность /
Популярное), сводка «Сейчас / За ночь» в dashboard. Обоснование выбора
фазы — [NEXT-PHASE.md](NEXT-PHASE.md). Реализация ещё не начата.

Предыдущий план: PLAN-005 (Community & Server Foundation) выполнен и
зафиксирован ([COMPLETED/PLAN-005.md](COMPLETED/PLAN-005.md)) со статусом
**IMPLEMENTATION COMPLETE — browser E2E прогнан, production-проверка остаётся
отдельным шагом** (см. Blockers внизу).

## Состояние продукта

MTA Market — marketplace + community + server platform (см. [PROJECT.md](../PROJECT.md)).
После PLAN-005 платформа больше не только про commerce: у неё есть живая
социальная поверхность вокруг сущности SERVER.

### Что появилось в PLAN-005

- **Server domain (A/B/C/D/E)**: модель Server (slug, owner, брендинг,
  connection-данные, lifecycle CREATED → PENDING_VERIFICATION → VERIFIED →
  ACTIVE (+ SUSPENDED/ARCHIVED)), staff-роли, регистрацию через мастер
  /servers/create, ownership verification через possession-токен интеграции
  (первый валидный heartbeat от модуля/скрипта → VERIFIED, audit + note),
  публичную страницу /servers/[slug] (hero с онлайн-статусом, review,
  follower count; табы Обзор/Live/Новости/Обновления/Отзывы/Сообщество),
  monitoring (ONLINE/OFFLINE/UNKNOWN; graceful OFFLINE только от сервера,
  тишина → UNKNOWN через sweep-job, никогда не «врёт» OFFLINE), statistics
  (peak/average/uptime по реальным сэмплам, 24h/7d/30d).
- **Global community (F)**: форум (7 категорий, темы, посты, реакции,
  редактирование, мягкое удаление, состояния OPEN/LOCKED/ARCHIVED,
  просмотры = реальные, pinned), хаб /community (latest/active/pinned/
  activity), server-linked обсуждения (G-004: привязка темы к серверу —
  только персоналом этого сервера).
- **Server news & updates (H/I)**: черновик → публикация (уведомления
  подписчикам), опциональное обсуждение новости (тред), обновления с
  версией/changelog, глобальная лента /news (один News object на все
  поверхности).
- **Reviews с токенами (J/K)**: отзывы сервера доступны ТОЛЬКО после
  claim одноразового токена интеграции (server-bound, expiry 24h по
  умолчанию, replay-protection, self-review ban, 1 отзыв/сервер,
  ✓ Verified Interaction badge = подтверждённое взаимодействие, скрытие
  отзывов модерацией с audit).
- **Follow & notifications (L/M)**: подписка на сервер, счётчик подписчиков,
  уведомления SERVER_NEWS / SERVER_UPDATE / FORUM_REPLY / REVIEW_EVENT /
  MODERATION, центр /notifications с непрочитанными и «прочитать всё»,
  колокольчик с бейджем в навбаре.
- **Dashboard & profile (N/O/P)**: «My MTA» блок в существующем dashboard
  (мои серверы, подписки, обсуждения, уведомления), публичный профиль
  /profile/[username] с бейджами (Server Owner / Verified Server /
  Verified Seller) и публичными серверами/ресурсами.
- **Moderation & reports (Q/R)**: админ-вкладки Серверы (inspect, verify/
  reject, suspend/restore/archive — каждое действие audit + уведомление
  владельцу), Жалобы (queue → resolve/dismiss + уведомление репортёру),
  модерация контента сообщества (news/reviews/threads).
- **Privacy by default (S/T)**: host/port никогда не покидают бэкенд;
  showResources/showStaff/showTechStack/showCommunity выключены по
  умолчанию; showStats=true по умолчанию, но выключение скрывает live-
  числа на API-уровне; связь Server↔Resource публична только по явному
  opt-in владельца (проверено E2E: «public API does not expose the
  resource list»).
- **Integration protocol (AC/§33)**: POST /integration/heartbeat (token
  possession → proof-of-control; только агрегаты: count/status), POST
  /integration/review-tokens (one-time токен для игрока). Модуль
  (mta-market-module): новый `source/drm/market_client.{hpp,cpp}` + Lua
  функции `mta_market_configure/start/stop/report_players/status/
  review_token` (privacy: только счётчики, никаких player identity).
- **Поиск (W)**: /search с явными типами результатов (Resources/Servers/
  Discussions + счётчики).
- **Seed (§34)**: scripts/seed-plan005.ts — 10 реалистичных серверов
  (online/offline/unknown/pending/suspended, приватность-вариации),
  14 пользователей, 7 категорий, темы с ответами и реакциями, новости
  (published+draft), обновления, 15 verified-отзывов, подписки,
  уведомления. Dev-инструмент scripts/dev-heartbeat.ts держит онлайн
  свежим (симулятор интеграции через публичный API).

## Приёмка PLAN-005 (2026-09-10)

- Backend-тесты: **337/337** (было 256; +81 PLAN-005: servers 28,
  community 21, reviews 15, news 16; REGRESSION: auth/commerce/DRM/
  payments не сломаны — payments-webhook требует TRUST_PROXY=true при
  локальном прогоне, в CI это дефолт).
- Prisma 8 migration formal path: `migration plan` (93 additive ops,
  16 новых таблиц + индексы/FK) → `db migrate` на dev-БД → "Applied 1
  migration(s) (93 operation(s))"; пакет
  `migrations/app/20260910T1051_plan005_community_server` в git.
- `tsc --noEmit` чист у server и web; production build проверяется CI.
- Playwright browser E2E: план005-спека прогоняется на живых dev-серверах
  (см. COMPLETED/PLAN-005.md для финального счёта).
- Live-проверка цепочки интеграции (ручная, на dev-серверах): heartbeat с
  токеном → monitoring ONLINE + verification VERIFIED → публичная страница
  показывает 431/800; выпуск review-token через /integration/review-tokens
  → claim через API → eligibility granted.
- Модуль (C++): market_client + Lua-функции добавлены; сборка модуля
  требует Linux-тулчейн (Windows-сборка модуля была и остаётся отдельной
  задачей — см. Blockers PLAN-004).

## Ре-верификация (2026-09-10, аудит документации и кода)

Независимый прогон всей цепочки после аудита трёх репозиториев:

- Backend-тесты: **337/337** на чистой БД (контракт-схема применена
  формальным путём `prisma db update`, 311 additive ops с нуля).
- Playwright browser E2E: **37/37** (plan001 12 + plan003 13 + plan005 12)
  на живых dev-серверах; для запуска Chromium потребовались системные
  библиотеки (libnss3/libnspr4/libasound2) — установка через
  LD_LIBRARY_PATH, без root.
- Модуль: `make -f source/drm/Makefile test` — ALL TESTS PASSED (Linux x64).
- Migration refs фикс: `refs/db.json` в mta-market-site указывал на
  промежуточный хеш (771428b3 после 0605_migration) вместо финального
  состояния после 1051_plan005 (52df04b6) — исправлен (site 6064911).
- Live-интеграционная цепочка воспроизведена: seed-plan005 → dev-heartbeat
  (7 серверов каждые 45с) → публичный API показывает ONLINE 436/800,
  VERIFIED; host/port в публичном payload отсутствуют.
- Документация приведена к целевой структуре (PROJECT/IDEAS/ACTIVE/
  COMPLETED; dedup SURFACE-MAP; PLAN-005 spec+record объединены) —
  коммиты 40561d9, bb9a729.

## Blockers

Нет блокеров кода. Ограничения/что осталось (в рамках PLAN-005 не было
обязательным):

1. **Windows-сборка модуля** (унаследовано из PLAN-004): POSIX-сокеты в
   http_client.cpp + отсутствие OpenSSL-линковки в CMake. market_client
   следует за существующей архитектурой и наследует это ограничение.
2. **Production verification** (как и после PLAN-004): live domain, real
   payments, restore drill.
3. **Почтовые уведомления**: in-app уведомления готовы; email-канал —
   будущий план (нужен digest/анти-спам дизайн).

Известные осознанные ограничения (кандидаты в следующие планы):

- Rate-limit счётчики подписок/новостей используют userRateLimit (in-memory
  Redis) — при росте аудитории пересмотреть.
- Feed /news не имеет сортировок/фильтров по серверу (минимум по плану).
- Поиск — ILIKE по name/description/title (pg_trgm — кандидат из PLAN-004
  blockers, теперь и для servers/threads).
- Нет email-уведомлений; нет push.

## Следующий шаг

Решение по DAILY-EXPERIENCE §50 принято: [NEXT-PHASE.md](NEXT-PHASE.md)
→ [PLAN-006 — Daily Experience Foundation](ACTIVE/PLAN-006.md) зарегистрирован
в [ACTIVE/](ACTIVE/). Работа идёт по workstreams плана (реализация не начата).
Прочие кандидаты (production verification — blockers PLAN-004/005, content
layer, расширенная статистика серверов, events, email/push) и любые темы
в этих списках не являются обязательством их реализовать.
