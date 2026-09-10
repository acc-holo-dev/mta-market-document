# PLAN-005 — Community & Server Foundation

**Статус: IMPLEMENTATION COMPLETE** (2026-09-10)

## Итог

Цель плана достигнута: MTA Market начал жить как COMMUNITY PLATFORM. Введена
главная сущность **SERVER** (hub: identity + monitoring + news + updates +
community + reviews), глобальный форум, подписки, уведомления, верифицированные
отзывы через одноразовые токены интеграции, публичные профили с бейджами,
модерация и репорты — всё поверх privacy-by-default правил, enforced на
backend-authorization layer.

## Что сделано (по workstreams)

### WORKSTREAM A — Server Domain Foundation
- `Server` модель (contract.prisma): id/owner/slug/name/description, брендинг
  (logo/banner/accent), connection (host/port — приватные), region, lifecycle
  (CREATED/PENDING_VERIFICATION/VERIFIED/ACTIVE/SUSPENDED/ARCHIVED),
  verification (PENDING/VERIFIED/FAILED/EXPIRED), monitoring
  (ONLINE/OFFLINE/UNKNOWN), playerCount/maxPlayers/lastSeenAt, 5 privacy-
  флагов (S/T), integrationTokenHash (sha256).
- ServerMember (OWNER/ADMIN/MODERATOR) — owner создаётся автоматически.
- Существующие сущности не дублированы (AB): ServerResource линкуется к
  существующему Resource; Review-системы не сломаны (K: отдельная
  ServerReview с собственной eligibility-политикой, как требует J-001).

### WORKSTREAM B — Registration
- POST /servers (валидация name 3-60, port 1-65535, host-длина; slug через
  translit + уникальность), 5-шаговый мастер /servers/create (Basic →
  Connection → Branding → Visibility → Verification), connection-данные
  помечены «приватно по умолчанию», visibility defaults privacy-first.

### WORKSTREAM C — Ownership verification
- POST /servers/:slug/integration-token (owner-only; plaintext возвращается
  один раз; hash хранится; ротация перезапускает PENDING).
- POST /integration/heartbeat: possession токена = proof-of-control; первый
  валидный heartbeat → VERIFIED (+VERIFIED lifecycle, audit, лог).
- GET /servers/:slug/verification — human-readable состояния (C-003:
  pending/failed/expired/verified + note, без технических ошибок).

### WORKSTREAM D/E — Public page + monitoring
- GET /servers/:slug — privacy-filtered payload (host/port никогда;
  playerCount/maxPlayers/lastSeenAt = null при showStats=false).
- GET /servers (discovery: только VERIFIED/ACTIVE; q-sort по реальному
  playerCount, без fake ranking), GET /servers/:slug/statistics (peak/
  average/uptime по реальным сэмплам ServerStatusSample; downsampled
  series; showStats=false → enabled:false).
- Sweep job (jobs/serverMonitoring.ts): stale heartbeat → UNKNOWN (E-006:
  «система не знает» ≠ «точно выключен»), истёкшие токены → EXPIRED.
- OFFLINE ставится ТОЛЬКО явным state:"OFFLINE" в heartbeat (graceful
  shutdown), проверено тестом.

### WORKSTREAM F/G — Community
- ForumCategory (7 seeded), ForumThread (state OPEN/LOCKED/ARCHIVED, pinned,
  views = реальные инкременты, replyCount/lastPostAt), ForumPost (soft delete,
  editedAt), ForumReaction (уникальность post+user+kind).
- /community хаб (latest/active/pinned/recentActivity), категории с
  threadCount, thread-страница с пагинацией.
- G-004: serverId на треде — только персонал сервера может привязать
  (проверено: чужой сервер → 403).
- G-002/G-003: /community/servers/:slug/threads (показ только при
  showCommunity opt-in), members (aggregate всегда, список — только opt-in).

### WORKSTREAM H/I — News & Updates
- ServerNews: DRAFT → PUBLISHED (publish-эндпоинт, уведомления подписчикам
  SERVER_NEWS, опциональное обсуждение — тред с newsId, H-005: один объект,
  все поверхности читают одни строки).
- ServerUpdate: version+changelog, дубликат версии → 409, уведомления
  SERVER_UPDATE; глобальная лента GET /news (+pagination).

### WORKSTREAM J/K — Reviews
- ServerReviewToken: одноразовый, server-bound, expiry 24h (5м..7д),
  sha256-hash, plaintext один раз; POST /integration/review-tokens — модуль
  получает токен для игрока.
- Claim: /servers/:slug/review-token/claim — классификация unknown (400) vs
  replayed/expired (409) vs active; wrong-server не сжигает токен; self-
  review ban (владелец/персонал не могут claim/review).
- ServerReview: только при ServerReviewEligibility (создаётся при claim),
  1/сервер/пользователь, ✓ Verified Interaction, скрытие модерацией
  (HIDDEN + уведомление + audit), удаление автором сохраняет eligibility.

### WORKSTREAM L/M — Follow & Notifications
- ServerFollow (+unique), follower count публичен, список — нет.
- Notification модель (5 типов), /notifications (filter=all|unread,
  unreadCount), mark read / read-all, колокольчик с бейджем в навбаре,
  мобильный drawer.

### WORKSTREAM N/O/P — Dashboard/Profile/Identity
- /dashboard/community эндпоинт + «My MTA» виджеты в существующем dashboard
  (owned servers с live-статусом, following+news+updates, discussions,
  notifications).
- /profiles/:username — публичные: серверы (только VERIFIED/ACTIVE),
  ресурсы (только PUBLISHED), badges (SERVER_OWNER/VERIFIED_SERVER/
  VERIFIED_SELLER — только реальные условия), forum-счётчики; приватное
  (email/purchases/balance) не возвращается никогда.

### WORKSTREAM Q/R — Moderation & Reporting
- /admin/servers (list/inspect с приватными данными), PATCH status
  (SUSPENDED/ARCHIVED/restore), PATCH verification (approve/reject+note) —
  каждое действие: recordAudit + MODERATION уведомление владельцу.
- Report (THREAD/POST/REVIEW/NEWS/SERVER/PROFILE) → очередь /admin/reports →
  resolve/dismiss (resolution) → уведомление репортёру + audit. Никаких
  автоматических банов.

### WORKSTREAM S/T — Privacy (критический раздел)
- showResources/showStaff/showTechStack/showCommunity = false по умолчанию;
  showStats = true по умолчанию.
- Backend-enforced: GET /servers/:slug/resources отдаёт данные ТОЛЬКО при
  opt-in (иначе enabled:false); «Этот сервер использует Resource X» никогда
  не публикуется автоматически.
- E2E-тест: privacy.test → «owner disables → public API does not expose»,
  «owner enables → relationship becomes public».

### WORKSTREAM U/V/W — Visual system & search
- Серверные страницы: banner/logo/accent в фиксированном platform layout;
  media через существующий валидированный /upload/media (magic bytes,
  5MB) + isOwnMediaUrl; lazy loading; platform typography/navigation
  неизменны.
- Форум — читаемый dense-список (ThreadRow), не клон marketplace-карточек.
- /search?q= — сгруппированные типизированные результаты (Resources/Servers/
  Discussions с count).

### WORKSTREAM Y/Z/AA — Mobile/performance/security
- Мобильные drawer/scrollable tabs/grids — всё поверх существующей системы.
- Lazy-загрузка изображений, деградация графиков (downsample ≤200 точек),
  пагинация везде.
- Проверки security: optional-auth не даёт эскалации; все мутации проверяют
  роль staff (OWNER/ADMIN/MODERATOR) на бэкенде; rate limits на создание
  контента (userRateLimit), лимиты размеров текста; audit на критических
  действиях.

### WORKSTREAM AC/§33 — Module integration
- mta-market-module: source/drm/market_client.{hpp,cpp} (HTTPS POST
  heartbeat/review-token, dedicated heartbeat thread, atomic player counts,
  privacy: только агрегаты) + source/functions/drm/market.cpp (Lua:
  mta_market_configure/start/stop/report_players/status/review_token).
- Протокол согласован с routes/integration.ts (heartbeat → {status,
  monitoring, verification}; review-tokens → {reviewToken, expiresAt}).

### §34 — Seed
- scripts/seed-plan005.ts: 10 серверов (Night City RP 428/800 ACTIVE ONLINE,
  Red County RP, Dust Rally Racing, Sunset Wire OFFLINE, Freeroam Central
  PENDING/UNKNOWN, Cedar Falls Survival showStats=false, Ghost Unit, Daybreak
  Drift, Harbor Heist SUSPENDED, Aurora League), 14 пользователей, 7
  категорий, 7 тем (19 постов, реакции, pinned), 18 новостей (3 draft), 6
  обновлений, 15 verified-отзывов, 21 подписка, 31+ уведомлений, 192 сэмпла
  мониторинга. Плюс scripts/dev-heartbeat.ts (симулятор интеграции).

### §35-40 — Testing
- Backend (vitest, +80 тестов):
  - plan005-servers.test.ts (28): регистрация, lifecycle-видимость, staff,
    edit-валидация, токен (issue/replay/forbidden), heartbeat-верификация,
    UNKNOWN через sweep, OFFLINE, archive, privacy (backend, не UI),
    follow/unfollow, admin verification/suspend + audit, 403 non-admin.
  - plan005-community.test.ts (21): хаб/категории, thread create/reply/
    edit/delete, реакции toggle, FORUM_REPLY уведомления, LOCKED → 409,
    moderator state + audit + уведомление, reports flow (create → queue →
    resolve → notify), G-004 (только свой сервер).
  - plan005-reviews.test.ts (15): eligibility 403 + объяснение, self-review
    ban, forged/wrong-server (токен не сгорает)/replay 409/expired 409,
    claim → eligibility → review (verified badge), duplicate 409,
    REVIEW_EVENT владельцу, moderation hide + notification.
  - plan005-news.test.ts (16): draft → publish (SERVER_NEWS подписчикам),
    drafts не публичны, detail с автором, обсуждение новости (newsId link),
    global feed + honest pagination, updates (dup 409), notification center
    (unread count, mark read, read-all, чужие не доступны), dashboard
    widgets.
- Playwright E2E: e2e/plan005.spec.ts (дискавери, серверная страница,
  privacy-скрытие статистики, register→create→token→news→follow→
  notification, forum create/reply/react, profile badges, notifications
  read-all).
- REGRESSION (§40): полный suite — auth, marketplace, purchases, seller,
  moderation, DRM, payments не задеты (итоговые счёта ниже).

### Финальные счёта приёмки (2026-09-10)

- Backend (vitest): **337/337** в 28 файлах (256 до плана + 81 новых:
  servers 28, community 22, reviews 15, news 16).
- Playwright browser E2E: **37/37** — полный регресс
  (plan001 полный продуктовый цикл: register → seller → wizard →
  модерация → покупка → лицензия; plan003 discovery/media/storefront;
  plan005 12 тестов: discovery с приватностью, живая страница сервера,
  форум, профиль/бейджи, register → create → token → недоступность до
  верификации, notifications read-all, follow toggle).
- Production build: server `tsc` exit 0; web `next build` exit 0
  (20 маршрутов, shared First Load JS 102 kB — workstream Z).
- Миграция: `migration plan` → `db migrate` (93 additive ops) на dev-БД;
  тестовая БД через `db update`; пакеты в git
  (migrations/app/20260910T1051_plan005_community_server + snapshot).
- Live-интеграционная цепочка: heartbeat с токеном → ONLINE + VERIFIED →
  публичная страница 431/800 → выпуск review-token (ручная проверка + api
  тесты).

## Уроки окружения (для воспроизведения прогона E2E)

1. **Артефакт подписи обязателен для plan001 E2E**: в apps/server/.env
   нужен валидный `ARTIFACT_SIGNING_PRIVATE_KEY` (Ed25519 PKCS8 base64) —
   без него wizard-загрузка артефакта падает на создании версии и тест
   «seller creates resource» зависает на шаге 3. Генерация:
   `crypto.generateKeyPairSync('ed25519')` → PKCS8 DER → base64.
2. **Rate limits для E2E**: полный suite делает сотни запросов с одного
   IP за минуты — dev-лимиты в apps/server/.env должны быть ослаблены
   (AUTH 1000, STANDARD 2000, STRICT 500, LOGIN 200). Счётчики живут в
   Redis и ПЕРЕЖИВАЮТ рестарт API — при длинной серии прогонов либо ждать
   окно, либо сбросить Redis.
3. **Stale .next = 500 на динамических страницах**: после production
   build (`next build`) в .next остаются pages-артефакты, ломающие dev-
   сервер (Cannot find module './vendor-chunks/...'). Лечение: остановить
   `next dev`, удалить .next, запустить заново. Не является багом продукта.
4. **ORM-нюансы, найденные тестами**: у ORM нет OR-комбинатора (поиск
   ветками по id) и `desc()` для boolean-полей (pinned-сортировка в JS);
   NULL-поля не допускают `??` внутри orderBy-колбэка — branch-запросы.
5. **timestamps в contract ORM** пишутся как ISO-строки
   (`new Date().toISOString()`), Date-объекты не принимаются.

## Финальный walkthrough (§42)

SCENARIO 1 (PLAYER): /servers → Night City RP (431/800 ONLINE) → news →
reviews (Verified Interaction) → community → follow → уведомление. ✅
SCENARIO 2 (SERVER OWNER): register → /servers/create (5 шагов) → токен →
heartbeat → VERIFIED → брендинг/приватность → publish news → discussion →
токен отзыва. ✅
SCENARIO 3 (COMMUNITY MEMBER): /community → category → thread → reply →
follow → notification. ✅
SCENARIO 4 (REVIEW): module issue token → claim на сервер-странице → review
→ badge. ✅ (E2E/API)
SCENARIO 5 (MODERATOR): /admin → Серверы (verify/suspend) → Жалобы
(resolve) → контент (hide review/lock thread) → audit rows. ✅

## Ограничения

- production-проверка (домен/TLS/боевой платёж) — вне скоупа, как в
  PLAN-004; blockers перенесены в CURRENT.md.
- Windows-сборка модуля — унаследованное ограничение (POSIX-сокеты в
  http_client.cpp); market_client архитектурно нейтрален к порту.
- Email-канал уведомлений не реализован (in-app только) — сознательно.
