# Карта репозиториев

Три репозитория. Дублировать спецификации между ними нельзя.

## mta-market-site

Веб-платформа: Express API, Next.js, Prisma, Docker.

Ответственность: аккаунты, каталог, заказы/покупки, платежи, модерация, license server HTTP.

Не класть сюда: полную спецификацию DRM, ADR, модель угроз.

## mta-market-module

Нативный модуль MTA (C++20).

Два слоя:

1. **SDK** — биндинг Lua, async, таймеры, userdata, CLI `mta`. Документация SDK: `other/documents/` (английский, это API компилятора).
2. **DRM spike** — `source/functions/drm/spike.cpp`, cipher = byte complement. Не production Guard.

## mta-market-document

Этот репозиторий. Контракт между сайтом и модулем, статус, ADR, платежи, безопасность.

## Контракты между репо

| Контракт | Где описан | Где код |
|---|---|---|
| HTTP API | [02-architecture/api.md](../02-architecture/api.md) | `apps/server/src/routes/` |
| Схема БД | [02-architecture/database.md](../02-architecture/database.md) | `apps/server/src/prisma/contract.prisma` |
| DRM протокол | [03-features/drm.md](../03-features/drm.md) | spike в module, v1 в `routes/drm.ts` |

Смена протокола: сначала документ и версия, потом код с обеих сторон.
