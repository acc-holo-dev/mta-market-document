status: current
version: 1.0
last_verified: 2026-09-09

# YooKassa provider

Source of truth: `mta-market-site/apps/server/src/lib/providers/payment-yookassa.ts`
(adapter), `src/lib/yookassa.ts` (HTTP transport), `src/lib/yookassaWebhook.ts`
(webhook verification helpers), `src/routes/payments.ts` (webhook route).

## Configuration

| Env var | Purpose |
|---|---|
| `YOOKASSA_ENABLED` | `true` enables the provider; missing credentials with it enabled is a **fatal startup error** |
| `YOOKASSA_SHOP_ID` | shop id (HTTP Basic user; webhook Basic user too) |
| `YOOKASSA_SECRET_KEY` | secret key (HTTP Basic password for API calls) |
| `YOOKASSA_NOTIFICATION_PASSWORD` | webhook Basic password (separate from the API secret key) |

Provider name in the registry: `YUKASSA`. Capabilities: `payment.create`,
`payment.verification`, `refund.create`; `payment.cancel` is declared
**implemented** (2026-09-09): `cancelPayment` -> `POST /v3/payments/{id}/cancel`, exposed as `POST /payments/cancel` (owner/admin, PENDING only, state PENDING -> CANCELED).

## Outbound API calls

- Base: `https://api.yookassa.ru/v3` (`/payments`, `/refunds`).
- Auth: HTTP Basic `shopId:secretKey` (per official docs — this is the actual
  provider protocol; no other auth layer is invented).
- Idempotency: every create call carries an `Idempotence-Key` (random UUID).
- Amounts: kopecks internally → decimal string with 2 digits on the wire
  (`(amount / 100).toFixed(2)`); responses parsed back to kopecks.
- Payments are created with `capture: true` and `confirmation: redirect`,
  `metadata.order_id` = internal order reference.
- HTTP errors include the status code so callers (reconciliation B-003) can
  separate definitive errors (404) from transient ones.

## Webhook verification (A-010) — the actual protocol, no invented HMAC

YooKassa does not sign webhooks with HMAC headers; verification is therefore:

1. **IP allowlist** (`isYooKassaIP`): YooKassa notification ranges
   (`185.71.76.0/27`, `185.71.77.0/27`, `77.75.153.0/25`, `77.75.156.11`,
   `77.75.156.35`, `77.75.154.128/25`, `2a02:5180::/32`). The checked address
   is `req.ip` — Express computes it through the trusted proxy chain
   (`trust proxy = 1` matches the nginx topology). Hand-parsing
   `X-Forwarded-For` is explicitly avoided (spoofable).
2. **HTTP Basic** (`verifyYooKassaAuth`): `Authorization: Basic
   base64(shopId:notificationPassword)`; both parts compared timing-safely
   (sha256 digest then `crypto.timingSafeEqual` — no length oracle).
3. **Provider re-fetch** (business verification, route layer): the webhook
   object id is re-fetched from the API; the **fetched** payment — not the
   webhook body — drives the entitlement decision.
4. Failure mapping: untrusted IP → 403; bad Basic → 401; provider disabled →
   503 (no unverified bypass).

## Webhook processing order

```
receive event
 → verify transport (IP + Basic)
 → persist PaymentProviderEvent
 → deduplicate (replay ⇒ one business effect; tested ×10)
 → re-fetch provider payment
 → validate: state succeeded, amount == order total, currency == order
   currency, metadata.order_id == internal reference, purchase binding
 → execute domain transition atomically (state machine + entitlement + ledger)
```

Mismatch handling (A-011): wrong amount → quarantined (no entitlement);
wrong currency → 409; non-succeeded state → acknowledged without business
effects; wrong purchase binding → 409 + quarantine; unknown purchase → 404;
missing `order_id` metadata → 400.

## Known limits (honest)

- Refund state refinement is synchronous (`succeeded → SUCCEEDED`, else
  `PENDING`); the refund webhook handler for async refinement is an E-008
  note, not implemented.
- No live-credential test run is recorded; CI runs with
  `YOOKASSA_ENABLED=false` and stubbed re-fetch
  (`tests/payments-webhook.test.ts`).
- IPv6 allowlist matching is prefix-based (`2a02:5180::/32` via startsWith),
  weaker than the v4 CIDR check.

## Test evidence

`tests/payments-webhook.test.ts`: "rejects webhooks from non-allowlisted IPs
(403)", "rejects forged Basic Auth (401)", "replaying the same event 10 times
produces ONE business effect", "rejects a wrong amount (quarantine, no
entitlement)", "rejects wrong currency (409)", "rejects when provider state is
not succeeded (409)", "rejects a provider payment bound to a different
purchase (409, quarantined)", "acknowledges non-succeeded event types without
business effects", "rejects unknown purchase reference (404)", "rejects
missing order_id metadata (400)", "returns 503 for /payments/webhook when
YOOKASSA_ENABLED=false".
