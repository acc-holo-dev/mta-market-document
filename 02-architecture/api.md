# HTTP API

База: `apps/server/src/index.ts`. Префикс без версии. JSON.

Служебные:

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/` | Имя сервиса, список префиксов |
| GET | `/health` | liveness |

## Auth `/auth`

| Метод | Путь | Auth |
|---|---|---|
| GET | `/discord` | нет |
| GET | `/discord/callback` | нет |
| POST | `/refresh` | cookie |
| POST | `/logout` | cookie |
| GET | `/me` | JWT |

## Resources `/resources`

| Метод | Путь | Auth |
|---|---|---|
| GET | `/` | нет |
| GET | `/:slug` | нет |
| POST | `/` | да |
| PATCH | `/:slug` | да |
| DELETE | `/:slug` | да |
| GET | `/:slug/versions` | нет |
| POST | `/:slug/versions` | да |
| GET | `/:slug/versions/:id` | зависит от права на файл |
| GET | `/:slug/reviews` | нет |
| POST | `/:slug/reviews` | да |
| PATCH | `/:slug/reviews/:id` | да |
| DELETE | `/:slug/reviews/:id` | да |

## Purchases `/purchases`

| Метод | Путь | Auth |
|---|---|---|
| POST | `/` | да (тело: `resourceSlug`) |
| GET | `/my` | да |
| GET | `/:id` | да |

Нет `POST /:id/complete` — обход оплаты этим путём убран.

## Payments `/payments`

| Метод | Путь | Auth |
|---|---|---|
| POST | `/create` | да |
| POST | `/webhook` | провайдер |
| POST | `/:id/simulate` | да, **только** `NODE_ENV !== production` |

## DRM `/drm`

| Метод | Путь | Auth |
|---|---|---|
| POST | `/activate` | да |
| POST | `/verify` | нет (ключ в теле) |
| GET | `/my-licenses` | да |
| DELETE | `/revoke/:id` | да |

## Upload `/upload`

Загрузка файла ресурса / превью; local или S3.

## Admin `/admin`

Список ресурсов, смена статуса, пользователи, роль, удаление отзыва, статистика. Роли ADMIN и MODERATOR.

Новый эндпоинт → строка в этой таблице в том же PR.
