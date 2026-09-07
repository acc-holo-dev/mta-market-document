# Статус

**Дата:** 2026-09-07  
**Фаза:** MVP / pre-production  
**Запуск с живыми деньгами:** нет

Статус пишется по коду, не по желаемому roadmap.

## Репозитории

| Репозиторий | Состояние |
|---|---|
| mta-market-site | MVP: API + тонкий фронт |
| mta-market-module | SDK v2.1 готов; DRM — spike |
| mta-market-document | этот канон |

## Что работает

- Backend Express + Prisma 8, идентификаторы в основном `String @id @default(cuid())`.
- Discord OAuth2, JWT, refresh в cookie, хеш refresh-токена, ротация.
- CRUD ресурсов, версии, отзывы, админ-модерация.
- Покупки, интерфейс платёжного провайдера, вебхук ЮKassa с идемпотентностью.
- `/payments/:id/simulate` только при `NODE_ENV !== production`.
- DRM v1: активация по `license.id`, случайные hex-ключи, private key хранится в БД.
- Docker Compose, rate limit через Redis, проверка секретов на старте в production.

## Чего нет (и это нормально фиксировать)

- DRM v2: Ed25519, lease, AES-256-GCM, Build Tool — только в спецификации.
- Двойная запись ledger в одной транзакции БД (комментарий в коде обещает транзакцию, кода `$transaction` нет).
- Песочница загрузок, malware-scan, подпись артефактов.
- Мультипровайдерный логин (Telegram / Яндекс / VK) — в схеме есть `Account.provider`, в роутах только Discord.
- Полноценные сервисы, скидки, заказ/корзина: модели в Prisma есть, пользовательский поток не закрыт.
- Фронт: главная, каталог, карточка, дашборд, логин — не магазин уровня API.
- Интеграционные и E2E-тесты сайта.
- Observability (трейсы, метрики, error tracking).

## Известный долг схемы

В `contract.prisma` смешанные типы:

- `Review.resourceId` объявлен как `Int`, у `Resource.id` — `String`.
- `FinancialTransaction.relatedPurchaseId` — `Int`, у `Purchase.id` — `String`.
- `PurchaseStatus` описан дважды.

Это ломает целостность модели; чинить в коде, не замалчивать в статусе.

## Оценка готовности

Ориентир: **не production**. Гейты — в [production-readiness.md](production-readiness.md).
