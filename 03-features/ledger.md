# Финансовый ledger

Purchase ≠ Payment ≠ начисление продавцу ≠ выплата.

## Таблицы

- `SellerBalance`: `availableAmount`, `inEscrowAmount`, `totalEarned` (копейки).
- `FinancialTransaction`: тип (`SELLER_REVENUE`, `PLATFORM_FEE`, `REFUND_*`, …), `amount`, `balanceAfter`.

Инвариант цели: баланс = сумма проводок; проводка не апдейтится, ошибка — сторно.

## Код сегодня

`apps/server/src/lib/ledger.ts`:

- `recordSellerRevenue` читает баланс, прибавляет, пишет баланс, пишет транзакцию.
- Комментарий говорит про атомарность; **общей транзакции БД нет**.
- Проверяется `platformFee + sellerRevenue === priceSnapshot`.

Два параллельных вебхука могут удвоить начисление. Пока это так, сверка с ЮKassa обязательна даже на закрытой бете.

## Схема

`relatedPurchaseId Int?` при строковом id покупки — починить вместе с атомарностью.
