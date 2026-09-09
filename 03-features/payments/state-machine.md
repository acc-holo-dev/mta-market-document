status: current
version: 1.0
last_verified: 2026-09-09

# Payment state machine

Source of truth: `mta-market-site/apps/server/src/lib/paymentStateMachine.ts`
(E-003). Every `Payment` status mutation goes through `assertTransition`;
illegal jumps throw `PaymentStateError`.

## States

`PENDING`, `SUCCEEDED`, `SETTLEMENT_PENDING`, `SETTLED`, `FAILED`, `CANCELED`,
`REFUNDED`, `PARTIALLY_REFUNDED` (mirrors the `PaymentStatus` contract enum).

## Allowed transitions

| From | To |
|---|---|
| `PENDING` | `SUCCEEDED`, `FAILED`, `CANCELED` |
| `SUCCEEDED` | `SETTLEMENT_PENDING`, `SETTLED`, `PARTIALLY_REFUNDED`, `REFUNDED` |
| `SETTLEMENT_PENDING` | `SETTLED`, `FAILED` |
| `SETTLED` | `PARTIALLY_REFUNDED`, `REFUNDED` |
| `PARTIALLY_REFUNDED` | `REFUNDED` (further partial refunds until fully refunded) |
| `FAILED` | — (terminal) |
| `CANCELED` | — (terminal) |
| `REFUNDED` | — (terminal) |

Notes:

- The plan shorthand "PENDING → SUCCEEDED → SETTLED" is the happy path; the
  implementation inserts the explicit `SETTLEMENT_PENDING` stage between
  success and settlement, and allows confirmed refunds from `SUCCEEDED` as
  well as from `SETTLED` (a captured-but-unsettled payment can be refunded).
- Provider-native statuses map onto this vocabulary via `fromYooKassaStatus`:
  `succeeded → SUCCEEDED`, `canceled → CANCELED`, `pending` /
  `waiting_for_capture → PENDING`; unknown → `PENDING` (conservative).

## Refund targets

`createRefund` accepts payments in `SUCCEEDED`, `SETTLED`,
`PARTIALLY_REFUNDED` only (409 `payment_not_refundable` otherwise). Effects:
`REFUNDED` when the refunded total reaches the captured amount, otherwise
`PARTIALLY_REFUNDED` — see [refunds.md](refunds.md).

## Entitlement connection

- `PENDING` purchase → download denied (403, tested).
- `SUCCEEDED`/`SETTLED` → entitlement active; license created on completion.
- `REFUNDED` (full) → purchase `REFUNDED`, license `REVOKED`, download denied.
- `PARTIALLY_REFUNDED` → entitlement stays active.

## Test evidence

`tests/ledger-refunds.test.ts` — "allows the canonical flow and rejects
illegal jumps" (transition table), plus refund-driven transitions.
