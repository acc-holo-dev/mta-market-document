# CURRENT — состояние проекта

Обновлено: 2026-09-10 (после выполнения PLAN-002).

## Активный план

Нет. PLAN-002 (Product Experience Foundation) выполнен и зафиксирован
(см. [COMPLETED/PLAN-002.md](COMPLETED/PLAN-002.md)).

Следующий цикл открывается новым планом `PLAN-003` в `ACTIVE/`.

## Состояние продукта

После PLAN-002 MTA Market визуально и логически воспринимается как единый
marketplace-продукт:

- Единый тёмный design system (design tokens, контрастные surfaces, один blue
  accent), расширенный UI-kit поверх существующих компонентов.
- Продуктовая навигация: desktop navbar (Маркетплейс / Покупки / Профиль /
  Мой магазин / Админ по роли) + полноценная мобильная версия (drawer).
- Homepage — marketplace-oriented: реальные секции ресурсов через существующий API.
- Marketplace — магазинная сетка с фильтрами, skeleton/empty/error states,
  полноценная товарная карточка (цена, рейтинг, продавец, типографический cover).
- Resource Detail — полноценная product page (hero, DRM, таймлайн версий, отзывы,
  purchase sidebar).
- Account — вкладки (Профиль / Обзор / Подключения / Баланс / Покупки).
- Seller — «Мой магазин» с дашбордом и наглядными состояниями ресурсов.
- Admin — «Панель управления» с дашбордом и человекочитаемой модерацией.

## Приёмка PLAN-002 (2026-09-10)

- Playwright browser E2E PLAN-001: **12/12** после редизайна (живые dev-серверы,
  PostgreSQL/Redis в Docker).
- Backend-тесты: 253/254. Единственное падение — `startup-policy` acceptance
  (процессный таймаут в dev-окружении); воспроизводится на дереве до PLAN-002,
  не является регрессией.
- `tsc --noEmit` чист у server и web; production build web проходит.

Подробная продуктовая картина: [../PROJECT.md](../PROJECT.md).

## Завершённые workstreams (PLAN-002)

Design System, Global Layout & Navigation, Homepage, Marketplace (+ResourceCard),
Resource Detail, Account Experience, Seller Studio, Admin Experience,
Mobile/Responsive (drawer, touch targets, grids), Accessibility (semantic HTML,
focus-visible, aria), UX States (loading/empty/error/success), Content Quality
(честные тексты, единая терминология, dynamic year), Frontend Architecture
(tokens + shared components, без дублирования), Testing (E2E regression),
Documentation (PROJECT.md, этот файл, PLAN-002 record).

## Blockers

Нет блокеров основного пользовательского сценария.

Известные ограничения (не блокируют Product Experience, но их нужно помнить):

- ЮKassa проверена в dev-контуре (`/payments/:id/simulate`); боевой webhook на
  реальных деньгах не прогонялся.
- DRM client: E2E против живого license-сервера и Windows-исполнение не проверены.
- Пополнение баланса отсутствует (UI честно сообщает о недоступности).
- Cover/screenshot поля у Resource отсутствуют в backend — карточки используют
  типографический cover (зафиксировано как backend gap в PLAN-002 record).
- Серверный sorting отсутствует — подбор секций homepage из существующих данных.
- Restore drill / алертинг / прод-наблюдаемость не прогонялись.
- `startup-policy` acceptance-тест падает по процессному таймауту в текущем
  dev-окружении (воспроизводится и до PLAN-002).

## Следующий шаг

Сформировать `PLAN-003`. Автоматически PLAN-003 не создаётся — сначала зафиксировано
фактическое состояние (PROJECT.md), следующий план определяется отдельно. Кандидаты
в цели — брать из [IDEAS/IDEAS.md](../IDEAS/IDEAS.md) и честных границ PROJECT.md;
типичные темы: cover/screenshot система для ресурсов, прод-контур платежей
(webhook + reconciliation на живом провайдере), пополнение баланса, наблюдаемость,
DRM E2E против живого сервера. Наличие идеи в IDEAS не является обязательством
её реализовать.
