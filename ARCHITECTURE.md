# MTA Market Documentation Architecture

## Proposed Structure

```
mta-market-document/
├── README.md                          # Repository overview
├── 00-INDEX.md                        # Navigation & quick links
│
├── 01-project/                        # Project Management
│   ├── status.md                      # Current status (ex: PROJECT_STATUS.md)
│   ├── roadmap.md                     # Future plans (ex: 12_ROADMAP.md)
│   ├── changelog.md                   # Version history
│   └── summary.md                     # Executive summary
│
├── 02-architecture/                   # System Design
│   ├── overview.md                    # High-level architecture (03_Архитектура.md)
│   ├── domain-model.md                # Domain entities (16_Модель_домена.md)
│   ├── database.md                    # DB schema (07_База_данных.md)
│   ├── api.md                         # API specification (08_API.md)
│   └── decisions.md                   # ADRs (13_Решения.md)
│
├── 03-features/                       # Feature Specifications
│   ├── drm-spec.md                    # DRM system (09_Спецификация_DRM.md)
│   ├── payments.md                    # Payment flow (06_Платежка.md)
│   ├── financial-ledger.md            # Accounting (17_Финансовый_ledger.md)
│   └── unique-features.md             # Differentiators (15_Уникальные_фичи.md)
│
├── 04-security/                       # Security & Compliance
│   ├── threat-model.md                # Threats (18_Модель_угроз.md)
│   ├── security-overview.md           # Security guide (10_Безопасность.md)
│   ├── audit-2025-01.md               # Full audit (MTA_MARKET_FULL_AUDIT.md)
│   └── requirements.md                # Security reqs (SECURITY_REQUIREMENTS.md)
│
├── 05-operations/                     # DevOps & Operations
│   ├── deployment.md                  # Deploy guide (DEPLOYMENT.md)
│   ├── testing.md                     # Test strategy (11_Тестирование.md)
│   └── monitoring.md                  # Observability (TBD)
│
├── 06-development/                    # Developer Guides
│   ├── getting-started.md             # Quick start (README_DEV.md)
│   ├── contributing.md                # Contribution guide (CONTRIBUTING.md)
│   ├── code-of-conduct.md             # CoC (CODE_OF_CONDUCT.md)
│   └── glossary.md                    # Terms (14_Глоссарий.md)
│
├── 07-planning/                       # Historical Planning Docs
│   ├── detail-plan.md                 # Original plan (04_Detail_Plan.md)
│   ├── information.md                 # Project info (02_Information.md)
│   └── for-humans.md                  # Non-tech overview (05_Для_человека.md)
│
├── 08-reports/                        # Stage Reports
│   ├── stage0-spike.md                # DRM spike (19_STAGE_0_SPIKE_REPORT.md)
│   ├── stage1-completion.md           # Stage 1 (20_STAGE_1_COMPLETION_REPORT.md)
│   ├── stage2-summary.md              # Stage 2 (STAGE2_SUMMARY.md)
│   ├── stage3-summary.md              # Stage 3
│   ├── stage4-summary.md              # Stage 4
│   ├── stage5-summary.md              # Stage 5
│   ├── stage6-summary.md              # Stage 6
│   └── stage7-summary.md              # Stage 7
│
└── 09-legacy/                         # Deprecated/Archive
    ├── old-readme.md                  # Original intro (01_Read_Me.md)
    └── doc-map-v1.md                  # Old structure (00_DOCUMENTATION_MAP.md)
```

## Design Principles

1. **Numeric prefixes** for guaranteed sort order
2. **Semantic grouping** by audience & purpose
3. **Flat within categories** (max 1 level deep)
4. **kebab-case filenames** for consistency
5. **Cross-links** preserved in content
6. **Git history** maintained through copy + delete

## Migration Mapping

| Old Location | New Location | Notes |
|---|---|---|
| `PROJECT_STATUS.md` | `01-project/status.md` | Current status |
| `MTA_MARKET_FULL_AUDIT.md` | `04-security/audit-2025-01.md` | Full audit |
| `document/03_Архитектура.md` | `02-architecture/overview.md` | Translate or keep |
| `document/09_Спецификация_DRM.md` | `03-features/drm-spec.md` | Core feature |
| `DEPLOYMENT.md` | `05-operations/deployment.md` | DevOps |
| `README_DEV.md` | `06-development/getting-started.md` | Dev guide |
| `STAGE*_SUMMARY.md` | `08-reports/stage*-summary.md` | Historical |

## Next Steps

1. Create folder structure
2. Copy files to new locations with `git mv` (preserve history)
3. Update internal cross-references
4. Generate `00-INDEX.md` with navigation
5. Archive original `document/` folder in mta-market-site
