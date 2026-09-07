# Тестирование

## Site

Сейчас: lint/typecheck/build в CI. Интеграционных тестов платежа и E2E витрины нет.

Минимальный ручной прогон перед мержем денег/DRM:

1. Логин Discord, `/auth/me`.
2. Создание ресурса, версия, модерация в PUBLISHED.
3. Покупка: simulate **только** на dev; на prod-like — реальный вебхук дважды (идемпотентность).
4. Activate лицензии чужим пользователем → 403.
5. Старт API без `JWT_SECRET` при `NODE_ENV=production` → отказ.

## Module

Пирамида уже есть:

- `mta test unit` — конфиг.
- `mta test lua` — встроенный Lua 5.1.
- `mta test integration` — закреплённый MTA server.

DRM spike покрыт `096_drm_spike.lua`. Новый протокол — отдельный набор, не ломать SDK-сьюты.

## Совместимость

Пока нет CI, который гоняет site API против module. Появится вместе с wire-форматом lease.
