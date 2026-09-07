# Roadmap

Порядок — по риску денег и лицензий, не по «ещё фича».

## Сейчас

1. Починить типы в Prisma (`Review.resourceId`, `relatedPurchaseId`, дубль enum).
2. Ledger: одна БД-транзакция на изменение баланса + запись.
3. Не разъезжаться: статус и readiness обновлять вместе с кодом.

## Дальше (P0 продукта)

1. DRM v2 в модуле: Ed25519, lease 24ч, без хранения private key на license server.
2. Build Tool: байткод + AES-256-GCM, не complement-spike.
3. Песочница и подпись артефактов до PUBLISHED.
4. Сверка с ЮKassa (reconciliation worker).

## Не в ближайший MVP

Подписки, биржа заказов, чаты, Elasticsearch, 3D-превью, Leak Radar, мультилогин всеми провайдерами сразу.

Календарные «5–7 месяцев» не обещаем. Готовность = закрытые гейты в [production-readiness.md](production-readiness.md).
