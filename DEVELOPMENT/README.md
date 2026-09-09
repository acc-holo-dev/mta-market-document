# Development Plan System

## Основной принцип

Development Plan — это не просто список задач.

**Каждый Plan переводит проект из одного рабочего состояния в другое.**

План описывает:

- цель (какое состояние нужно получить);
- исходное состояние (где проект находится сейчас);
- workstreams (направления работ);
- критерии завершения (когда план считается выполненным — по фактической
  проверке продукта, а не по «код компилируется»);
- итоговую запись о выполнении.

Планы живут в:

- `DEVELOPMENT/ACTIVE/` — активный план (один за раз);
- `DEVELOPMENT/COMPLETED/` — завершённые планы с итоговой записью
  (цель, что сделано, тесты, приёмка, ограничения, дата).

Текущее состояние всегда отражено в [CURRENT.md](CURRENT.md).

## История планов

| План | Статус | Суть |
|---|---|---|
| [PLAN-001](COMPLETED/PLAN-001.md) | COMPLETED (2026-09-10) | Initial Product Release: довести MTA Market до первого цельного рабочего продукта (auth → marketplace → seller → модерация → покупка → лицензия), проверенного реальным browser flow. |

## Правила работы с планами

1. Новый цикл разработки = новый план (`PLAN-002`, ...). Один активный план за раз.
2. План фиксирует переход состояний, а не список желаемых фич: «из состояния A в состояние B».
3. Всё, что не требуется для цели плана, уходит в [IDEAS](../IDEAS/IDEAS.md), а не в план.
4. План считается завершённым только после проверки пользовательского/продуктового
   результата (для продуктовых фич — реальный browser flow, а не только компиляция и
   модульные тесты).
5. После завершения план переносится в `COMPLETED/` с фактическим итогом, без будущих идей.

## Воспроизведение среды разработки и проверки

```sh
# сервисы
docker compose up -d postgres redis        # в mta-market-site

# БД: применить контракт-схему
cd apps/server
npx prisma contract emit && npx prisma db update --confirm default   # DATABASE_URL из .env

# сервер и фронтенд
npx tsx watch src/index.ts                 # API :3001
pnpm --filter @mta-market/web dev          # web :3000

# admin-аккаунт для разработки (G-004; отказ в NODE_ENV=production)
npx tsx scripts/dev-admin.ts --email admin@dev.local --username admin --password 'dev-password-123'
```

Проверка качества:

```sh
pnpm type-check          # оба приложения
pnpm --filter @mta-market/server test    # backend tests (254)
pnpm --filter @mta-market/web build      # production build web
pnpm test:e2e:admin && pnpm test:e2e     # browser E2E (12) — нужны запущенные серверы
```

Модуль (Linux x64): `g++ -std=c++17 -I source source/drm/*.cpp tests_drm/main.cpp -lssl -lcrypto`
→ запустить бинарник, ожидается `ALL TESTS PASSED`.

Переменные окружения: `apps/server/.env.example` (обязательны `DATABASE_URL`, `JWT_SECRET`;
`ARTIFACT_SIGNING_PRIVATE_KEY` — Ed25519 PKCS8 base64 для подписи артефактов;
`YOOKASSA_*` опциональны — иначе dev-completion `POST /payments/:id/simulate`).
