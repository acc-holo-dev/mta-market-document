status: current
version: 1.0
last_verified: 2026-09-09

# Payments — overview

Source of truth: `mta-market-site/apps/server/src/lib/paymentProvider.ts`,
`src/lib/commerce.ts`, `src/routes/payments.ts`. Sub-pages:
[state-machine.md](state-machine.md) · [yookassa.md](yookassa.md) ·
[refunds.md](refunds.md) · [reconciliation.md](reconciliation.md).

## Architecture

Provider-neutral core, one implemented adapter:

```
routes/payments.ts ──> PaymentProviderRegistry (E-010)
                          │ get(name) / getDefault()
                          ▼
                   IPaymentProvider (E-001)         lib/refunds.ts (E-008)
                          │                              │
                          ▼                              ▼
              YooKassaPaymentProvider (E-002)    applyRefundEffects (K-004/F-003)
                          │ wraps
                          ▼
                   lib/yookassa.ts (HTTP Basic transport)
```

- Routes call the registry/provider, never a provider SDK directly.
- Internal `Payment` rows are provider-agnostic (`provider` +
  `providerPaymentId`); no core invariant depends on YooKassa naming.
- Money is represented in **minor units** (kopecks) internally; `PaymentAmount
  {value, currency}` with ISO 4217 at the provider boundary.
- Registry admits future providers (T-Bank, Alfa, crypto — all PLANNED, see
  [../01-project/status.md](../01-project/status.md)); nothing else is
  implemented.

## IPaymentProvider surface

`createPayment`, `getPayment`, `cancelPayment` (unwired for YooKassa —
declared unsupported), `createRefund` (idempotence key), `verifyWebhook`
(transport authenticity only: IP allowlist / Basic auth; business checks are
re-fetch based, in the route layer), `isEnabled`, `supportsCapability`
(capabilities: `payment.create`, `payment.verification`, `payment.cancel`,
`refund.create`).

## Checkout to entitlement flow

1. `POST /payments/create` (authenticated): `commerce.createResourceCheckout`
   resolves the published resource + version, validates the discount code
   (campaign + per-user + usage limits), snapshots the order line immutably
   (title, unit price, discount amount — C-006), creates Order + OrderItem +
   Purchase (PENDING).
   - `finalTotal == 0` (free or fully discounted, C-003/C-008): completes
     atomically **without any payment intent** — no zero-amount provider
     call; ledger posts nothing (F-005).
2. Paid: provider `createPayment` → redirect URL to the provider.
3. Provider webhook `POST /payments/webhook`: transport verification (IP
   allowlist + Basic) → persist `PaymentProviderEvent` → deduplicate →
   **re-fetch the provider payment** → validate amount/currency/binding/state
   (A-011) → state transition + entitlement atomically. Mismatches are
   **quarantined** (no entitlement, alertable row).
4. `/payments/:id/simulate` exists only when `NODE_ENV !== "production"`
   (A-005; 404 otherwise — tested).

## Webhook hardening summary (A-010)

- IP allowlist (YooKassa ranges) checked against `req.ip` with
  `trust proxy = 1` — never a raw `X-Forwarded-For` element.
- HTTP Basic with shopId + notification password, timing-safe comparison.
- Provider re-fetch before any entitlement decision; no HMAC headers are
  invented (YooKassa's actual protocol has none).
- Webhook returns 503 when the provider is disabled — no unverified bypass.
- Replay idempotency: same event N times → one business effect.
Details: [yookassa.md](yookassa.md).

## Amount invariant (A-011)

For a paid order: provider amount == order final total; provider currency ==
order currency; provider reference (`metadata.order_id`) == internal
payment/order reference; provider state must be `succeeded`. Any mismatch →
rejected/quarantined, no entitlement.

## Test evidence

- `tests/payments-webhook.test.ts` — IP/auth/replay/amount/currency/binding/
  state/A-005/A-010 cases.
- `tests/commerce.test.ts` — checkout, discounts, C-007 concurrency.
- `tests/ledger-refunds.test.ts` — settlement, refunds, INV-012/INV-013.
- `tests/reconciliation.test.ts` — provider re-fetch mismatches.
