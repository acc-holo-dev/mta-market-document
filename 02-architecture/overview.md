# Обзор архитектуры

```
Браузер / MTA-сервер покупателя
        │ HTTPS
        ▼
   Nginx (прод: TLS, лимиты)
        │
   ┌────┴────┐
   ▼         ▼
Next.js    Express API
(apps/web) (apps/server)
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
  PostgreSQL Redis  S3/R2
```

Монолит с модулями внутри одного процесса API. Не микросервисы.

## Стек (факт)

- Node.js 20+ (в README сайта фигурирует 24), TypeScript, pnpm, Turbo.
- Express, Prisma 8 (`contract.prisma`).
- PostgreSQL 16, Redis 7.
- Next.js 15 + React 19 + Tailwind.
- Discord OAuth2, ЮKassa, опционально S3.

## Границы модулей API

`auth`, `resources`, `versions`, `reviews`, `purchases`, `payments`, `upload`, `drm`, `admin`.

Нативный модуль MTA — отдельный процесс на сервере покупателя. С license server говорит по HTTP; протокол v2 ещё не в коде.

## Что сознательно не делаем

Отдельный License Server как сервис, очередь на каждый вебхук, Elasticsearch. Когда нагрузка появится — выделять по факту, не заранее.
