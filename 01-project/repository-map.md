status: current
version: 1.1
last_verified: 2026-09-09

# Карта репозиториев

Три репозитория. Дублировать спецификации между ними нельзя.
Детальная архитектура: [02-architecture/architecture.md](../02-architecture/architecture.md),
контракт репозиториев: [02-architecture/repository-contract.md](../02-architecture/repository-contract.md).

## mta-market-site

Веб-платформа: Express API, Next.js, Prisma, Docker.

Ответственность: аккаунты, каталог, заказы/покупки, платежи, модерация, license server HTTP.

Не класть сюда: полную спецификацию DRM, ADR, модель угроз.

## mta-market-module

Нативный модуль MTA (C++20).

Три слоя:

1. **SDK** — биндинг Lua, async, таймеры, userdata, CLI `mta`. Документация SDK: `other/documents/` (английский, это API компилятора).
2. **DRM client subsystem** — `source/drm/**`: HTTPS-клиент, защищённое key store, Ed25519, AES-256-GCM, жизненный цикл лицензии по протоколу v2 (Block 6, H-001..H-006). Юнит-тесты: `make -f source/drm/Makefile test`. Инвентарь: `docs/H-001-inventory.md`.
3. **DRM spike (исторический)** — `source/functions/drm/spike.cpp`, cipher = byte complement. Оставлен как feasibility-проверка, не production Guard.

## mta-market-document

Этот репозиторий. Контракт между сайтом и модулем, статус, ADR, платежи, безопасность.

## Контракты между репо

| Контракт | Где описан | Где код |
|---|---|---|
| HTTP API | [02-architecture/api.md](../02-architecture/api.md) | `apps/server/src/routes/` |
| Схема БД | [02-architecture/database.md](../02-architecture/database.md) | `apps/server/src/prisma/contract.prisma` |
| DRM протокол v2 (заморожен) | [03-features/drm/protocol-v2.md](../03-features/drm/protocol-v2.md) | `apps/server/src/lib/drm/protocol.ts` + `mta-market-module/source/drm/` |
| Формат артефакта | [03-features/artifacts/format.md](../03-features/artifacts/format.md) | `apps/server/src/lib/artifact/types.ts` |

Смена протокола: сначала документ и версия, потом код с обеих сторон.
