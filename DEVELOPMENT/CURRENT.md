# CURRENT — состояние проекта

Обновлено: 2026-09-10.

## Активный план

Нет. PLAN-001 завершён (см. [COMPLETED/PLAN-001.md](COMPLETED/PLAN-001.md)).
Следующий цикл открывается новым планом `PLAN-002` в `ACTIVE/`.

## Состояние продукта

Initial Product Release достигнут и подтверждён приёмкой 2026-09-10:

- Полный продуктовый цикл (register → login → profile → balance → Marketplace →
  resource detail → seller area → create resource → artifact upload → «Завершить» →
  `PENDING_REVIEW` → admin → approve/reject → `PUBLISHED` → покупка (free/paid) →
  license → purchases/account) **проходит в реальном браузере** (Playwright/Chromium,
  живые dev-серверы, PostgreSQL/Redis в Docker) — 12/12.
- Backend-тесты 254/254 (чистое окружение); `tsc --noEmit` чист у обоих приложений;
  production build web проходит.
- Модуль: юнит-тесты DRM-клиента — `ALL TESTS PASSED` (Linux x64).

Подробная продуктовая картина: [../PROJECT.md](../PROJECT.md).

## Завершённые workstreams (PLAN-001)

Authentication, Account/Profile, Balance, Marketplace, Seller, Moderation, Admin,
Commerce, Frontend Completion, Product Consistency, Testing (backend + browser E2E),
Release Readiness, Cleanup, Documentation (эта структура).

## Blockers

Нет блокеров основного пользовательского сценария.

Известные ограничения (не блокируют Initial Product Release, но их нужно помнить):

- ЮKassa проверена в dev-контуре (`/payments/:id/simulate`); боевой webhook на реальных
  деньгах не прогонялся.
- DRM client: E2E против живого license-сервера и Windows-исполнение не проверены.
- Пополнение баланса отсутствует (баланс реальный и персистентный).
- Restore drill / алертинг / прод-наблюдаемость не прогонялись.

## Следующий шаг

Сформировать `PLAN-002` (план развития работающего продукта, не «план спасения»).
Кандидаты в цели — брать из [IDEAS/IDEAS.md](../IDEAS/IDEAS.md) и честных границ в
PROJECT.md; типичные темы: прод-контур платежей (webhook + reconciliation на живом
провайдере), пополнение баланса, наблюдаемость/эксплуатация, DRM E2E против живого
сервера. Наличие идеи в IDEAS не является обязательством её реализовать.
