status: current
version: 1.1
last_verified: 2026-09-09

# Оглавление

Читать в этом порядке. Других markdown-канонов в проекте нет.

## Проект

| Файл | Содержание |
|---|---|
| [README.md](README.md) | Зачем этот репозиторий |
| [01-project/status.md](01-project/status.md) | Что работает в коде сегодня |
| [01-project/production-readiness.md](01-project/production-readiness.md) | Гейты до публичного запуска |
| [01-project/repository-map.md](01-project/repository-map.md) | Границы трёх репозиториев |
| [01-project/roadmap.md](01-project/roadmap.md) | Что дальше |

## Архитектура

| Файл | Содержание |
|---|---|
| [02-architecture/overview.md](02-architecture/overview.md) | Слои системы |
| [02-architecture/architecture.md](02-architecture/architecture.md) | Три репозитория, runtime-границы (P-004) |
| [02-architecture/repository-contract.md](02-architecture/repository-contract.md) | Контракт репозиториев: владение, релизы, совместимость (P-012) |
| [02-architecture/compatibility-matrix.md](02-architecture/compatibility-matrix.md) | Матрица совместимости site / API / DRM / module (P-013) |
| [02-architecture/domain-model.md](02-architecture/domain-model.md) | Сущности и инварианты |
| [02-architecture/database.md](02-architecture/database.md) | Схема Prisma |
| [02-architecture/api.md](02-architecture/api.md) | HTTP API |
| [02-architecture/decisions.md](02-architecture/decisions.md) | ADR |

## Возможности

| Файл | Содержание |
|---|---|
| [03-features/drm/protocol-v2.md](03-features/drm/protocol-v2.md) | DRM Protocol v2 — замороженный контракт (P-008) |
| [03-features/drm.md](03-features/drm.md) | DRM: цель v2 vs реализация v1 |
| [03-features/artifacts/format.md](03-features/artifacts/format.md) | Формат артефакта: manifest v1 (P-009) |
| [03-features/artifacts/signing.md](03-features/artifacts/signing.md) | Подпись артефактов (Ed25519) (P-009) |
| [03-features/artifacts/compatibility.md](03-features/artifacts/compatibility.md) | Совместимость и граф зависимостей (P-009) |
| [03-features/payments/overview.md](03-features/payments/overview.md) | Платежи: архитектура и поток (P-010) |
| [03-features/payments/state-machine.md](03-features/payments/state-machine.md) | Машина состояний платежа (P-010) |
| [03-features/payments/yookassa.md](03-features/payments/yookassa.md) | Провайдер ЮKassa, верификация вебхука (P-010) |
| [03-features/payments/refunds.md](03-features/payments/refunds.md) | Возвраты: INV-013, K-004 (P-010) |
| [03-features/payments/reconciliation.md](03-features/payments/reconciliation.md) | Сверка финансов (P-010) |
| [03-features/payments.md](03-features/payments.md) | Платежи и ЮKassa (краткий обзор) |
| [03-features/ledger.md](03-features/ledger.md) | Баланс продавца |

## Безопасность и эксплуатация

| Файл | Содержание |
|---|---|
| [04-security/overview.md](04-security/overview.md) | Что уже есть в коде |
| [04-security/threats.md](04-security/threats.md) | Модель угроз (качественная) |
| [04-security/threat-model-linkage.md](04-security/threat-model-linkage.md) | Угроза → требование → код → тест → статус (P-011) |
| [05-operations/deployment.md](05-operations/deployment.md) | Docker и окружение |
| [05-operations/backup.md](05-operations/backup.md) | Бэкапы: RPO/RTO, восстановление (O-003) |
| [05-operations/secrets.md](05-operations/secrets.md) | Секреты и ключи: владение, ротация (O-004) |
| [05-operations/release-channels.md](05-operations/release-channels.md) | Каналы релизов (post-MVP), YANK (R-003) |
| [05-operations/testing.md](05-operations/testing.md) | Как проверяем |
| [09-legacy/README.md](09-legacy/README.md) | Политика исторических документов (P-005/P-006) |

## Разработка

| Файл | Содержание |
|---|---|
| [06-development/getting-started.md](06-development/getting-started.md) | Локальный запуск |
| [06-development/contributing.md](06-development/contributing.md) | Как менять код и документы |
| [06-development/glossary.md](06-development/glossary.md) | Термины |

## Правило

Спецификация меняется **здесь**. README в `site` и `module` только запускают проект и ссылаются сюда.
