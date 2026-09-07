# База данных

Единственный файл схемы: `mta-market-site/apps/server/src/prisma/contract.prisma`.  
Этот документ — производная. При расхождении верить Prisma.

## Идентификаторы

Большинство PK: `String @id @default(cuid())`.  
Сессии хранят `refreshTokenHash` (SHA-256), `tokenFamily`, `reuseDetected`.

## Перечисления (сокращённо)

- User: `USER | ADMIN | MODERATOR`, `ACTIVE | SUSPENDED | BANNED`
- Resource: типы SCRIPT/MAP/MODEL/TEXTURE/SOUND/GAMEMODE; статусы DRAFT → PENDING_REVIEW → PUBLISHED | SUSPENDED
- Purchase: PENDING | COMPLETED | REFUNDED | FAILED (в файле enum объявлен дважды — долг)
- License: ACTIVE | REVOKED | EXPIRED
- Payment: YUKASSA | STRIPE | TEST; PENDING | SUCCEEDED | FAILED | REFUNDED
- Деньги: целые **копейки**, не рубли с плавающей точкой

## Денежные таблицы

- `Payment` — связь с провайдером, `providerPaymentId` уникален.
- `PaymentProviderEvent` — уникальность `(provider, providerEventId, eventType)`.
- `SellerBalance` — PK = `userId`.
- `FinancialTransaction` — append-only по смыслу; поле `relatedPurchaseId` сейчас `Int?` при `Purchase.id` String — **баг схемы**.

## Отзывы

`Review.resourceId Int` при `Resource.id String` — **баг схемы**.

## Миграции

Менять схему только через Prisma-контракт. Не заводить второй `schema.prisma`.
