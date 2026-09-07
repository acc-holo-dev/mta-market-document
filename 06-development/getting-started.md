# Быстрый старт

Нужны: Docker, Node 20+, pnpm 9+.

## Сайт

```bash
git clone https://github.com/acc-holo-dev/mta-market-site
cd mta-market-site
cp .env.example .env
# DISCORD_CLIENT_ID / SECRET, JWT_SECRET
docker compose up -d postgres redis
pnpm install
pnpm --filter @mta-market/server contract:emit
pnpm dev
```

Либо `docker compose up -d` со сборкой backend/frontend.

## Модуль

CMake 3.27+, Ninja, компилятор (MinGW/MSVC/GCC), Python 3.11+ для CLI `mta`.

```bash
git clone https://github.com/acc-holo-dev/mta-market-module
cd mta-market-module
# см. other/tools — CLI
mta doctor
mta build
mta test
```

Справка SDK (EN): `other/documents/`. Протокол маркета — этот репозиторий, [03-features/drm.md](../03-features/drm.md).
