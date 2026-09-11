# MTA Market — документация

Карта этого репозитория. Код живёт в трёх репозиториях:

| Репозиторий | Роль |
|---|---|
| [mta-market-site](https://github.com/acc-holo-dev/mta-market-site) | Backend (Express/Postgres) + frontend (Next.js) платформы |
| [mta-market-module](https://github.com/acc-holo-dev/mta-market-module) | Нативный C++ модуль MTA: SDK + DRM client + market integration client |
| **mta-market-document** (этот репозиторий) | Продуктовая документация и Development Plan system |

## Структура

| Путь | Что там |
|---|---|
| [PROJECT.md](PROJECT.md) | **Что такое MTA Market сегодня**: продукт, репозитории, роли, реализованные сценарии, честные границы. Начинать отсюда. |
| [VISION.md](VISION.md) | **Кем MTA Market должен стать**: долгосрочное видение экосистемы (Community + Servers + Content + Market + Trust + Identity). Целевое состояние, не описание текущего кода. |
| [PRODUCT-ARCHITECTURE.md](PRODUCT-ARCHITECTURE.md) | **Из каких частей состоит платформа**: подсистемы, интеграции, границы. |
| [PRODUCT-MODEL.md](PRODUCT-MODEL.md) | **Как живут сущности**: состояния, permissions, visibility, отношения. Канонический список сущностей — здесь. |
| [PRODUCT-SURFACE-MAP.md](PRODUCT-SURFACE-MAP.md) | **Какие пользовательские поверхности существуют**: экраны, URL, роли, потоки. |
| [DAILY-EXPERIENCE.md](DAILY-EXPERIENCE.md) | **Как платформа создаёт причину возвращаться**: живая Home, LIVE-слой, activity как read-layer, петли возврата по ролям (Player / Server Owner / Creator / Member), правила шума, приватности и ranking. |
| [IDEAS/](IDEAS/) | **Идеи на будущее**. Наличие идеи здесь не создаёт обязательство её реализовать. |
| [DEVELOPMENT/](DEVELOPMENT/) | **Development Plan system**: планы-переходы, текущее состояние, завершённые планы, замороженные контракты. |
| [DEVELOPMENT/CURRENT.md](DEVELOPMENT/CURRENT.md) | Текущее состояние продукта и статус планов — начинать отсюда при работе над кодом. |
| [DEVELOPMENT/ACTIVE/](DEVELOPMENT/ACTIVE/) | Активный план (ровно один). Сейчас пуст — PLAN-009 завершён. |
| [DEVELOPMENT/COMPLETED/](DEVELOPMENT/COMPLETED/) | Завершённые планы: спецификация + запись о выполнении (PLAN-001…005). |
| [DEVELOPMENT/REFERENCE/](DEVELOPMENT/REFERENCE/) | Замороженные технические контракты (DRM Protocol v2, контракт репозиториев, матрица совместимости). Код — источник истины при расхождении. |

## Правила

- Единственный источник истины по коду — код и тесты. Документация описывает, но не заменяет.
- Foundational-документы (VISION / ARCHITECTURE / MODEL / SURFACE MAP / DAILY
  EXPERIENCE) описывают **целевое** состояние платформы. Какой subset реализован
  сейчас — см. [PROJECT.md](PROJECT.md) и [DEVELOPMENT/CURRENT.md](DEVELOPMENT/CURRENT.md).
- Действующие development plans не хранятся в корне репозитория: активный план —
  в `DEVELOPMENT/ACTIVE/`, завершённый — в `DEVELOPMENT/COMPLETED/`.
- Исторические документы предыдущих структур каталогов доступны в git-истории
  этого репозитория.
- Подробная техническая энциклопедия (API/DB/Deployment) будет создаваться позже,
  отдельным решением.
