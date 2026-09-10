PLAN-004 — Production Readiness & Operational Hardening

Контекст проекта:

Репозитории:

- acc-holo-dev/mta-market-site
- acc-holo-dev/mta-market-module
- acc-holo-dev/mta-market-document

Завершённые планы:

PLAN-001 — Initial Product Release
PLAN-002 — Product Experience Foundation
PLAN-003 — Marketplace Core

Сейчас MTA Market уже является работающим marketplace-продуктом:

- authentication;
- Discord;
- username/email/password;
- profile;
- user balance;
- Marketplace;
- search;
- categories;
- filters;
- sorting;
- resource covers;
- screenshots;
- resource detail;
- seller storefront;
- seller creation wizard;
- artifact upload;
- moderation;
- admin panel;
- purchases;
- licenses;
- payments;
- reviews;
- versions;
- DRM v2;
- mta-market-module;
- responsive frontend;
- browser E2E.

Главная проблема теперь не в product features.

Главная проблема:

MTA Market должен быть подготовлен к реальному production deployment.

==================================================
1. ГЛАВНАЯ ЦЕЛЬ
==================================================

PLAN-004 должен перевести проект из:

"working development product"

в:

"production-ready service".

Цель:

Реальный сервер должен быть способен работать с настоящими пользователями, настоящими деньгами и настоящими ресурсами без зависимости от ручных dev-only процедур.

После завершения PLAN-004 должно быть возможно:

1. развернуть MTA Market на production infrastructure;
2. использовать production PostgreSQL;
3. использовать durable media/object storage;
4. принимать реальные платежи;
5. принимать реальные payment webhooks;
6. выдавать реальные licenses;
7. работать с DRM v2;
8. видеть состояние системы;
9. получать уведомления о критических проблемах;
10. восстановить систему после критического сбоя;
11. выполнить rollback deployment;
12. контролируемо обновлять database schema;
13. безопасно хранить production secrets;
14. проверить production security configuration;
15. понимать, что происходит с системой после deployment.

Главный критерий:

Если завтра появятся настоящие пользователи, система должна быть готова их обслуживать.

==================================================
2. ЖЁСТКИЕ ПРИНЦИПЫ
==================================================

2.1. Не добавлять новые marketplace features.

2.2. Не начинать PLAN-005.

2.3. Не делать redesign frontend.

2.4. Не менять DRM protocol v2 без критической необходимости.

2.5. Не переписывать backend без причины.

2.6. Не использовать dev simulation как доказательство production readiness.

2.7. Не считать production-ready то, что работает только локально.

2.8. Не использовать local filesystem как единственный durable production storage.

2.9. Не оставлять production secrets в repository.

2.10. Каждое production-critical утверждение должно иметь подтверждение:
- code;
- test;
- deployment procedure;
- real environment verification.

==================================================
3. ПОКРЫТИЕ
==================================================

Основной repository:

mta-market-site

Проверяемый repository:

mta-market-module

Documentation:

mta-market-document

Не менять module без необходимости.

==================================================
4. WORKSTREAM A — PRODUCTION CONFIGURATION
==================================================

A-001 — Production environment matrix

Составить полный список production environment variables:

- DATABASE_URL;
- REDIS_URL;
- JWT_SECRET;
- OAuth secrets;
- DRM_SERVER_PRIVATE_KEY;
- ARTIFACT_SIGNING_PRIVATE_KEY;
- S3 credentials;
- YooKassa credentials;
- frontend/backend URLs;
- SMTP/email;
- rate limits;
- storage configuration;
- monitoring configuration;
- any other runtime secrets.

Разделить:

DEVELOPMENT
STAGING
PRODUCTION

Не допускать, чтобы dev values случайно использовались в production.

---

A-002 — Production startup validation

Startup validation должна проверять все production-critical параметры.

Если конфигурация небезопасна:

server должен fail fast.

Не разрешать запуск production с:

- weak JWT secret;
- отсутствующим DRM key;
- отсутствующим artifact key;
- отсутствующим database;
- включённым production storage without credentials;
- enabled payments without credentials.

---

A-003 — Rate limits

Проверить production limits отдельно от E2E/dev.

Не переносить большие E2E values:

AUTH/STANDARD/LOGIN/REFRESH

в production.

Создать понятные production defaults.

---

A-004 — Trust proxy

Проверить:

TRUST_PROXY

и реальную topology:

Browser
→ Nginx
→ Backend.

Убедиться, что req.ip и rate limiting определяют реальный client IP.

---

A-005 — Cookie production configuration

Проверить:

- Secure;
- HttpOnly;
- SameSite;
- domain;
- path;
- expiration.

Убедиться, что refresh cookie безопасен в production HTTPS.

==================================================
5. WORKSTREAM B — MEDIA / OBJECT STORAGE
==================================================

B-001 — Fix S3 media contract

Критический gap:

Сейчас `/upload/media` при S3 возвращает абсолютный S3 public URL, а resource media attachment ожидает `/media/...`.

Это нужно исправить.

Нельзя решать это разрешением произвольных external URLs.

Нужен единый storage abstraction:

Upload
→ internal object key
→ database reference
→ controlled public delivery URL

---

B-002 — Public media delivery

Определить корректную production media delivery strategy:

Local:
GET /media/:name

S3/R2:
controlled public media URL или CDN/object URL.

При любом варианте:

- arbitrary external URL запрещён;
- paid artifacts не должны быть public;
- media и artifacts должны быть различимы.

---

B-003 — Durable storage

Production media и artifacts не должны зависеть от:

container filesystem.

Использовать:

S3 / Cloudflare R2 / совместимый object storage.

---

B-004 — Artifact privacy

Проверить, что:

paid artifact
→ private storage
→ authorized download
→ short-lived signed URL.

Прямой public object access запрещён.

---

B-005 — Verify Nginx /uploads exposure

Обязательно проверить:

GET /uploads/<artifact>

Не должен позволять скачать защищённый paid artifact напрямую.

Если public `/uploads/` создаёт bypass:

удалить или закрыть этот route.

Это P0 security check.

---

B-006 — Cleanup

Проверить:

- cover replacement;
- screenshot deletion;
- resource deletion;
- abandoned uploads;
- failed uploads.

Не допускать бесконтрольного роста orphaned objects.

---

B-007 — Storage lifecycle

Определить lifecycle policy для object storage.

Минимально:

- active resources;
- old versions;
- deleted resources;
- temporary files.

Не удалять объекты, которые ещё нужны существующим licenses.

==================================================
6. WORKSTREAM C — DATABASE & MIGRATIONS
==================================================

C-001 — Production migration strategy

Определить единый production process:

backup
→ migration
→ verification
→ application deployment.

Нельзя обновлять production schema вручную без documented procedure.

---

C-002 — Migration command

Проверить текущий Prisma tooling и определить безопасную команду для production migrations.

Не использовать development-only schema push как production migration mechanism.

---

C-003 — Migration rollback strategy

Для каждой потенциально breaking migration определить:

- forward migration;
- backward recovery;
- data backup;
- application compatibility.

---

C-004 — Database backup

Настроить documented backup policy.

Минимально определить:

- frequency;
- retention;
- storage;
- encryption;
- access;
- restore procedure.

---

C-005 — Restore drill

Не считать backup рабочим, пока restore не проверен.

Выполнить:

production-like DB
→ backup
→ restore
→ verify application.

Не делать destructive action над реальной production DB для теста.

Использовать staging/test clone.

---

C-006 — Connection limits

Проверить:

- Postgres max connections;
- backend connection behavior;
- Prisma/client pooling;
- concurrency.

==================================================
7. WORKSTREAM D — PAYMENT PRODUCTION
==================================================

D-001 — Real YooKassa configuration

Проверить production:

- shop id;
- secret;
- notification password;
- callback/webhook endpoint;
- HTTPS;
- signature/authentication requirements.

---

D-002 — Real webhook

Симулятор больше не является доказательством.

Нужно проверить настоящий webhook lifecycle в staging/production-like environment.

Сценарий:

create payment
→ provider
→ webhook
→ verification
→ payment state
→ purchase
→ license.

---

D-003 — Idempotency

Проверить повторную доставку одного webhook.

Expected:

одна бизнес-операция.

Не:

двойной purchase;
двойной balance movement;
двойной entitlement.

---

D-004 — Failed payment

Проверить:

created
pending
failed
cancelled
expired

только в пределах существующей payment state machine.

---

D-005 — Refund

Проверить существующий refund flow.

---

D-006 — Reconciliation

Проверить scheduler/job в production.

Убедиться, что он:

- запускается;
- логируется;
- не создаёт duplicate ledger entries;
- может быть повторно запущен;
- обнаруживает discrepancies.

---

D-007 — Real money verification

Не использовать настоящие деньги без безопасного контролируемого сценария.

Если provider предоставляет test/sandbox environment, сначала пройти его полностью.

После этого отдельно документировать remaining production verification.

==================================================
8. WORKSTREAM E — BALANCE / LEDGER
==================================================

E-001 — Balance invariants

Проверить:

balance available
↔ ledger
↔ purchases
↔ refunds
↔ payouts

не расходятся.

---

E-002 — Atomic financial operations

Проверить transaction boundaries.

Например:

payment completed
→ purchase completed
→ license active

не должно оставлять систему в частично завершённом состоянии.

---

E-003 — Duplicate protection

Повторная обработка:

payment
webhook
reconciliation

не должна дважды изменить money state.

---

E-004 — Precision

Проверить, что деньги хранятся в integer minor units / существующем безопасном формате.

Не использовать floating point для финансовых расчётов.

==================================================
9. WORKSTREAM F — DRM LIVE INTEGRATION
==================================================

F-001 — Live DRM server flow

Проверить реальный:

installation
→ challenge
→ activate
→ lease
→ artifact binding
→ DEK release.

Не менять frozen DRM protocol v2.

---

F-002 — Client integration

Использовать существующий mta-market-module.

Проверить взаимодействие с live/staging license server.

---

F-003 — Windows verification

Провести реальную проверку module на Windows.

Linux unit tests недостаточны.

---

F-004 — Failure scenarios

Проверить:

- invalid lease;
- expired lease;
- wrong installation key;
- wrong artifact hash;
- revoked license;
- server unavailable;
- invalid signature.

---

F-005 — Restore scenario

Проверить, что восстановление server/DB не ломает корректные active installations.

==================================================
10. WORKSTREAM G — SECURITY HARDENING
==================================================

G-001 — Secrets

Проверить:

- git history;
- repository;
- Docker layers;
- logs;
- CI output.

Никаких:

JWT secrets;
OAuth secrets;
payment credentials;
DRM private keys;
S3 secrets

в repository.

---

G-002 — Dependency audit

CI сейчас содержит:

pnpm audit --prod --audit-level high || true

Проверить это.

Production security pipeline не должен автоматически игнорировать high/critical vulnerabilities.

Нужно определить policy:

- block on critical;
- block on high;
- approved exceptions.

---

G-003 — CSP

Проверить текущую CSP.

Особенно:

unsafe-inline
unsafe-eval

Сделать максимально строгую policy, совместимую с приложением.

Не ломать legitimate frontend behavior.

---

G-004 — Security headers

Проверить:

- CSP;
- X-Content-Type-Options;
- X-Frame-Options / frame-ancestors;
- Referrer-Policy;
- HSTS;
- Permissions-Policy.

---

G-005 — CORS

В production same-origin topology должна быть корректно настроена.

Не использовать wildcard origins с credentials.

---

G-006 — Direct storage access

Проверить все:

- /uploads;
- /media;
- S3 public endpoints;
- artifact download routes.

Никаких bypass.

---

G-007 — Authorization audit

Проверить critical routes:

- purchases;
- licenses;
- resource editing;
- media;
- seller;
- admin;
- disputes;
- payments.

Frontend visibility не считать authorization.

---

G-008 — Rate-limit abuse

Проверить:

- login;
- registration;
- refresh;
- upload;
- payment;
- admin actions.

==================================================
11. WORKSTREAM H — NGINX / HTTPS
==================================================

H-001 — Domain

Заменить generic:

server_name _

на корректный production domain configuration.

---

H-002 — TLS

Проверить:

- certificate;
- private key permissions;
- TLS 1.2/1.3;
- certificate renewal;
- expiration monitoring.

---

H-003 — HSTS

После подтверждения HTTPS:

Strict-Transport-Security

с корректным max-age.

Не включать preload необдуманно.

---

H-004 — HTTP redirect

HTTP
→ HTTPS

проверить в реальной topology.

---

H-005 — Websocket / upgrade

Проверить only if existing functionality requires it.

Не сохранять лишние proxy Upgrade directives без причины.

---

H-006 — Request limits

Согласовать:

Nginx upload limit
backend multer limit
application size limit.

Не должно быть конфликтующих limits.

==================================================
12. WORKSTREAM I — OBSERVABILITY
==================================================

I-001 — Logging

Проверить structured logs:

- request id;
- timestamp;
- level;
- route;
- duration;
- status;
- user context where safe.

Не логировать:

- passwords;
- tokens;
- private keys;
- payment secrets;
- full sensitive payloads.

---

I-002 — Metrics

Использовать существующие `/metrics`.

Минимально иметь наблюдаемость для:

- request count;
- request latency;
- errors;
- auth failures;
- payment failures;
- uploads;
- moderation;
- DB;
- job/reconciliation.

---

I-003 — Readiness

Проверить:

/live
/ready
/health

Разделение должно реально соответствовать deployment semantics.

---

I-004 — External uptime

Добавить внешний uptime monitoring для:

HTTPS application
API health/readiness.

---

I-005 — Alerts

Определить alerts минимум на:

- application down;
- health check failing;
- high error rate;
- payment failures;
- reconciliation failure;
- DB unavailable;
- storage failure;
- certificate expiration.

==================================================
13. WORKSTREAM J — JOBS / BACKGROUND PROCESSING
==================================================

J-001 — Reconciliation scheduler

Проверить:

- start;
- interval;
- stop;
- failure;
- retry;
- duplicate execution.

---

J-002 — Job visibility

Ошибки scheduled jobs должны попадать в logs/metrics.

---

J-003 — Graceful shutdown

Проверить:

SIGTERM
→ stop scheduler
→ finish critical work
→ close HTTP
→ exit.

Existing shutdown logic already exists; проверить реальный behavior.

==================================================
14. WORKSTREAM K — CI/CD
==================================================

K-001 — CI must represent reality

Проверить:

- lint;
- typecheck;
- tests;
- build;
- security;
- Docker.

---

K-002 — Security gate

Убрать unconditional:

|| true

для production-blocking security vulnerabilities, либо формализовать exception mechanism.

---

K-003 — Image provenance

Docker images должны однозначно соответствовать commit SHA.

---

K-004 — Deployment versioning

Не полагаться исключительно на mutable:

latest

для rollback.

Production deployment должен позволять:

deploy exact image
→ verify
→ rollback exact image.

---

K-005 — Rollback

Документировать:

current version
→ previous version.

Проверить rollback в staging.

---

K-006 — Health-gated deployment

Не считать deployment успешным, пока:

- backend ready;
- frontend healthy;
- database compatible;
- reverse proxy healthy.

==================================================
15. WORKSTREAM L — DOCKER / RUNTIME
==================================================

L-001 — Non-root

Backend runner уже использует non-root user.

Проверить frontend аналогично.

---

L-002 — File permissions

Проверить writable directories:

uploads
temporary files
logs where relevant.

---

L-003 — Container resources

Определить разумные:

- memory limits;
- CPU limits;
- restart policy.

Не выставлять произвольные значения без проверки поведения.

---

L-004 — Health checks

Проверить, что healthchecks действительно отражают состояние service.

---

L-005 — Persistent volumes

Проверить:

Postgres
Redis
media/object storage.

Не допускать потери данных после container recreation.

==================================================
16. WORKSTREAM M — REDIS
==================================================

M-001 — Production Redis

Проверить production configuration.

---

M-002 — Failure semantics

Принять окончательное решение:

Redis outage

должен:

- fail closed для security-critical rate limits;
- или иметь другое explicitly justified behavior.

Нельзя оставлять это случайным.

---

M-003 — Persistence

Если Redis содержит только cache/rate-limit state:

document this.

Если есть critical state:

обеспечить persistence.

==================================================
17. WORKSTREAM N — EMAIL
==================================================

N-001 — Production SMTP

Проверить:

- SMTP;
- sender;
- TLS;
- credentials;
- DNS.

---

N-002 — Email failure

Email provider outage не должен ломать основной marketplace flow, если email не является critical dependency.

---

N-003 — User-facing emails

Проверить existing flows:

- welcome;
- password recovery, если существует;
- payment notifications, если существуют.

==================================================
18. WORKSTREAM O — PASSWORD SECURITY
==================================================

O-001 — Password recovery

Проверить, существует ли полноценный password reset.

Если отсутствует и production account model требует его:

реализовать минимальный безопасный reset flow.

Не хранить reset tokens в plaintext.

---

O-002 — Password policy

Проверить:

- minimum length;
- breached/common password considerations if feasible;
- bcrypt configuration.

---

O-003 — Session invalidation

Проверить:

password change
→ existing refresh sessions.

Определить, должны ли они быть invalidated.

==================================================
19. WORKSTREAM P — DATA RETENTION / PRIVACY
==================================================

P-001 — Sensitive data inventory

Определить:

- passwords;
- email;
- IP;
- user-agent;
- payment metadata;
- OAuth identities;
- licenses;
- audit events.

---

P-002 — Logging retention

Не хранить sensitive operational logs бесконечно.

---

P-003 — User deletion

Проверить существующую user deletion/account lifecycle.

Не удалять данные, которые юридически/business required.

Не придумывать сложную privacy platform без необходимости.

==================================================
20. WORKSTREAM Q — BACKUP & DISASTER RECOVERY
==================================================

Q-001 — Backup policy

Документировать:

WHAT
WHEN
WHERE
HOW LONG
WHO CAN ACCESS

---

Q-002 — Database restore

Проверить реальный restore.

---

Q-003 — Object storage recovery

Проверить:

- media;
- artifacts.

---

Q-004 — Disaster scenario

Смоделировать:

server lost
→ new server
→ restore DB
→ restore object storage access
→ deploy exact application version
→ service healthy.

---

Q-005 — RTO/RPO

Установить реальные target values для первой production версии.

Не придумывать enterprise SLA.

Нужны реалистичные значения для проекта.

==================================================
21. WORKSTREAM R — SCALABILITY BASELINE
==================================================

R-001 — Current bottlenecks

Проверить текущие:

- search;
- popularity;
- rating;
- homepage;
- uploads;
- database.

---

R-002 — SQL aggregation

Убрать production dependence от:

rating/popular first 1000 records in memory.

Перенести на SQL aggregation, если это уже необходимо для expected production scale.

---

R-003 — Search

Оценить текущий ILIKE search.

Для первой production версии разрешается оставить PostgreSQL search, если benchmark показывает достаточную производительность.

Не внедрять Meilisearch автоматически.

---

R-004 — Basic load test

Провести простой load test:

- anonymous Marketplace;
- resource detail;
- login;
- authenticated requests.

Не нужен гигантский performance engineering project.

Главное — узнать текущие limits.

==================================================
22. WORKSTREAM S — PRODUCTION DOMAIN / DEPLOYMENT
==================================================

S-001 — Real domain

Определить:

- production domain;
- API topology;
- HTTPS;
- OAuth callback;
- YooKassa webhook;
- media URLs.

---

S-002 — DNS

Проверить:

A/AAAA
CNAME
www/root
any needed subdomains.

---

S-003 — OAuth callbacks

Проверить Discord callback на production URL.

---

S-004 — Payment callback

Проверить production webhook URL.

---

S-005 — CORS / same-origin

Проверить весь browser flow через реальный домен.

---

S-006 — Cookie domain

Проверить refresh cookie через real HTTPS origin.

==================================================
23. WORKSTREAM T — FINAL PRODUCTION RUNBOOK
==================================================

Создать короткий executable runbook.

Должно быть понятно:

Как:

1. подготовить server;
2. настроить secrets;
3. настроить database;
4. настроить Redis;
5. настроить S3/R2;
6. выполнить migration;
7. deploy backend;
8. deploy frontend;
9. запустить Nginx;
10. проверить health;
11. проверить OAuth;
12. проверить payments;
13. проверить media;
14. проверить DRM;
15. проверить rollback.

Не делать огромную энциклопедию.

Это operational document.

==================================================
24. WORKSTREAM U — STAGING
==================================================

U-001 — Production-like staging

Создать staging topology максимально близкую к production:

Nginx
+
HTTPS
+
Frontend
+
Backend
+
Postgres
+
Redis
+
S3/R2
+
YooKassa sandbox
+
DRM server.

---

U-002 — Staging acceptance

Перед production пройти:

- registration;
- login;
- Marketplace;
- seller;
- resource;
- media;
- moderation;
- purchase;
- webhook;
- license;
- module.

==================================================
25. WORKSTREAM V — FINAL ACCEPTANCE
==================================================

Финальный acceptance должен быть не только unit tests.

Сценарий:

STAGING / PRODUCTION-LIKE

1. User registers.
2. User logs in.
3. Session survives reload.
4. User sees balance.
5. Seller account works.
6. Seller creates resource.
7. Seller uploads cover.
8. Seller uploads screenshots.
9. Seller uploads artifact.
10. Seller submits moderation.
11. Admin reviews resource.
12. Admin publishes.
13. Resource appears Marketplace.
14. Buyer opens Resource Detail.
15. Buyer gets free resource.
16. Buyer buys paid resource.
17. Real/sandbox provider confirms payment through actual webhook.
18. Purchase completes.
19. License activates.
20. DRM server accepts installation.
21. Module completes expected lifecycle.
22. Media loads from production storage.
23. Direct artifact access remains forbidden.
24. Monitoring sees requests.
25. Backup/restore verified.
26. Deployment rollback verified.

==================================================
26. TESTING REQUIREMENTS
==================================================

Перед завершением:

- [ ] Backend tests pass.
- [ ] Frontend typecheck passes.
- [ ] Server typecheck passes.
- [ ] Frontend production build passes.
- [ ] Browser E2E passes.
- [ ] Module tests pass.
- [ ] Production-like integration tests pass.
- [ ] Payment webhook test passes.
- [ ] Media storage test passes.
- [ ] DRM live integration test passes.
- [ ] Backup restore drill passes.
- [ ] Rollback drill passes.
- [ ] Load baseline passes.

Если какой-либо check невозможно выполнить в текущем окружении, не объявлять его PASS.

Записать:

NOT VERIFIED

и указать причину.

==================================================
27. P0 / P1 / P2 CLASSIFICATION
==================================================

P0 — блокирует production:

- S3 media contract;
- artifact privacy;
- database migration process;
- production secrets;
- real payment webhook;
- DRM live integration;
- HTTPS/domain;
- backup/recovery;
- critical authorization;
- production rate limits.

P1 — желательно до public launch:

- alerts;
- strict dependency security gate;
- SQL aggregation;
- rollback;
- load baseline;
- certificate monitoring;
- external uptime.

P2 — после launch:

- advanced autoscaling;
- advanced CDN;
- complex search infrastructure;
- advanced observability;
- large-scale performance optimization.

==================================================
28. НЕ ДЕЛАТЬ
==================================================

Не делать:

- новый marketplace;
- новые product features;
- новый DRM;
- новую payment architecture;
- новый search engine без benchmark;
- микросервисы;
- Kubernetes;
- event-driven architecture;
- message broker только ради production;
- сложный autoscaling;
- enterprise observability platform;
- глобальный refactor.

Если существующая система работает и закрывает requirement — оставить её.

==================================================
29. DEFINITION OF DONE
==================================================

PLAN-004 считается COMPLETED только если:

CONFIGURATION
- [ ] Production env matrix существует.
- [ ] Production startup validation закрывает critical secrets.
- [ ] Dev limits не используются в production.
- [ ] Cookies настроены безопасно.

STORAGE
- [ ] S3/R2 media contract исправлен.
- [ ] Media storage durable.
- [ ] Paid artifacts private.
- [ ] `/uploads` bypass отсутствует.
- [ ] Media cleanup работает.

DATABASE
- [ ] Production migration procedure существует.
- [ ] Backup существует.
- [ ] Restore проверен.
- [ ] DB connections проверены.

PAYMENTS
- [ ] Production configuration documented.
- [ ] Webhook verified.
- [ ] Idempotency verified.
- [ ] Failure states verified.
- [ ] Reconciliation verified.

DRM
- [ ] Live server integration verified.
- [ ] Windows module verified.
- [ ] Failure scenarios verified.

SECURITY
- [ ] Secrets absent from repo.
- [ ] Critical dependency vulnerabilities have release policy.
- [ ] CSP hardened.
- [ ] Security headers verified.
- [ ] CORS verified.
- [ ] Authorization audit completed.
- [ ] Direct storage access checked.

INFRA
- [ ] HTTPS works.
- [ ] Domain works.
- [ ] Nginx works.
- [ ] Health checks work.
- [ ] Docker runtime verified.
- [ ] Redis semantics defined.
- [ ] Email production path verified.

OBSERVABILITY
- [ ] logs work;
- [ ] metrics work;
- [ ] uptime works;
- [ ] critical alerts exist.

RECOVERY
- [ ] DB restore verified.
- [ ] Object storage recovery verified.
- [ ] Disaster recovery scenario verified.
- [ ] Rollback verified.

QUALITY
- [ ] Existing E2E remains green.
- [ ] Production-like E2E passes.
- [ ] Load baseline known.
- [ ] No known P0 blocker remains.

==================================================
30. FINAL STATUS RULE
==================================================

Не ставить:

PLAN-004 = COMPLETED

если существуют непроверенные P0 пункты.

Допустимый финальный статус:

IMPLEMENTATION COMPLETE
BUT PRODUCTION NOT VERIFIED

если код готов, но реальные production checks ещё невозможно выполнить.

Только после прохождения production-like acceptance:

PLAN-004 = COMPLETED

==================================================
31. FINAL DOCUMENTATION
==================================================

После завершения обновить:

mta-market-document/PROJECT.md
mta-market-document/DEVELOPMENT/CURRENT.md
mta-market-document/DEVELOPMENT/COMPLETED/PLAN-004.md

PLAN-004.md должен содержать:

- цель;
- исходное production gap state;
- все найденные P0/P1/P2;
- что исправлено;
- что проверено;
- какие реальные окружения использовались;
- результаты tests;
- payment verification;
- DRM verification;
- storage verification;
- backup/restore verification;
- rollback verification;
- known limitations;
- final production readiness verdict.

==================================================
32. ОСНОВНОЙ КРИТЕРИЙ
==================================================

После PLAN-004 должно быть возможно сказать:

"Мы понимаем, как этот сервис запускается.
Мы понимаем, где хранятся данные.
Мы понимаем, как проходят деньги.
Мы понимаем, как работает DRM.
Мы понимаем, как мониторить сервис.
Мы умеем восстановить его после сбоя.
Мы умеем откатить deployment.
И у нас нет известных критических production blockers."

Это и есть конечная цель PLAN-004.

Не количество изменённых файлов.

Не количество тестов.

Не количество новых функций.

А доказанная способность MTA Market безопасно работать в production.
