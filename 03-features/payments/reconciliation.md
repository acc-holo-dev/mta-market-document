status: current
version: 1.0
last_verified: 2026-09-09

# Reconciliation

Source of truth: `mta-market-site/apps/server/src/lib/reconciliation/`
(`service.ts`, `internal.ts`, `types.ts`) and `src/jobs/reconciliation.ts`
(scheduler, B-003/E-009).

## The daily cycle

`startReconciliationScheduler()` is wired in `src/index.ts` after boot. Default
interval **24 h** (`RECONCILIATION_INTERVAL_MS` env), first tick after 60 s,
graceful `stop()` on shutdown, no-op in test env. Windows tile the timeline:
`periodStart = now - interval`, `periodEnd = now`.

Five independent steps (a failing step is recorded and never crashes the
server):

1. **payment reconciliation** — provider `YUKASSA`: real provider re-fetch
   when enabled; amount/status mismatches and missing provider records are
   reported.
2. **refund reconciliation** — internal scan of REFUNDED payments; provider
   refund API side arrives with the refund webhook phase (reported
   unavailable — no fake mismatches).
3. **payout reconciliation** — internal `SELLER_PAYOUT` ledger scan; provider
   payout source is a later phase.
4. **provider event mismatch** — `PaymentProviderEvent` rows vs `Payment`
   rows: events without an internal payment, or with FAILED processing.
5. **internal ledger check** — purchases vs seller balances full scan
   (`reconcileAllPurchases`).

## Reports and mismatches

- Each run persists a `ReconciliationReport` (period, step, status:
  `completed | mismatches_found | failed | skipped`, mismatchCount) and
  `ReconciliationMismatch` rows (kind — e.g. `provider_event_mismatch` —
  references, payload).
- Mismatch kinds per `lib/reconciliation/types.ts` include amount/status/
  missing-provider/provider-event classes; `status` of a result is
  `ok | mismatches_found`.
- Alerting: structured log lines (metric source until O-001 grows) +
  persistent alertable rows. `getReconciliationSummary()` aggregates recent
  runs; `resolveMismatch()` closes the loop; admin surface exposes reports.
- Manual run: `pnpm --filter @mta-market/server reconciliation:run` (yesterday
  window) and `reconciliation:summary`.

## What reconciliation does NOT do (honest)

- It does not auto-correct money: mismatches are reported for human/ops
  action, not silently mutated.
- Refund/payout provider-side verification is not implemented yet (steps 2–3
  are internal scans).
- There is no alerting channel (email/IM) wired to FAILED reports yet — a
  structured log + DB row is the current output.

## Test evidence

`tests/reconciliation.test.ts`: "runs all five steps and persists reports",
"detects amount/status/missing-provider mismatches via provider re-fetch",
"flags provider events without internal payment or with FAILED processing",
"summary is readable after runs", "is a no-op in test environment",
"executes cycles on an interval and stops cleanly".
