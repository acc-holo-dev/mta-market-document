# PROJECT — MTA Market

Обновлено: 2026-09-10 (после выполнения PLAN-004 — Production Readiness & Operational Hardening).

Этот документ описывает текущее состояние MTA Market с точки зрения продукта.
Это не roadmap и не task list — развитие проекта ведётся через
[Development Plan system](DEVELOPMENT/README.md).

---

## Что такое MTA Market

MTA Market — маркетплейс серверных ресурсов для Multi Theft Auto: San Andreas
(MTA:SA): скрипты, карты, модели, гейммоды. Продавцы публикуют ресурсы,
покупатели приобретают или бесплатно получают их, а лицензии на платные
ресурсы защищены DRM.

Честная формулировка DRM: защита **повышает стоимость массового копирования**
и делает лицензии контролируемыми и атрибутируемыми; это не абсолютная защита
от декомпиляции. Покупка ≠ платёж ≠ лицензия — это разные сущности с разными
машинами состояний.

## Репозитории и зачем каждый нужен

| Репозиторий | Зачем существует |
|---|---|
| `acc-holo-dev/mta-market-site` | Всё веб-приложение: Express API (портреты auth, ресурсы, версии/артефакты, покупки, платежи, модерация, admin, DRM-сервер v2, ledger) и Next.js frontend. БД — PostgreSQL, кэш/rate-limit — Redis. |
| `acc-holo-dev/mta-market-module` | Нативный C++ модуль для серверов MTA: зрелый SDK (Lua-биндинги) + DRM client subsystem, который общается с сервером площадки по замороженному DRM Protocol v2 и расшифровывает защищённые ресурсы. |
| `acc-holo-dev/mta-market-document` | Продуктовая документация и Development Plan system (этот репозиторий). Не содержит кода и дублей схем. |

## Основные части системы (уже существуют)

- **Аутентификация**: локальная регистрация/вход (username или email + пароль, bcrypt),
  OAuth (Discord/Yandex/Google), связывание идентичностей, JWT access + refresh-ротация
  в HttpOnly cookie, детекция переиспользования refresh-токена, logout.
- **Профиль и баланс**: отображаемое имя/аватар редактируются; баланс пользователя —
  реальная строка в БД (`userBalance`, копейки/RUB), у каждого нового аккаунта 0.
- **Маркетплейс**: каталог опубликованных ресурсов с фильтрами (бесплатные/платные, тип),
  карточка ресурса с версиями и отзывами.
- **Seller flow**: заявка «стать продавцом» → одобрение админом → кабинет продавца
  (ресурсы, услуги, статусы) → мастер создания ресурса (инфо → тип/цена → artifact →
  отправка на модерацию).
- **Artifact pipeline**: загрузка архива, SHA-256, строгая валидация (статический анализ
  содержимого ZIP), Ed25519-подпись артефакта платформой, версии релизов
  (CANDIDATE → VERIFIED → PUBLISHED, yank).
- **Модерация**: очередь `PENDING_REVIEW`, approve → `PUBLISHED`, reject → `SUSPENDED`
  с причиной; append-only лог `moderationEvent` + audit trail.
- **Коммерция**: checkout создаёт Order + OrderItem + Purchase; бесплатный ресурс
  завершается сразу без платёжного провайдера; платный — через ЮKassa (webhook) или
  dev-completion (`/payments/:id/simulate`, недоступен в production); после завершения
  выдаётся License; скидки/промокоды с неизменяемым снимком строки заказа.
- **Финансовый ledger**: двойная запись по счетам, платформенная комиссия и выплата
  продавцу считаются при завершении; reconciliation-джобы.
- **DRM v2**: Ed25519-подписи, challenge → installation → activate → lease (7 дней),
  DEK aes-256-gcm, rotation ключей. Сервер — в `mta-market-site`, клиент — в
  `mta-market-module/source/drm/`.
- **Споры (disputes)**: покупатель открывает спор по завершённой покупке, сообщения,
  переходы статуса админом.
- **Admin Panel**: статистика платформы, модерация ресурсов, одобрение продавцов,
  споры, отзыв версий (yank).

## Как взаимодействуют роли

**Buyer (покупатель)**
1. Регистрируется/входит (пароль или Discord), видит баланс 0.00 ₽ в шапке.
2. Ищет ресурс в Marketplace, фильтрует бесплатные/платные, открывает карточку.
3. Бесплатный — кнопка «Получить» (мгновенно, без платежа); платный — «Купить сейчас»
   и оплата через ЮKassa.
4. Видит покупки и лицензии в «Профиль» и «Покупки», оставляет отзыв после покупки,
   может открыть спор.

**Seller (продавец)**
1. Подаёт заявку «Стать продавцом»; админ одобряет в Admin Panel.
2. В кабинете видит свои ресурсы и статусы, создаёт ресурс через мастер
   (основное → тип/цена → файл+версия → проверка → «Завершить»).
3. Ресурс уходит в `PENDING_REVIEW`; результат модерации виден в кабинете
   («Опубликован» / «Приостановлен»).

**Admin (админ/модератор)**
1. Заходит в Admin Panel (пункт «Админ» в навигации виден только ADMIN/MODERATOR).
2. Видит очередь модерации с метаданными ресурса, публикует или отклоняет с причиной.
3. Одобряет/отклоняет заявки продавцов, разбирает споры, может yank версию.
4. Обычный пользователь в Admin Panel не допускается — ни UI, ни API (403).

## Реализованные бизнес-сценарии

1. Регистрация → вход (username/email/Discord) → профиль → баланс → сессия переживает reload.
2. Продавец: регистрация → заявка → одобрение → создание ресурса → загрузка artifact
   (реальная подпись) → «Завершить» → `PENDING_REVIEW`.
3. Модерация: admin публикует → ресурс появляется в Marketplace; admin отклоняет →
   ресурс не публикуется, причина сохранена, продавец видит статус.
4. Покупка: бесплатное получение (без записи payment, с лицензией) и платная покупка
   (payment → завершение → purchase `COMPLETED` + license `ACTIVE`).
5. Спор по завершённой покупке; админ-разбор.
6. DRM-жизненный цикл: entitlement → installation → challenge → activate → lease
   (протокол v2; клиент в модуле, серверные тесты проходят).

## Что считается Initial Product Release

**PLAN-001 = Initial Product Release**: первый цельный рабочий продукт, в котором
полный пользовательский цикл — регистрация → покупка/продажа → модерация → публикация →
лицензия — проходит в реальном браузере без ручного вмешательства разработчика.

Приёмка PLAN-001 (2026-09-10): браузерный E2E (Playwright/Chromium против живых
dev-серверов и БД) 12/12; backend-тесты 254/254; typecheck обоих приложений чистый;
production build frontend проходит; модульные тесты DRM-клиента проходят.
Детали: [DEVELOPMENT/COMPLETED/PLAN-001.md](DEVELOPMENT/COMPLETED/PLAN-001.md).

## Product Experience (PLAN-002)

**PLAN-002 = Product Experience Foundation**: превращение работающего Initial Product
Release в цельный визуально и UX-зрелый marketplace-продукт без добавления новых
backend-функций и без переписывания архитектуры. Полный фронтенд-редизайн поверх
существующей функциональности:

- **Design system**: единые design tokens (тёмная navy-тема, контрастные surfaces,
  один blue accent, spacing/radius/typography) в `globals.css` + `tailwind.config.js`;
  расширенный UI-kit (Tabs, Select, Modal/ConfirmDialog, Avatar, Badge-терминология,
  Skeleton, Rating, Price, ResourceCard, SectionHeader, EmptyState с иконками).
- **Навигация**: продуктовый Navbar (Маркетплейс / Покупки / Профиль / Мой магазин /
  Админ по роли; баланс и аватар справа) с полноценной мобильной версией (drawer),
  активные состояния, sticky-хедер без layout-скачков; профессиональный Footer
  с актуальным годом.
- **Homepage**: marketplace-oriented вместо landing — hero с CTA «Маркетплейс»,
  реальные секции (Новинки, Высокий рейтинг, Бесплатные ресурсы) через существующий
  API, неброский seller CTA.
- **Маркетплейс**: магазинная сетка 1→4 колонки, фильтры с русскими подписями реальных
  enum-значений, skeleton loading, отдельные empty states (пустой каталог / пустые
  фильтры), полноценная товарная карточка (типографический cover без фейковых
  картинок, цена, рейтинг, продавец).
- **Resource Detail**: product page — hero area, breadcrumbs, DRM-объяснение,
  таймлайн версий, полноценные отзывы, sidebar покупки («Уже приобретено» после
  покупки).
- **Account**: единая area с вкладками (Профиль / Обзор / Подключения / Баланс /
  Покупки), история покупок с лицензиями и спорами.
- **Seller Studio**: «Мой магазин» — дашборд с показателями (ресурсы/опубликовано/
  на модерации/черновики/продано), управление ресурсами с наглядными состояниями.
- **Admin**: «Панель управления» — дашборд статистики, вкладки модерации/продавцов/
  споров/версий, человекочитаемые события модерации и переходы споров.
- **Качество**: честные тексты без непроверяемых обещаний (убрано «мгновенные выплаты»),
  единая русская терминология, dynamic copyright year, focus-visible и aria-атрибуты.

Приёмка PLAN-002 (2026-09-10): браузерный E2E PLAN-001 12/12 после редизайна;
typecheck обоих приложений чистый; production build web проходит; backend-тесты
253/254 (единственное падение — `startup-policy` acceptance, процессный таймаут
в dev-окружении; падает и на дереве до PLAN-002).
Детали: [DEVELOPMENT/COMPLETED/PLAN-002.md](DEVELOPMENT/COMPLETED/PLAN-002.md).

## Production Readiness (PLAN-004)

**PLAN-004 = Production Readiness & Operational Hardening**: перевод проекта из
«working development product» в «production-ready service» без новых product
features. Ключевые результаты:

- **Единый media storage контракт** (B-001/B-002): `/upload/media` в S3-режиме
  сохраняет объект под `media/<opaque-name>` и возвращает контролируемый
  публичный путь `/media/<name>` (а не абсолютный S3 URL, ломавший контракт
  resource media); `GET /media/:name` работает в обоих режимах (стриминг из
  бакета или 302 на `MEDIA_PUBLIC_BASE_URL`); arbitrary external URLs
  по-прежнему запрещены, платные артефакты приватны.
- **Production configuration** (A): полная матрица env (`.env.example` с
  DEVELOPMENT/STAGING/PRODUCTION-пометками), fail-fast startup validation
  (включая `DRM_MASTER_KEY`), production rate limits зафиксированы в
  `docker-compose.prod.yml` и не наследуют dev/E2E значения, cookies/origin
  policy задокументированы.
- **Инфраструктура** (H/L/K): `IMAGE_TAG`-пиннинг и хранение предыдущей версии
  (rollback одной командой), health-gated deploy (`compose up --wait`), шаг
  миграций в deploy (backup → migrate → deploy → verify), исправленный
  uploads-volume (chown в Dockerfile), nginx: закрыт латентный `/uploads`
  bypass, строгий CSP, HSTS, security headers, удалены WebSocket-директивы.
- **Финансовая целостность** (D/E): исправлен знак кэша баланса продавца при
  refund (расхождение с double-entry), детерминированный id settlement-транзакции
  + repair-путь в alreadyCompleted-ветке (закрыто crash-окно между
  завершением покупки и ledger-начислением), обработка `payment.canceled`
  (PENDING больше не висит вечно; provider truth wins при позднем succeeded),
  единый маппинг статусов в reconciliation.
- **DRM client** (F): исправлен межрепозиторный P0 — клиент подписывал
  challenge как ASCII base64-текст, сервер ожидает подпись над декодированными
  байтами; контракт зафиксирован interop-тестом.
- **Безопасность** (G/K): CI security gate реально блокирует (audit-gate.sh с
  waivers + TTL), публикация образов гейтится на test/security/lint, все 23
  high/critical уязвимости закрыты (next 15.5.24 + overrides), CSP/HSTS на
  уровне nginx, `.dockerignore` исключает вложенные .env из build context.
- **Надёжность** (M/J): Redis rate-limit fail-closed для security-critical
  лимитеров, graceful shutdown закрывает HTTP + Redis + DB, глобальный
  error-middleware (async-ошибка больше не крушит процесс),
  `unhandledRejection` handler.
- **Документация** (T/U): runbook `docs/operations/production-runbook.md`,
  `docs/operations/database-migrations.md`, `docs/operations/backup-restore.md`
  (RPO 24h / RTO 4h), backup.sh читает uploads из Docker volume.

Приёмка PLAN-004 (2026-09-10): backend-тесты **256/256** (включая исторически
падавший startup-policy acceptance), browser E2E **25/25**, typecheck server/web
чистый, production build server/web проходит, lint чистый, модульные тесты
DRM-клиента **ALL TESTS PASSED** (включая новый challenge interop-тест),
audit-gate — 0 high/critical. Статус: **IMPLEMENTATION COMPLETE —
PRODUCTION NOT VERIFIED** (реальные P0-дриллы на живой инфраструктуре —
sandbox-платёж, Windows DRM, боевой домен — требуют окружения, см. CURRENT.md).
Детали: [DEVELOPMENT/COMPLETED/PLAN-004.md](DEVELOPMENT/COMPLETED/PLAN-004.md).

## Честные границы текущего состояния

- Платежи: webhook/idempotency/state machine покрыты тестами; боевой контур
  ЮKassa (sandbox → real money, D-007) не прогонялся — нужен staging с
  sandbox-магазином и зарегистрированным webhook URL.
- DRM: серверная сторона и клиентские примитивы проверены; live-флоу против
  работающего сервера и **Windows-исполнение модуля** не выполнялись (F-003) —
  сборка MSVC/MinGW и DPAPI key store ни разу не компилировались.
- Payouts: контур «начислено → выведено» не замкнут (нет списания
  availableAmount и SELLER_PAYOUT) — закрыть до реальных выплат продавцам.
- Password reset/смена пароля с инвалидацией сессий отсутствуют (O-001/O-003) —
  при production-запуске пароль-логина нужен минимальный flow.
- Rating/popular сортировки всё ещё in-memory (кап 1000) — SQL-агрегация
  необходима перед ростом каталога (R-002).
- Restore drill, load-тест и внешний uptime-мониторинг — процедуры
  задокументированы, но не прогонялись на реальном deployment'е.
- Пополнение баланса не реализовано (UI честно сообщает).

Эти пункты — кандидаты в следующий план (см. [DEVELOPMENT/CURRENT.md](DEVELOPMENT/CURRENT.md)).
