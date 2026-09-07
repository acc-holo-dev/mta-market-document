# Безопасность (как в коде)

Полные аудиты удалены: они врали статусом. Живые дыры — в [status.md](../01-project/status.md) и [threats.md](threats.md).

## Есть

- JWT + Discord OAuth; refresh hash в БД, ротация, reuse flag.
- Fail-fast секретов в production (`startupValidation.ts`).
- Rate limit Redis (`standard` / `auth` / `strict`).
- Часть входов через Zod (`validate` middleware).
- CUID-проверка параметров (`validateCuid`).
- Владелец лицензии при activate.
- Simulate оплаты вырезан из production-сборки маршрутов.
- Идемпотентность вебхуков на уровне таблицы событий.
- Prisma (не сырой SQL) для запросов.

## Нет / опасно

- Private key installation в БД и в ответе API.
- Ledger без транзакции.
- Загрузки без песочницы и без подписи.
- Статика `/uploads` с диска приложения.
- Не все эндпоинты с Zod.
- DRM verify принимает пару ключей без подписи challenge.
- Compose содержит дефолтный JWT и пароль Postgres.

## Принцип

Новый эндпоинт с деньгами или файлами: валидация, authz по владельцу, без dev-байпаса в production, строка в API-доке.
