status: current
version: 1.0
last_verified: 2026-09-09

# Refunds

Source of truth: `mta-market-site/apps/server/src/lib/refunds.ts` (E-008) and
`src/routes/payments.ts` (admin-only HTTP surface). Policy inputs: INV-013
(refund cannot exceed captured amount), K-004 (refund vs license),
F-003 (balanced ledger posting).

## Refund lifecycle

A refund is an independent lifecycle attached to a captured payment:

```
PENDING → SUCCEEDED | FAILED | CANCELED
```

`createRefund({actorId, paymentId, amount?, reason?})`:

1. Payment must exist and be in `SUCCEEDED` / `SETTLED` / `PARTIALLY_REFUNDED`
   (409 `payment_not_refundable` otherwise).
2. **INV-013 ceiling**: prior refunds with status `SUCCEEDED` **or `PENDING`**
   (in-flight counts against the cap) are summed; `refundable = amount -
   refundedSoFar`. Requesting more → 409 `refund_exceeds_captured`. This is
   validated before the provider call.
3. `Refund` row created with status `PENDING`.
4. Provider `createRefund` (idempotence key = UUID).
5. Provider `succeeded` → row `SUCCEEDED` + `applyRefundEffects`; provider
   `pending` → stays `PENDING` (logged); provider error → row `FAILED` with
   `lastError`, 502 `provider_refund_failed`.

HTTP surface: `POST /payments/refunds` (ADMIN only; 403 tested),
`GET /payments/:paymentId/refunds` (ADMIN only).

## K-004 policy: refund vs entitlement/license

Documented policy (deliberate, tested):

- **Full confirmed refund** → payment `REFUNDED`, purchase status `REFUNDED`
  (`refundedAt` set), license `REVOKED`, download denied (403).
- **Partial refund** → payment `PARTIALLY_REFUNDED`; **license stays ACTIVE**
  — the buyer paid the remainder.
- Revocation happens only when the refund is **confirmed** by the provider —
  never on a mere refund request.

## Effects of a confirmed refund (`applyRefundEffects`)

- Idempotent: re-running for the same refund is a no-op guard + state-machine
  idempotency for `REFUNDED`/`PARTIALLY_REFUNDED` targets.
- Entitlement resolution: resource purchases only (`payment.purchaseId` →
  `Purchase` → `Resource` → `sellerId`, revenue split).
- Ledger posting (F-003), proportional to the original revenue split:
  - `DEBIT  seller_available:<sellerId>` (seller part), or
    `DEBIT  refund_reserve` when no seller;
  - `DEBIT  platform_revenue` (platform part);
  - `CREDIT platform_cash` (full refunded amount).
  All entries share `transactionId = refund:<refundId>`, memo
  `refund:<refundId>`; `postLedgerEntries` rejects an unbalanced group
  (INV-012).
- Legacy cash cache: `SellerBalance` / `FinancialTransaction` rows of type
  `REFUND_FROM_SELLER` still written for reconciliation compatibility.

## Post-refund download policy

`tests/download-auth.test.ts`: "refunded entitlement cannot download (403)".
A partially refunded payment keeps download access (not covered by a
dedicated test — recorded honestly).

## Test evidence

`tests/ledger-refunds.test.ts`: "rejects a refund request from a non-admin
(403)", "partial refund -> PARTIALLY_REFUNDED, license stays active", "full
refund -> REFUNDED, purchase refunded, license revoked", "rejects refunds
exceeding the captured amount (INV-013, 409)", "rejects refunding a payment
that was never captured (409)", plus balanced-ledger refund assertions.
