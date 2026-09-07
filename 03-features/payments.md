# Платежи

## Модель

`Payment` — факт у провайдера. `Purchase` — право на товар. Их нельзя сливать в одну таблицу «оплачено».

Целевая цепочка: Order → Payment → Purchase (+ License).  
Текущий API: `POST /purchases` создаёт покупку; `POST /payments/create` создаёт платёж; вебхук двигает статусы.

## Провайдер

Интерфейс: `apps/server/src/lib/paymentProvider.ts` (`IPaymentProvider`).  
Живая реализация: ЮKassa. STRIPE/TEST — значения enum, не полноценные адаптеры.

Деньги в **копейках**. Валюта по умолчанию RUB.

## Вебхук

`POST /payments/webhook`. События пишутся в `PaymentProviderEvent` с уникальностью `(provider, providerEventId, eventType)`, чтобы повтор не проводил оплату дважды.

Холд денег — у ЮKassa (Safe Deal / split), не «поле escrow в Postgres как касса» (ADR-009).

## Dev-обход

`POST /payments/:id/simulate` собирается только если `NODE_ENV !== production`. В бою маршрут отсутствует. Не добавлять `complete` без провайдера.

## Выплаты продавцам

Отдельный продуктовый поток. Ledger фиксирует начисление; перевод — операция провайдера + сверка.
