# MTA Market — документация

Карта этого репозитория. Код живёт в трёх репозиториях:

| Репозиторий | Роль |
|---|---|
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | Backend (Express/Postgres) + frontend (Next.js) маркетплейса |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | Нативный C++ модуль MTA: SDK + DRM client subsystem |
| **mta-market-document** (этот репозиторий) | Продуктовая документация и Development Plan system |

## Структура

| Путь | Что там |
|---|---|
| [PROJECT.md](PROJECT.md) | **Что такое MTA Market**: продукт, репозитории, роли (buyer/seller/admin), реализованные сценарии, что считается Initial Product Release. Не roadmap и не task list. |
| [IDEAS/](IDEAS/) | **Идеи на будущее**. Наличие идеи здесь не создаёт обязательство её реализовать. |
| [DEVELOPMENT/](DEVELOPMENT/) | **Development Plan system**: планы-переходы, текущее состояние, завершённые планы, замороженные контракты. |
| [DEVELOPMENT/CURRENT.md](DEVELOPMENT/CURRENT.md) | Текущий активный план и состояние продукта — начинать отсюда. |
| [DEVELOPMENT/COMPLETED/PLAN-001.md](DEVELOPMENT/COMPLETED/PLAN-001.md) | Итоговая запись PLAN-001 (Initial Product Release), приёмка 2026-09-10. |
| [DEVELOPMENT/REFERENCE/](DEVELOPMENT/REFERENCE/) | Замороженные технические контракты (DRM Protocol v2, контракт репозиториев, матрица совместимости). Код — источник истины при расхождении. |

## Правила

- Единственный источник истины по коду — код и тесты. Документация описывает, но не заменяет.
- Исторические документы предыдущей структуры (каталоги `01…09`) выведены из активного
  использования и удалены; при необходимости они доступны в git-истории этого репозитория
  (состояние до коммита реструктуризации).
- Подробная техническая энциклопедия (API/DB/Deployment) будет создаваться позже, отдельным решением.
