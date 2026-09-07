# Деплой

## Локально / стенд

В `mta-market-site`:

```bash
cp .env.example .env
docker compose up -d
```

- Frontend: http://localhost:3000  
- API: http://localhost:3001  
- Health: http://localhost:3001/health  

`docker-compose.yml` — **стенд**: пароль Postgres `dev_password`, JWT с дефолтом. В бой не копировать.

## Переменные API (минимум)

Обязательны в production: `JWT_SECRET` (≥32), `DATABASE_URL`, `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_REDIRECT_URI`.

По флагу: ЮKassa (`YOOKASSA_ENABLED=true` + shop/secret/notification password), S3 (`S3_ENABLED=true` + bucket/ключи).

Рекомендуются: `REDIS_URL`, `FRONTEND_URL`.

## Прод

`docker-compose.prod.yml` + свои секреты, TLS на nginx, не публиковать Postgres/Redis наружу. Бэкап тома Postgres до первого реального платежа.

Модуль на MTA-сервер покупателя не деплоится этим compose: отдельный `.dll`/`.so` в `modules/` и строка в `mtaserver.conf`.
