# File Relevance Audit — mta-market-site

**Date:** 2026-09-08
**Scope:** Configuration, source code, and generated contract consistency
**Status:** ⚠️ 1 BLOCKER found (contract regeneration required)

---

## Summary

Проведена проверка актуальности файлов в `mta-market-site` после завершения
TASK-001–TASK-018. Все конфигурационные файлы и исходный код актуальны и
согласованы. Обнаружен **один критический блокер**: сгенерированный Prisma
контракт (`contract.d.ts` / `contract.json`) устарел и не содержит новые модели.

---

## ✅ Проверено и актуально

### Root configuration
| File | Status | Notes |
|---|---|---|
| `.env.example` | ✅ | Все переменные соответствуют `startupValidation.ts` |
| `docker-compose.yml` | ✅ | `YOOKASSA_NOTIFICATION_PASSWORD` (исправлено) |
| `docker-compose.prod.yml` | ✅ | `YOOKASSA_NOTIFICATION_PASSWORD` (исправлено) |
| `package.json` | ✅ | Turbo monorepo, Node >=20, pnpm >=9 |
| `turbo.json` | ✅ | build / dev / lint / type-check |
| `.gitignore` | ✅ | Исключает dist, .next, .turbo, .env, uploads |
| `LICENSE` | ✅ | MIT |
| `nginx.conf` | ✅ | Reverse proxy, SSL, rate limiting |

### Backend (`apps/server`)
| File | Status | Notes |
|---|---|---|
| `package.json` | ✅ | Prisma 8, CLI scripts (artifact, drm, reconciliation) |
| `.env.example` | ✅ | JWT, Discord, YooKassa, S3, Email |
| `tsconfig.json` | ✅ | |
| `prisma.config.ts` | ✅ | |
| `Dockerfile` | ✅ | Production build |
| `src/index.ts` | ✅ | Entry point + startup validation |
| `src/lib/` | ✅ | 19 модулей (security + providers + services) |
| `src/routes/` | ✅ | 9 route files |
| `src/middleware/` | ✅ | validate.ts, validateCuid.ts |
| `src/jobs/` | ✅ | reconciliation.ts |
| `src/cli/` | ✅ | artifact.ts, drm.ts |
| `src/tests/` | ✅ | 4 test files |

### Frontend (`apps/web`)
| File | Status | Notes |
|---|---|---|
| `package.json` | ✅ | Next.js 15, React 19, TailwindCSS |
| `.env.example` | ✅ | `NEXT_PUBLIC_API_URL` |
| `next.config.ts` | ✅ | |
| `tailwind.config.js` | ✅ | |
| `postcss.config.js` | ✅ | |
| `tsconfig.json` | ✅ | |

### Packages / Scripts / CI
- `packages/tsconfig/` ✅
- `packages/eslint-config/` ✅
- `scripts/backup.sh`, `scripts/deploy.sh` ✅
- `.github/workflows/ci.yml` ✅
- `.github/ISSUE_TEMPLATE/` ✅

---

## 🔴 BLOCKER: Устаревший сгенерированный контракт

### Проблема

`contract.prisma` (source of truth) обновлён и содержит новые модели, но
сгенерированные файлы **не перегенерированы**:

| File | Modified | Contains new models? |
|---|---|---|
| `src/prisma/contract.prisma` | 08.09.2026 01:45 | ✅ Yes (source) |
| `src/prisma/contract.d.ts` | 07.09.2026 05:48 | ❌ **No** |
| `src/prisma/contract.json` | 07.09.2026 05:48 | ❌ **No** |

### Отсутствующие модели в контракте

Проверено поиском в `contract.d.ts` / `contract.json`:

- ❌ `Order`
- ❌ `OrderItem`
- ❌ `Service`
- ❌ `ServiceOrderItem`
- ❌ `ServicePurchase`
- ❌ `Discount`
- ❌ `ReconciliationReport`
- ❌ `ReconciliationMismatch`
- ❌ `ArtifactSignature`
- ❌ `SandboxRun`

### Последствия

Код, использующий новые модели, **не скомпилируется** (`tsc --noEmit` упадёт):

- `src/lib/order.ts` → `db.orm.public.Order`
- `src/lib/discount.ts` → `db.orm.public.Discount`
- `src/lib/reconciliation/service.ts` → `prisma.reconciliationReport`
- `src/lib/prisma.ts` (wrapper) → `db.orm.public.Order/Service/Discount/...`
- `src/cli/artifact.ts`, `src/cli/drm.ts` → новые модели

### Решение

Перегенерировать контракт из обновлённой схемы:

```bash
pnpm db:emit
# эквивалентно:
# pnpm --filter @mta-market/server contract:emit
# → prisma contract emit
```

**Требуется:** Node.js + pnpm окружение (в текущей среде `npx`/`pnpm` недоступны).

### После регенерации

1. Проверить что `contract.d.ts` содержит все новые модели
2. Запустить `pnpm type-check` — должен пройти
3. Запустить `pnpm build`
4. Применить миграцию БД (см. TASK-003)

---

## 🔧 Исправления, внесённые при аудите

### 1. Синхронизация env var YooKassa (TASK-005)

Код (`startupValidation.ts`, `yookassaWebhook.ts`) использует
`YOOKASSA_NOTIFICATION_PASSWORD`, но примеры и compose-файлы содержали
устаревший `YOOKASSA_WEBHOOK_SECRET`.

**Исправлено:**
- `.env.example` → `YOOKASSA_NOTIFICATION_PASSWORD`
- `apps/server/.env.example` → `YOOKASSA_NOTIFICATION_PASSWORD`
- `docker-compose.yml` → `YOOKASSA_NOTIFICATION_PASSWORD`
- `docker-compose.prod.yml` → `YOOKASSA_NOTIFICATION_PASSWORD`

### 2. Дублирование `PurchaseStatus` в схеме

В `contract.prisma` было два одинаковых enum `PurchaseStatus`. Удалён дубль,
оставлен один актуальный (с `DISPUTED`):

```prisma
enum PurchaseStatus {
  PENDING
  COMPLETED
  REFUNDED
  FAILED
  DISPUTED
}
```

### 3. Обновлён `prisma.ts` wrapper

Добавлены новые модели в compatibility wrapper:
`Discount`, `Order`, `OrderItem`, `Service`, `ServiceOrderItem`, `ServicePurchase`.

### 4. Реорганизация reconciliation модуля

TASK-018 уже реализовал provider reconciliation (`service.ts`, `types.ts`).
Мой файл внутренней сверки был перемешён:

- `src/lib/reconciliation.ts` → `src/lib/reconciliation/internal.ts`
- Добавлен экспорт функций `internal` в `index.ts`
  (`reconcileAllPurchases`, `isReconciled`, `getReconciliationStatus`)

Теперь модуль имеет правильную структуру:
```
lib/reconciliation/
├── index.ts       — публичные экспорты
├── internal.ts    — внутренняя сверка (seller balances vs purchases)
├── service.ts     — сверка с провайдером (TASK-018)
└── types.ts       — типы
```

---

## Итоговая оценка

| Категория | Статус |
|---|---|
| Конфигурация | ✅ Актуальна |
| Исходный код | ✅ Актуален |
| Схема БД (`contract.prisma`) | ✅ Актуальна |
| Сгенерированный контракт | 🔴 **Устарел — требуется `pnpm db:emit`** |
| Документация | ✅ Перенесена в `mta-market-document` |

**Готовность:** код и конфигурация готовы; для сборки требуется регенерация
Prisma-контракта в окружении с Node.js/pnpm.

---

## Следующие шаги

1. В окружении с Node.js выполнить `pnpm install && pnpm db:emit`
2. Запустить `pnpm type-check` и `pnpm build`
3. Применить миграцию БД (TASK-003 verification)
4. Продолжить TASK-019+ (artifact signing уже частично есть в `src/cli/artifact.ts`)
