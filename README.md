# MTA Market — документация

**Статус:** MVP / не готов к продакшену  
**Язык канона:** русский  
**Обновлено:** 2026-09-07

Единственный источник правды по продукту. Код живёт в двух репозиториях:

| Репозиторий | Роль |
|---|---|
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | Backend + frontend |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | Нативный модуль MTA (SDK + DRM-spike) |

Карта документов: [00-INDEX.md](00-INDEX.md).  
Текущее состояние: [01-project/status.md](01-project/status.md).

## Что это

Площадка продажи серверных ресурсов для MTA:SA. Покупка ≠ лицензия ≠ платёж. DRM заявлен как cost-raising (массовое копирование усложняется), не как абсолютная защита.

## Честные границы

- Сайт — рабочий MVP: Discord OAuth, каталог, покупки, ЮKassa-скелет, админка.
- Модуль — зрелый C++ SDK; DRM — spike (`spike_drm_load`), не протокол v2.
- Деньги, sandbox загрузок и асимметричный DRM в прод не выводить.
