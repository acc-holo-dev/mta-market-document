status: current
version: 1.0
last_verified: 2026-09-09

# 09-legacy — historical documents

Policy (P-005/P-006): historical reports and superseded documents live here,
marked **HISTORICAL** (header `status: historical`), and are **never used as a
current source of truth**. They are kept only as an audit trail of decisions
and past claims. The current state of the project is
[../01-project/status.md](../01-project/status.md); the canonical index is
[../00-INDEX.md](../00-INDEX.md).

## What lives here

| File | What it was | Why historical |
|---|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Proposed documentation structure | superseded by the current `00-INDEX.md` + directory layout |
| [COMPLETE_SESSION_SUMMARY_FINAL.md](COMPLETE_SESSION_SUMMARY_FINAL.md) | Session report 2026-09-07 ("17 tasks complete") | dated snapshot; claims superseded by verified status |
| [FINAL_SESSION_COMPLETE.md](FINAL_SESSION_COMPLETE.md) | Session report 2026-09-07 | dated snapshot |
| [SESSION_COMPLETE_FINAL.md](SESSION_COMPLETE_FINAL.md) | Session report 2026-09-07 | dated snapshot |
| [backend-summary-2026-09-07.md](backend-summary-2026-09-07.md) | Backend stage summary (moved from `01-project/summary.md`) | contains "100% complete" claims that P-003 explicitly forbids unless proven — kept as a record of what was claimed, not what is true |
| [doc-map-v1.md](doc-map-v1.md) | First-generation documentation map | replaced by `00-INDEX.md` |

## Related (kept in place, historical by nature)

- `08-reports/**` — stage/task verification reports (Block 0–7, task
  completions). They are dated evidence records; the *current* evidence table
  is [../01-project/status.md](../01-project/status.md).
- `04-security/audit-2025-01.md` — old audit; superseded by
  [../04-security/threat-model-linkage.md](../04-security/threat-model-linkage.md)
  for current requirement→test linkage (its qualitative threats remain
  readable in `04-security/threats.md`).

## Rule for new documents

Every architecture/security/feature document carries a P-006 header:

```yaml
status: current | historical
version: x.y
last_verified: YYYY-MM-DD
```

A document whose content is no longer true is either corrected or moved here
with `status: historical` — it is never left in place silently wrong.
