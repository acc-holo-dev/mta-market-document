# CURRENT — состояние проекта

Обновлено: 2026-09-10 (после выполнения PLAN-003).

## Активный план

Нет. PLAN-003 (Marketplace Core) выполнен и зафиксирован
(см. [COMPLETED/PLAN-003.md](COMPLETED/PLAN-003.md)).

Следующий цикл открывается новым планом в `ACTIVE/`.

## Состояние продукта

После PLAN-003 MTA Market — полноценный магазин ресурсов:

- **Media foundation**: у каждого ресурса может быть обложка и до 8
  скриншотов (server-side magic-byte валидация, opaque имена, лимиты,
  публичная раздача через `/media/`). Существующие ресурсы без медиа
  работают с типографическим fallback.
- **Seller media UX**: шаг «Оформление» в wizard создания ресурса
  (обложка + скриншоты с понятными состояниями загрузки, порядком и
  ошибками), product-like preview перед отправкой на модерацию,
  редактор оформления в кабинете продавца (для draft/pending).
- **Resource Card V2 / Resource Detail V2**: товарная карточка с обложкой,
  продавцом, типом, ценой и рейтингом; product page с галереей (cover +
  screenshots, лайтбокс с Esc/←/→), seller block, версией в hero.
- **Seller storefront**: публичная витрина `/sellers/:username` (только
  APPROVED-профиль или published-ресурсы; непубличные не отдаются).
- **Discovery**: настоящий серверный search (title/description/продавец),
  категории по реальным domain-типам, price free/paid, сортировки
  (popular по завершённым покупкам, rating, newest, price asc/desc) — всё
  с URL state (`/resources?q=hud&type=SCRIPT&price=free&sort=popular&page=2`).
- **Homepage**: реальные секции (Новинки / Популярное / Бесплатные) через
  агрегированный `/resources/homepage` (один запрос), каждая с «Смотреть всё».
- **Moderation V2**: админ видит полную карточку товара (обложка,
  скриншоты, версии с artifact/validation статусами) — «товар так, как его
  увидит покупатель». State machine не менялась.
- **Seed data**: 16 published-ресурсов всех типов с настоящими обложками/
  скриншотами (сгенерированные PNG, без stock photos), отзывами и
  завершёнными покупками (настоящий popularity signal). Идемпотентный
  `scripts/seed-plan003.ts`.

## Приёмка PLAN-003 (2026-09-10)

- Playwright browser E2E: **25/25** (PLAN-001: 12, PLAN-003: 13) на живых
  dev-серверах (PostgreSQL/Redis в Docker).
- Backend-тесты: **253/254**. Единственное падение — `startup-policy`
  acceptance (процессный таймаут в dev-окружении); воспроизводится на
  дереве до PLAN-003, не является регрессией.
- `tsc --noEmit` чист у server и web; production build web и server проходит.
- Нет критических console/runtime ошибок на ключевых страницах.

## Завершённые workstreams (PLAN-003)

Resource Media Foundation (coverUrl + ResourceMedia + upload/serving +
magic-byte security), Seller Media UX (wizard Presentation step, upload
states, reordering, preview, media editing), Resource Card V2, Resource
Detail V2 (gallery/lightbox/seller block), Seller Storefront, Marketplace
Search, Categories, Filters, Sorting (с реальным popularity), URL state,
Homepage real sections, Moderation V2 (product presentation + media),
Mobile drawer, Accessibility, Seed data (16 реалистичных ресурсов),
Testing (25 E2E), Backend contract (единый /resources query), Documentation.

## Blockers

Нет блокеров основного пользовательского сценария.

Известные ограничения (не блокируют Marketplace Core, но их нужно помнить):

- Медиа хранится локально (S3-путь реализован, но выключен в dev).
- Rating/popular сортировки капнуты 1000 записей в памяти (dev-масштаб);
  для production нужна SQL-агрегация.
- Compatibility/requirements/features показываются только при наличии
  реальных данных (сейчас их нет — UI не придумывает).
- Rate limits dev-контура подняты для E2E (AUTH/STANDARD 2000, LOGIN 500,
  REFRESH 400) — production значения пересмотреть.
- ЮKassa проверена в dev-контуре (`/payments/:id/simulate`); боевой webhook
  на реальных деньгах не прогонялся.
- DRM client: E2E против живого license-сервера и Windows-исполнение не
  проверены.
- Пополнение баланса отсутствует (UI честно сообщает).
- `startup-policy` acceptance-тест падает по процессному таймауту в текущем
  dev-окружении (воспроизводится и до PLAN-003).

## Следующий шаг

Сформировать следующий план отдельным решением (автоматически не создаётся).
Кандидаты — из [IDEAS/IDEAS.md](../IDEAS/IDEAS.md) и честных границ выше:
типичные темы — прод-контур платежей (webhook + reconciliation на живом
провайдере), пополнение баланса, наблюдаемость, DRM E2E против живого
сервера, S3-деплой медиа, SQL-агрегации для rating/popular. Наличие идеи в
IDEAS не является обязательством её реализовать.
