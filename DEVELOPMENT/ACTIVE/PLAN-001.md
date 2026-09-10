# PLAN-001 — Initial Product Assembly & Completion

**Status:** ACTIVE
**Type:** Initial Product Release / Product Assembly
**Repositories:**

* `acc-holo-dev/mta-market-site`
* `acc-holo-dev/mta-market-module`
* `acc-holo-dev/mta-market-document`

---

# 1. Цель плана

Цель `PLAN-001` — не создать MTA Market с нуля и не переписать существующую архитектуру.

Цель — взять текущее состояние всех трёх репозиториев, сохранить уже работающие подсистемы и довести проект до первого цельного, реально используемого продукта.

После полного завершения `PLAN-001` пользователь должен иметь возможность:

1. зарегистрироваться;
2. войти через логин/email + пароль;
3. войти через Discord;
4. попасть в рабочий Marketplace;
5. просматривать бесплатные и платные ресурсы;
6. открыть страницу ресурса;
7. получить бесплатный ресурс или приобрести платный;
8. видеть свои покупки и лицензии;
9. видеть собственный баланс;
10. открыть и редактировать базовый профиль;
11. получить доступ к seller-функционалу;
12. создать ресурс;
13. заполнить необходимые данные;
14. загрузить artifact;
15. закончить создание ресурса кнопкой `Готово`;
16. отправить ресурс на модерацию;
17. увидеть статус модерации;
18. администратору открыть Admin Panel;
19. администратору увидеть ресурс в очереди модерации;
20. одобрить или отклонить ресурс;
21. после одобрения увидеть ресурс в Marketplace;
22. пройти основной пользовательский flow от начала до конца без ручного вмешательства разработчика.

План считается завершённым только тогда, когда этот продуктовый цикл реально работает.

---

# 2. Главный принцип

## Не переписывать то, что уже существует

Перед реализацией каждой задачи агент обязан проверить:

* текущий код;
* текущую БД/Prisma schema;
* текущие API routes;
* текущий frontend;
* текущие tests;
* текущую реализацию module;
* текущую документацию;
* последние коммиты.

Если требуемая возможность уже существует, задача должна быть переведена из:

`IMPLEMENT`

в:

`VERIFY / FIX / INTEGRATE / COMPLETE`

---

# 3. Источники истины

При работе над PLAN-001 использовать следующий порядок доверия:

1. текущий код;
2. текущие тесты;
3. frozen contracts / protocol specifications;
4. актуальная техническая документация;
5. старые roadmap/planning documents.

Старые planning-документы не являются обязательным roadmap.

Если старый документ противоречит текущему коду, не восстанавливать старую реализацию автоматически.

---

# 4. Что НЕ входит в PLAN-001

Следующие функции не должны добавляться в PLAN-001 только потому, что они существовали в старых планах:

* Live Demo;
* Asset Studio;
* Leak Radar;
* watermark forensic system;
* seller analytics;
* subscriptions;
* advanced search engine;
* bundles;
* wishlist;
* influencer/referral systems;
* A/B listings;
* scheduled releases;
* seller API;
* seller webhooks;
* advanced release channels;
* public license-health metrics;
* advanced compatibility intelligence;
* subscriptions;
* chat;
* мобильное приложение;
* собственный форум;
* любые другие future features.

Эти элементы должны быть перенесены в `IDEAS`.

---

# 5. Целевое состояние PLAN-001

После завершения PLAN-001 MTA Market должен представлять собой минимальный, но цельный marketplace:

```text
USER
 |
 +--> Registration / Login
 |
 +--> Profile
 |
 +--> Balance
 |
 +--> Marketplace
 |     |
 |     +--> Free Resource
 |     |
 |     +--> Paid Resource
 |
 +--> Purchase
 |     |
 |     +--> Entitlement / License
 |
 +--> Seller
 |     |
 |     +--> Create Resource
 |     +--> Upload Artifact
 |     +--> Submit
 |             |
 |             v
 |        PENDING_REVIEW
 |
 +--> Admin
       |
       +--> Moderation Queue
       |
       +--> Approve
       |      |
       |      v
       |   PUBLISHED
       |
       +--> Reject
```

---

# 6. Workstream A — Authentication

## A-001 — Local registration

Implement a normal account registration flow:

* username/login;
* email;
* password;
* password confirmation;
* validation;
* unique username;
* unique email where applicable;
* secure password hashing;
* account creation;
* automatic normal user role;
* appropriate error handling.

Не создавать отдельную несовместимую auth-систему.

Она должна использовать существующий authentication/session layer.

### Acceptance

* новый пользователь может зарегистрироваться;
* пользователь получает рабочую сессию;
* пароль никогда не хранится в plaintext;
* повторный username/email корректно отклоняется;
* ошибка отображается человеку;
* existing OAuth system не ломается.

---

## A-002 — Local login

Добавить:

* username/email + password login;
* session creation;
* access token;
* refresh mechanism;
* logout;
* invalid credentials handling.

### Acceptance

Пользователь может войти:

* через username;
* через email;
* через Discord.

---

## A-003 — Registration/login frontend

Создать полноценные страницы:

* `/auth/login`;
* `/auth/register`.

На странице входа:

* username/email;
* password;
* Discord;
* link to registration.

На регистрации:

* username;
* email;
* password;
* confirm password;
* link to login.

---

## A-004 — Existing OAuth preservation

Проверить, что после добавления password auth продолжают работать:

* Discord login;
* existing identity/account linking;
* refresh token rotation;
* logout.

---

# 7. Workstream B — Account / Profile

## B-001 — Account page

Довести account area до базового пользовательского профиля.

Минимальный состав:

* username;
* email;
* display name;
* avatar, если уже поддерживается;
* linked accounts;
* balance;
* account information.

---

## B-002 — Profile editing

Пользователь должен иметь возможность изменить доступные базовые поля профиля.

Нельзя разрешать изменение:

* role;
* moderation flags;
* protected identity fields;
* административных параметров.

---

## B-003 — Account navigation

Создать или привести к единому виду пользовательскую навигацию:

* Marketplace;
* Profile / Account;
* Purchases;
* Seller;
* Balance;
* Admin — только для ADMIN/MODERATOR.

---

# 8. Workstream C — Balance

## C-001 — User balance model

Проверить существующую модель seller balance / ledger.

Не считать seller balance автоматически эквивалентом пользовательского баланса.

Определить и реализовать полноценное persisted balance для пользователя.

Минимально:

```text
User
 |
 +--> Balance
```

Balance должен храниться в БД.

---

## C-002 — Balance initialization

Каждый новый пользователь должен получить корректное начальное состояние:

```text
balance = 0
```

Баланс не должен быть hardcoded только на frontend.

---

## C-003 — Balance UI

Баланс должен быть виден пользователю:

* в profile/account;
* в подходящем месте основной навигации/header.

---

## C-004 — No top-up yet

В PLAN-001 не требуется реализовывать пополнение баланса.

UI может показывать:

`Пополнение пока недоступно`

но само наличие balance должно быть реальным.

---

# 9. Workstream D — Marketplace

## D-001 — Marketplace as primary product surface

Marketplace должен стать основной страницей продукта после входа.

Минимум:

* resource list;
* published resources;
* free resources;
* paid resources;
* price;
* type;
* seller;
* resource card;
* link to resource detail.

---

## D-002 — Basic filtering

Поддержать только действительно необходимую фильтрацию:

* free/paid;
* type, если существующий domain model это позволяет.

Не добавлять сложный search engine.

---

## D-003 — Resource detail

Довести существующую страницу ресурса до рабочего продукта.

Минимум:

* title;
* description;
* type;
* price;
* seller;
* versions where available;
* reviews where available;
* purchase action;
* free acquisition action;
* purchase state;
* ownership state;
* license state.

---

## D-004 — Resource states

Frontend должен корректно обрабатывать:

* loading;
* not found;
* published;
* unavailable;
* suspended where applicable.

Draft/PENDING_REVIEW resources не должны попадать в публичный Marketplace.

---

# 10. Workstream E — Seller

## E-001 — Seller entry

Проверить существующий seller application flow.

Не переделывать его без необходимости.

Довести UX до состояния:

```text
User
 |
 +--> Become Seller
 |
 +--> Seller Application
 |
 +--> PENDING
 |
 +--> APPROVED
 |
 +--> Seller Area
```

---

## E-002 — Seller dashboard

Seller dashboard должен позволять:

* видеть свои ресурсы;
* видеть их statuses;
* создавать ресурс;
* открыть существующий draft;
* отправить ресурс на модерацию;
* видеть результат модерации.

---

## E-003 — Resource creation flow

Вместо разрозненного создания сущности сделать единый понятный flow:

```text
Step 1
Basic information

Step 2
Resource type / price

Step 3
Artifact

Step 4
Preview / validation

Step 5
Final confirmation

Step 6
Submit
```

Фактическое количество экранов может отличаться.

Главное — пользователь должен воспринимать это как один процесс создания ресурса.

---

## E-004 — Required resource data

Определить минимально необходимые поля на основании уже существующего backend/domain model.

Не заставлять пользователя заполнять поля, которые не нужны для первой версии.

---

## E-005 — Artifact upload

Подключить существующую artifact pipeline.

Пользователь должен иметь возможность:

* выбрать artifact;
* загрузить его;
* увидеть progress/error;
* получить результат validation;
* понять, что именно загружено.

Не создавать параллельную систему хранения artifacts.

---

## E-006 — Final submit

Финальная кнопка должна переводить ресурс в:

```text
PENDING_REVIEW
```

только если все обязательные данные присутствуют и artifact pipeline находится в допустимом состоянии.

До отправки ресурс остаётся:

```text
DRAFT
```

---

# 11. Workstream F — Moderation

## F-001 — Moderation queue

Использовать существующий `/admin/resources`.

Очередь должна явно показывать:

* resource;
* seller;
* type;
* created date;
* current status.

---

## F-002 — Resource review

Администратор должен иметь возможность открыть информацию о ресурсе перед решением.

Минимально видеть:

* title;
* description;
* price;
* seller;
* artifact/version information;
* validation state.

---

## F-003 — Approve

При approve:

```text
PENDING_REVIEW
        ↓
PUBLISHED
```

После этого ресурс появляется в Marketplace.

Не создавать отдельный ручной publish operation вне moderation flow.

---

## F-004 — Reject

При reject ресурс должен перейти в корректное непубличное состояние.

Должна сохраняться причина отказа, если текущая модель это поддерживает.

Seller должен иметь возможность понять результат модерации.

---

## F-005 — Moderation audit

Сохранить существующий moderation event / audit infrastructure.

Не удалять audit trail ради упрощения UI.

---

# 12. Workstream G — Admin

## G-001 — Admin detection

Если пользователь имеет:

```text
ADMIN
```

или соответствующую административную роль, frontend должен давать доступ к Admin Panel.

Обычный пользователь не должен видеть административную функциональность.

---

## G-002 — Admin navigation

После входа администратора в основной navigation должен появляться Admin entry.

Администратор не должен вручную вводить неизвестный URL, чтобы попасть в систему.

---

## G-003 — Existing admin functionality

Сохранить и довести уже существующие возможности:

* moderation;
* sellers;
* disputes;
* versions;
* basic platform stats.

Не переписывать их без необходимости.

---

## G-004 — Default admin account for development

В development environment должен существовать воспроизводимый способ получить ADMIN account.

Это может быть:

* seed;
* development bootstrap;
* documented admin promotion mechanism.

Нельзя создавать скрытый hardcoded backdoor.

---

# 13. Workstream H — Commerce

## H-001 — Free acquisition

У бесплатного ресурса:

```text
price = 0
```

должен быть отдельный acquisition flow.

Не создавать фиктивный платеж.

После acquisition пользователь получает соответствующее право/licence.

---

## H-002 — Paid purchase

Проверить существующий paid purchase flow.

Требуемая цепочка:

```text
Marketplace
 ↓
Buy
 ↓
Purchase
 ↓
Payment
 ↓
Payment confirmation
 ↓
Entitlement / License
```

---

## H-003 — Payment integration verification

Использовать существующую payment abstraction и YooKassa integration.

Не добавлять новый payment provider.

Не менять архитектуру платежей без необходимости.

---

## H-004 — Ownership

После успешного приобретения пользователь должен видеть:

* purchase;
* resource;
* purchase status;
* license/entitlement state.

---

# 14. Workstream I — DRM / Artifact / Module

## I-001 — Do not rewrite DRM v2

DRM v2 уже существует.

Запрещено переписывать protocol v2 только ради PLAN-001.

---

## I-002 — Marketplace integration

Проверить фактическую связь:

```text
published resource
 ↓
version
 ↓
artifact
 ↓
signature
 ↓
purchase/license
 ↓
installation
 ↓
module
 ↓
DRM
```

---

## I-003 — Module integration test

Провести реальный integration scenario:

1. получить entitlement/license;
2. создать installation;
3. пройти challenge;
4. activate;
5. получить lease;
6. проверить artifact binding;
7. проверить DEK release;
8. убедиться, что module способен пройти ожидаемый protected-resource lifecycle.

---

## I-004 — Do not reimplement existing cryptography

Использовать существующие:

* Ed25519;
* AES-256-GCM;
* key store;
* canonical JSON;
* lease verification.

---

# 15. Workstream J — Frontend Completion

## J-001 — Global navigation

Привести UI к единому продукту.

Минимальная структура:

```text
MTA Market
Marketplace
Profile
Purchases
Balance

Seller
Admin (conditional)
```

---

## J-002 — Auth states

Frontend должен корректно обрабатывать:

* anonymous;
* authenticated;
* seller;
* moderator;
* admin.

---

## J-003 — Empty / Loading / Error states

Для основных экранов обязательно должны существовать понятные состояния:

* loading;
* empty;
* failure;
* success.

---

## J-004 — No placeholder product surfaces

Не должно оставаться главных UI-элементов вида:

* API pending;
* coming soon;
* TODO;
* placeholder;
* mock-only functionality

в тех местах, которые относятся к обязательному PLAN-001 flow.

---

# 16. Workstream K — Product Consistency

## K-001 — Unified terminology

Использовать единые термины:

* Resource;
* Seller;
* Buyer;
* Marketplace;
* Purchase;
* License;
* Balance;
* Moderation;
* Published;
* Draft;
* Pending Review.

---

## K-002 — Status consistency

Frontend и backend должны использовать существующие status values и state machine.

Не вводить параллельные frontend-only statuses.

---

## K-003 — Permission consistency

Проверить permissions на backend и frontend.

Frontend hiding не заменяет backend authorization.

---

# 17. Workstream L — Testing

## L-001 — Authentication E2E

Проверить:

```text
register
login
logout
Discord login
session restore
```

---

## L-002 — Buyer E2E

Проверить:

```text
login
 ↓
Marketplace
 ↓
resource
 ↓
free/paid acquisition
 ↓
purchase
 ↓
license
 ↓
account
```

---

## L-003 — Seller E2E

Проверить:

```text
login
 ↓
seller
 ↓
create resource
 ↓
fill data
 ↓
upload artifact
 ↓
Done
 ↓
PENDING_REVIEW
```

---

## L-004 — Admin E2E

Проверить:

```text
admin login
 ↓
Admin
 ↓
Moderation
 ↓
resource
 ↓
approve
 ↓
PUBLISHED
 ↓
Marketplace
```

Отдельно проверить reject flow.

---

## L-005 — Full product E2E

Это главный тест PLAN-001.

Один сценарий должен проходить полный цикл:

```text
Seller creates resource
        ↓
Resource submitted
        ↓
Admin approves
        ↓
Resource appears in Marketplace
        ↓
Buyer registers/logs in
        ↓
Buyer acquires resource
        ↓
Purchase/license created
        ↓
Buyer sees entitlement
```

Если ресурс защищён DRM-механикой, дополнительно проверяется module lifecycle.

---

# 18. Workstream M — Cleanup

## M-001 — Remove dead code

Удалить только подтверждённо неиспользуемые:

* legacy routes;
* dead components;
* obsolete UI;
* obsolete helper code.

---

## M-002 — Remove misleading placeholders

Убрать интерфейсные элементы, которые создают ощущение, что функция существует, если она фактически не работает.

---

## M-003 — Fix obvious inconsistencies

Исправлять реальные дефекты, обнаруженные во время сборки PLAN-001.

Не использовать PLAN-001 как повод для глобального refactor.

---

# 19. Workstream N — Release Readiness

## N-001 — Development reproducibility

На чистой development environment должно быть возможно:

```text
clone
install dependencies
start services
migrate database
seed development state
run backend
run frontend
login
use Marketplace
```

---

## N-002 — Admin reproducibility

Должен существовать документированный development способ получить ADMIN account.

---

## N-003 — Environment verification

Проверить:

* environment variables;
* database;
* Redis;
* artifact storage;
* authentication;
* payment configuration where needed.

---

## N-004 — Build verification

Перед завершением PLAN-001 необходимо пройти:

* frontend typecheck;
* backend tests;
* frontend tests where present;
* module tests;
* relevant integration tests;
* full product E2E.

---

# 20. Workstream O — Documentation

В рамках PLAN-001 НЕ создаётся полная техническая энциклопедия проекта.

Необходимо только:

* обновить README/document map при изменении структуры;
* поддерживать DEVELOPMENT/CURRENT;
* зафиксировать факт завершения PLAN-001;
* при необходимости добавить короткие инструкции для нового flow.

Большая техническая документация будет создана после завершения продукта.

---

# 21. Definition of Done

`PLAN-001` считается **COMPLETED** только если выполнены все условия.

## User

* [ ] User can register with username/email/password.
* [ ] User can login with username/email/password.
* [ ] User can login with Discord.
* [ ] User can logout.
* [ ] Session survives page reload.
* [ ] User can view profile.
* [ ] User can edit allowed profile data.
* [ ] User has a real persisted balance.
* [ ] User can see balance.

## Marketplace

* [ ] Marketplace is the main product surface.
* [ ] Published resources are visible.
* [ ] Free resources are visible.
* [ ] Paid resources are visible.
* [ ] Resource detail works.
* [ ] Basic filters work.
* [ ] Non-published resources are not publicly visible.

## Seller

* [ ] User can enter seller flow.
* [ ] Seller application works.
* [ ] Approved seller can access seller area.
* [ ] Seller can create resource.
* [ ] Seller can fill required data.
* [ ] Seller can upload artifact.
* [ ] Seller can finish resource creation.
* [ ] Resource becomes `PENDING_REVIEW`.
* [ ] Seller can see moderation status.

## Admin

* [ ] ADMIN account can authenticate.
* [ ] Admin navigation appears for admin.
* [ ] Admin panel is accessible.
* [ ] Moderation queue works.
* [ ] Admin can inspect submitted resource.
* [ ] Admin can approve resource.
* [ ] Admin can reject resource.
* [ ] Moderation events are preserved.
* [ ] Approved resource appears in Marketplace.

## Commerce

* [ ] Free acquisition works.
* [ ] Paid purchase flow works.
* [ ] Payment flow works where configured.
* [ ] Purchase is persisted.
* [ ] Entitlement/license is persisted.
* [ ] Buyer can see purchase/license.

## DRM / Module

* [ ] Existing DRM v2 remains intact.
* [ ] Existing artifact security remains intact.
* [ ] Marketplace license flow connects correctly to DRM.
* [ ] Module integration scenario passes.

## Quality

* [ ] No critical known blocker remains in the core product flow.
* [ ] No core page is a placeholder.
* [ ] No required flow depends on manual DB edits.
* [ ] Core E2E passes.
* [ ] Development setup is reproducible.

---

# 22. Priority Rules

При конфликте задач использовать этот порядок:

```text
P0 — blocks the product flow
P1 — breaks a required user experience
P2 — quality / consistency issue
P3 — polish
```

Работа над P2/P3 не должна блокировать завершение P0.

---

# 23. Change Control

Во время PLAN-001 запрещается автоматически добавлять новые большие подсистемы.

Новая идея должна пройти:

```text
Idea
 ↓
Does PLAN-001 require it?
 ├── YES → add to PLAN-001
 └── NO  → move to IDEAS
```

Особенно запрещается расширять PLAN-001 следующими аргументами:

* «это когда-то было в roadmap»;
* «это было в старом task list»;
* «это архитектурно красиво»;
* «это пригодится потом».

---

# 24. Agent Operating Rules

ИИ-агент обязан:

1. Перед изменением читать текущую реализацию.
2. Проверять существующий API перед созданием нового endpoint.
3. Проверять существующую модель БД перед созданием новой.
4. Проверять существующие frontend components перед созданием новых.
5. Не дублировать authentication, payment, DRM или artifact infrastructure.
6. Не удалять существующую рабочую функциональность без причины.
7. Не менять frozen DRM v2 contract в рамках PLAN-001.
8. После каждого крупного изменения запускать соответствующие tests/typecheck.
9. При обнаружении устаревшего документа не следовать ему автоматически.
10. Если реализация и документация расходятся, сначала подтвердить фактическое состояние по коду и тестам.
11. При необходимости сделать небольшой refactor внутри текущего subsystem, но не выполнять глобальный redesign.
12. Любое изменение, затрагивающее другой репозиторий, должно учитывать cross-repository contract.
13. Не считать задачу завершённой только потому, что код компилируется.
14. Функция считается завершённой только после проверки её пользовательского или системного результата.

---

# 25. Definition of Completion for the Whole Plan

Финальная проверка должна выглядеть так:

```text
CLEAN DEVELOPMENT ENVIRONMENT
        ↓
REGISTER
        ↓
LOGIN
        ↓
PROFILE
        ↓
BALANCE
        ↓
SELLER
        ↓
CREATE RESOURCE
        ↓
UPLOAD ARTIFACT
        ↓
DONE
        ↓
PENDING_REVIEW
        ↓
ADMIN
        ↓
MODERATION
        ↓
PUBLISHED
        ↓
MARKETPLACE
        ↓
BUYER
        ↓
PURCHASE / FREE ACQUISITION
        ↓
LICENSE
        ↓
MODULE / DRM
        ↓
SUCCESS
```

Если этот сценарий проходит воспроизводимо — `PLAN-001` завершён.

---

# 26. После завершения

После выполнения всех требований:

1. обновить `DEVELOPMENT/CURRENT.md`;
2. перенести `PLAN-001` из `ACTIVE` в `COMPLETED`;
3. зафиксировать фактическое состояние продукта;
4. не добавлять туда будущие идеи;
5. следующий development cycle начинать новым `PLAN-002`.

`PLAN-002` уже не должен быть планом спасения проекта.

Он должен быть планом развития работающего продукта.
