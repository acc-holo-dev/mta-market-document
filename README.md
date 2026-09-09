status: current
version: 1.1
last_verified: 2026-09-09

# MTA Market — документация

**Статус:** MVP / не готов к продакшену  
**Язык канона:** русский (спецификации — английский)  
**Обновлено:** 2026-09-09

Единственный источник правды по продукту. Код живёт в двух репозиториях:

| Репозиторий | Роль |
|---|---|
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | Backend + frontend |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | Нативный модуль MTA (SDK + DRM client subsystem) |

Карта документов: [00-INDEX.md](00-INDEX.md).  
Текущее состояние: [01-project/status.md](01-project/status.md).

## Что это

Площадка продажи серверных ресурсов для MTA:SA. Покупка ≠ лицензия ≠ платёж. DRM заявлен как cost-raising (массовое копирование усложняется), не как абсолютная защита.

## Честные границы

- Сайт — рабочий MVP: OAuth (Discord/Yandex/Google), каталог, покупки, платежи (ЮKassa) с машиной состояний и возвратами, двойная запись ledger, DRM v2 на стороне сервера, песочница загрузок, 222 теста (2026-09-09). Не продакшен: нет E2E в браузере, нет CI-гейтов тестов/секретов, нет restore-drill.
- Модуль — зрелый C++ SDK + DRM client subsystem протокола v2 (`source/drm/`), юнит-тесты проходят (Linux x64). E2E против живого сервера и Windows-исполнение ещё не проверены.
- Деньги и асимметричный DRM в прод не выводить, пока не закрыты гейты [01-project/production-readiness.md](01-project/production-readiness.md).
