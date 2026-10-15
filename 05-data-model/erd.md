# Модель данных: ER-диаграммы

Связи между таблицами по доменным группам (диаграммы Mermaid `erDiagram`) и пояснение главных агрегатов. Полное описание колонок, ключей, индексов и перечислений — в [tables.md](tables.md); у каждой таблицы там есть раздел с якорем `tables.md#<имя_таблицы>`.

Связанные разделы: [архитектура](../07-architecture/architecture.md), бизнес-логика доменов `../02-business-logic/<домен>.md`, открытые вопросы — `../_oq/DATA.md`.

**Как читать диаграммы.**

- В сущностях показаны первичный ключ (`PK`), внешние ключи (`FK`), уникальные (`UK`) и 2–5 ключевых бизнес-полей; полный набор колонок — в tables.md. Типы упрощены: `int`/`bigint` — целые, `varchar`/`text` — строки, `numeric` — деньги/объёмы, `timestamptz` — момент времени (UTC), `jsonb` — JSON.
- Сплошная линия — внешний ключ в БД; подпись — колонка FK в дочерней таблице. Пунктир — **логическая связь без FK** (полиморфная ссылка, ссылка-автор, курсор, JSON-список id) с пометкой «логич.».
- Кардинальности: `||` — ровно один, `|o` — ноль или один (FK допускает NULL), `o{` — ноль или много, `o|` — ноль или один (FK уникален, связь 1:1).
- Сущности из других групп показаны «заглушками» — только с первичным ключом.
- Чтобы диаграммы оставались читаемыми, на части из них не показаны связи-авторы с `users` (`created_by`, `changed_by`, `decided_by`, `updated_by` и т.п.) и ссылки пакета документов на `attachments` — они перечислены в tables.md у каждой таблицы.

---

## 1. Обзор: группы и связи между ними

```mermaid
flowchart LR
  ID["Пользователи и доступ<br/>users, roles, role_permissions,<br/>auth_sessions, supplier_users"]
  AU["Аудит<br/>security_audit_log"]
  BI["Биллинг ИИ<br/>billing_*, ai_operations,<br/>ai_calls, topup_requests"]
  CA["Справочники<br/>work_types, units,<br/>unit_aliases, status_definitions"]
  PO["Портфели и проекты<br/>portfolios, projects,<br/>гранты доступа"]
  SU["Контрагенты<br/>suppliers и supplier_*"]
  TE["Тендеры (ядро)<br/>tenders, разделы, позиции, условия,<br/>приглашения, КП, раунды"]
  BU["Бюджет<br/>budget_*"]
  CO["Коммуникации<br/>comments, chat_*,<br/>app_notifications"]
  AI["ИИ-анализ<br/>ai_analysis_results,<br/>ai_insight_cache, ai_chat_*"]
  SE["Безопасность<br/>инциденты, РКН, SIEM,<br/>правила техлогов"]
  PL["Платформа и файлы<br/>attachments, background_jobs,<br/>кэш, лимиты, настройки"]

  ID -->|"supplier_users"| SU
  ID -->|"гранты доступа"| PO
  ID -.->|"user_id (логич.)"| AU
  PO -->|"projects → tenders"| TE
  CA -->|"виды работ"| TE
  CA -->|"виды работ"| SU
  SU -->|"приглашения, победитель"| TE
  BU -->|"бюджетная позиция"| TE
  TE -->|"тред на тендер × контрагент,<br/>комментарии к позициям"| CO
  SU --> CO
  TE -->|"снимок СТ"| AI
  AI -->|"обращения к модели"| BI
  AU -->|"события → правила → инциденты"| SE
  PL -.->|"вложения (полиморфно)"| SU
  PL -.->|"вложения, задания"| TE
  ID --> BI
```

Пользователь (`users`) — точка входа почти для всех групп: он автор действий, ответственный, адресат уведомлений. Контрагенты и тендеры связаны через приглашения (`tender_invitations`), на которых держатся КП и чат «тендер × контрагент». Файлы привязаны к сущностям полиморфно (`attachments.entity_type/entity_id`), журнал ИБ ссылается на пользователей и сущности логически — без внешних ключей, чтобы записи журнала переживали любые удаления.

### 1.1. Главные агрегаты

**Тендер** ([`tenders`](tables.md#tenders)) — центральный агрегат системы:

- *Состав (ФКП):* тендер → разделы [`tender_sections`](tables.md#tender_sections) (дерево через `parent_section_id`, позиции и подразделы перемежаются по `items_before`) → позиции [`tender_items`](tables.md#tender_items) (объём, единица, атрибуты; снятые позиции остаются с `is_removed`). Дополнительные условия — [`tender_condition_fields`](tables.md#tender_condition_fields). Каждая значимая правка состава даёт новую версию шаблона ФКП [`tender_fkp_versions`](tables.md#tender_fkp_versions) (JSON-слепок) и увеличивает `tenders.tender_revision`.
- *Контекст:* проект [`projects`](tables.md#projects) (обязателен), виды работ [`tender_work_types`](tables.md#tender_work_types), бюджетная позиция и плановая сумма, ответственный, пакет документов (ссылки на [`attachments`](tables.md#attachments)), черновик протокола [`tender_protocols`](tables.md#tender_protocols) (1:1), журнал [`activity_log`](tables.md#activity_log).
- *Участники:* приглашение [`tender_invitations`](tables.md#tender_invitations) (тендер × контрагент, персональная ссылка) → ровно одно КП.

**КП** ([`quotes`](tables.md#quotes)) — агрегат внутри тендера, 1:1 с приглашением: КП → версии [`quote_versions`](tables.md#quote_versions) (каждая подача — новая версия, одна текущая; фиксирует номер версии ФКП и подтверждение НДС) → строки [`quote_items`](tables.md#quote_items) (цены материалов и работ за единицу по позиции тендера; отказ от позиции; корректировка объёма или новая позиция с решением специалиста) и значения условий [`quote_condition_values`](tables.md#quote_condition_values). До подачи состояние формы хранится в черновике [`invite_quote_drafts`](tables.md#invite_quote_drafts). Сравнительная таблица (СТ) не хранится — строится из текущих версий КП всех участников (с кэшем готового ответа в `heavy_read_cache`).

**Раунд** ([`tender_rounds`](tables.md#tender_rounds)) — повторный запрос КП: раунд → адресаты [`tender_round_recipients`](tables.md#tender_round_recipients) (контрагент, доставка письма, ответ = ссылка на поданную версию КП).

**Контрагент** ([`suppliers`](tables.md#suppliers)) — второй крупный агрегат: карточка (реквизиты, шифруемые контакты организации, оси проверок СБ/юрист/финансы, вычисляемый сводный статус, пул) → контакты-представители [`supplier_contacts`](tables.md#supplier_contacts) (ПДн, шифруются; опционально связаны с пользователем) и пользователи ЛК через [`supplier_users`](tables.md#supplier_users) → специализация [`supplier_work_types`](tables.md#supplier_work_types) и опыт [`supplier_work_experience`](tables.md#supplier_work_experience) → выручка по годам [`supplier_revenues`](tables.md#supplier_revenues) → автопроверки по реестрам [`supplier_checks`](tables.md#supplier_checks) → история решений [`supplier_qualification_history`](tables.md#supplier_qualification_history) (append-only, триггеры) и лента статусных событий [`supplier_status_events`](tables.md#supplier_status_events); документы анкеты — вложения `entity_type='supplier'`.

**Пользователь** ([`users`](tables.md#users)) — одна роль ([`roles`](tables.md#roles), права — [`role_permissions`](tables.md#role_permissions)), сессии устройств [`auth_sessions`](tables.md#auth_sessions), шаг MFA [`login_challenges`](tables.md#login_challenges), одноразовые ссылки [`password_reset_tokens`](tables.md#password_reset_tokens), гранты на портфели/проекты, для поставщика — привязка к контрагенту.

**Портфель** ([`portfolios`](tables.md#portfolios)) → проекты и подпроекты [`projects`](tables.md#projects) → тендеры. Доступ специалиста — гранты на портфель или проект с наследованием вниз по дереву.

**Чат** ([`chat_threads`](tables.md#chat_threads)) — один тред на пару «тендер × контрагент» → сообщения [`chat_messages`](tables.md#chat_messages) (глобальный номер события, ответы, мост к ячейке КП) → ревизии [`chat_message_revisions`](tables.md#chat_message_revisions); персональные отметки прочтения [`chat_thread_reads`](tables.md#chat_thread_reads).

**Инцидент ИБ** ([`security_incidents`](tables.md#security_incidents)) — создаётся правилом [`security_incident_rules`](tables.md#security_incident_rules) по событиям журнала [`security_audit_log`](tables.md#security_audit_log) → лента расследования [`security_incident_timeline`](tables.md#security_incident_timeline) → пакеты для РКН [`rkn_packages`](tables.md#rkn_packages).

**Клиент биллинга** ([`billing_clients`](tables.md#billing_clients)) → ИИ-операции [`ai_operations`](tables.md#ai_operations) → обращения к модели [`ai_calls`](tables.md#ai_calls); книга баланса [`billing_entries`](tables.md#billing_entries) (списание — одна запись на операцию) и заявки на пополнение [`topup_requests`](tables.md#topup_requests).

---

## 2. Пользователи, аудит, биллинг

### 2.1. Пользователи, доступ, аудит

Журнал ИБ ссылается на пользователя логически (без FK). Таблица пауз входа `login_throttle` связей не имеет (ключ — «e-mail + IP»).

```mermaid
erDiagram
  roles ||--o{ users : "role_id"
  roles ||--o{ role_permissions : "role_id"
  suppliers ||--o{ supplier_users : "supplier_id"
  users ||--o| supplier_users : "user_id"
  users ||--o{ auth_sessions : "user_id"
  users ||--o{ login_challenges : "user_id"
  users ||--o{ password_reset_tokens : "user_id"
  users |o..o{ security_audit_log : "user_id (логич.)"
  users {
    int id PK
    int role_id FK
    varchar email UK
    varchar full_name
    bool is_active
    bool is_superadmin
  }
  roles {
    int id PK
    varchar code UK
    varchar name
  }
  role_permissions {
    int role_id PK, FK
    varchar code PK
  }
  supplier_users {
    int id PK
    bigint user_id FK, UK
    bigint supplier_id FK
    bool is_active
  }
  auth_sessions {
    uuid id PK
    int user_id FK
    varchar refresh_hash UK
    timestamptz expires_at
    timestamptz revoked_at
  }
  login_challenges {
    bigint id PK
    int user_id FK
    varchar token_hash UK
    timestamptz expires_at
  }
  password_reset_tokens {
    int id PK
    int user_id FK
    varchar token_hash UK
    timestamptz expires_at
    timestamptz used_at
  }
  login_throttle {
    text key PK
    int failures
    timestamptz blocked_until
  }
  security_audit_log {
    int id PK
    varchar action
    varchar entity_type
    bigint entity_id
    timestamptz created_at
  }
  suppliers {
    int id PK
  }
```

### 2.2. Биллинг ИИ

Баланс клиента = сумма `billing_entries.tokens`. Списание за операцию — не более одной записи (`uq_billing_entries_operation`). `billing_owner_settings` — одна строка настроек владельца, связей не имеет.

```mermaid
erDiagram
  billing_clients ||--o{ ai_operations : "client_id"
  users |o--o{ ai_operations : "user_id"
  billing_clients ||--o{ ai_calls : "client_id"
  ai_operations |o--o{ ai_calls : "operation_id"
  billing_clients ||--o{ topup_requests : "client_id"
  users ||--o{ topup_requests : "requested_by"
  users |o--o{ topup_requests : "decided_by"
  billing_clients ||--o{ billing_entries : "client_id"
  ai_operations |o--o{ billing_entries : "operation_id"
  topup_requests |o--o{ billing_entries : "topup_request_id"
  users |o--o{ billing_entries : "created_by"
  billing_clients {
    bigint id PK
    varchar code UK
    numeric multiplier
    bool block_at_zero
  }
  billing_owner_settings {
    int id PK
    numeric usd_rate
  }
  ai_operations {
    bigint id PK
    bigint client_id FK
    bigint user_id FK
    varchar purpose
    varchar status
    varchar op_key
  }
  ai_calls {
    bigint id PK
    bigint client_id FK
    bigint operation_id FK
    varchar purpose
    varchar model
    numeric cost_rub
  }
  topup_requests {
    bigint id PK
    bigint client_id FK
    bigint requested_by FK
    bigint decided_by FK
    bigint tokens
    varchar status
  }
  billing_entries {
    bigint id PK
    bigint client_id FK
    bigint operation_id FK
    bigint topup_request_id FK
    bigint created_by FK
    varchar kind
    bigint tokens
  }
  users {
    int id PK
  }
```

---

## 3. Каталог и портфели

Справочники и дерево объектов. Гранты доступа (`user_portfolio_access`, `user_project_access`) в БД не имеют внешних ключей — связи логические. Пользовательские статусы связаны с тендерами и КП по строковому коду (`custom_status_code`).

```mermaid
erDiagram
  work_types |o--o{ work_types : "parent_id"
  units ||--o{ unit_aliases : "unit_id"
  projects |o--o{ projects : "parent_project_id"
  portfolios ||--o{ projects : "portfolio_id"
  projects ||--o{ project_work_type_configs : "project_id"
  tender_templates |o--o{ project_work_type_configs : "template_id"
  work_types ||--o{ project_work_type_configs : "work_type_id"
  users ||..o{ user_portfolio_access : "user_id (логич.)"
  portfolios ||..o{ user_portfolio_access : "portfolio_id (логич.)"
  users ||..o{ user_project_access : "user_id (логич.)"
  projects ||..o{ user_project_access : "project_id (логич.)"
  projects ||--o{ tenders : "project_id"
  status_definitions |o..o{ tenders : "custom_status_code (логич.)"
  status_definitions |o..o{ quotes : "custom_status_code (логич.)"
  work_types {
    int id PK
    bigint parent_id FK
    varchar name UK
    bool is_active
  }
  units {
    int id PK
    varchar name UK
  }
  unit_aliases {
    varchar alias_key PK
    int unit_id FK
  }
  status_definitions {
    int id PK
    varchar entity_type
    varchar code
    varchar label
  }
  portfolios {
    int id PK
    varchar name
    bool is_archived
  }
  projects {
    int id PK
    bigint portfolio_id FK
    bigint parent_project_id FK
    varchar name
    varchar address
    bool is_archived
  }
  user_portfolio_access {
    int user_id PK
    int portfolio_id PK
  }
  user_project_access {
    int user_id PK
    int project_id PK
  }
  project_work_type_configs {
    int id PK
    bigint project_id FK
    bigint work_type_id FK
    bigint template_id FK
    date collect_from
    date collect_to
  }
  users {
    int id PK
  }
  tender_templates {
    int id PK
  }
  tenders {
    int id PK
  }
  quotes {
    int id PK
  }
```

---

## 4. Поставщики

Связи-авторы смены статусов (`*_changed_by`, `decided_by`, `performed_by`, `created_by`) с `users` на диаграмме не показаны.

```mermaid
erDiagram
  suppliers ||--o{ supplier_contacts : "supplier_id"
  users |o--o| supplier_contacts : "user_id"
  suppliers ||--o{ supplier_users : "supplier_id"
  users ||--o| supplier_users : "user_id"
  suppliers ||--o{ supplier_qualification_history : "supplier_id"
  suppliers ||--o{ supplier_status_events : "supplier_id"
  suppliers ||--o{ supplier_work_types : "supplier_id"
  work_types ||--o{ supplier_work_types : "work_type_id"
  suppliers ||--o{ supplier_revenues : "supplier_id"
  suppliers ||--o{ supplier_work_experience : "supplier_id"
  work_types ||--o{ supplier_work_experience : "work_type_id"
  suppliers ||--o{ supplier_checks : "supplier_id"
  suppliers |o--o{ supplier_registration_invitations : "supplier_id"
  suppliers ||..o{ attachments : "entity_type=supplier (логич.)"
  supplier_registration_invitations |o..o{ suppliers : "source_invitation_id (логич.)"
  suppliers {
    int id PK
    int sb_status_changed_by FK
    int legal_status_changed_by FK
    int finance_status_changed_by FK
    int pool_changed_by FK
    varchar inn UK
    varchar legal_name
    varchar status
    varchar overall_status
    varchar pool
    bool is_deleted
  }
  supplier_contacts {
    int id PK
    int supplier_id FK
    int user_id FK, UK
    text full_name
    bool is_primary
  }
  supplier_users {
    int id PK
    bigint user_id FK, UK
    bigint supplier_id FK
    bool is_active
  }
  supplier_qualification_history {
    bigint id PK
    int supplier_id FK
    int decided_by FK
    varchar prev_status
    varchar new_status
    timestamptz decided_at
  }
  supplier_status_events {
    bigint id PK
    int supplier_id FK
    int user_id FK
    varchar kind
    varchar new_status
  }
  supplier_work_types {
    int supplier_id PK, FK
    int work_type_id PK, FK
  }
  supplier_revenues {
    int id PK
    int supplier_id FK
    int year
    numeric amount
  }
  supplier_work_experience {
    int id PK
    int supplier_id FK
    int work_type_id FK
    varchar customer_name
    numeric amount_thousand_rub
  }
  supplier_checks {
    int id PK
    bigint supplier_id FK
    bigint performed_by FK
    varchar check_type
    varchar result
  }
  supplier_registration_invitations {
    int id PK
    bigint created_by FK
    bigint supplier_id FK
    varchar token_hash UK
    varchar mode
    varchar status
  }
  users {
    int id PK
  }
  work_types {
    int id PK
  }
  attachments {
    int id PK
  }
```

---

## 5. Тендеры (ядро)

### 5.1. Тендер и его состав

Связи-авторы с `users` и ссылки пакета документов (`doc_*_attachment_id`) на `attachments` не показаны.

```mermaid
erDiagram
  budget_positions |o--o{ tenders : "budget_position_id"
  projects ||--o{ tenders : "project_id"
  suppliers |o--o{ tenders : "winner_supplier_id"
  work_types |o--o{ tenders : "work_type_id"
  tenders ||--o{ tender_work_types : "tender_id"
  work_types ||--o{ tender_work_types : "work_type_id"
  tender_sections |o--o{ tender_sections : "parent_section_id"
  tenders ||--o{ tender_sections : "tender_id"
  tender_sections ||--o{ tender_items : "section_id"
  tenders ||--o{ tender_items : "tender_id"
  tenders ||--o{ tender_condition_fields : "tender_id"
  tenders ||--o{ tender_fkp_versions : "tender_id"
  tenders ||--o| tender_protocols : "tender_id"
  tenders ||--o{ activity_log : "tender_id"
  tenders {
    int id PK
    bigint project_id FK
    bigint work_type_id FK
    bigint budget_position_id FK
    bigint responsible_user_id FK
    bigint winner_supplier_id FK
    bigint doc_tz_attachment_id FK
    bigint doc_smr_attachment_id FK
    bigint doc_surety_attachment_id FK
    bigint staff_notes_by FK
    varchar tender_number UK
    varchar title
    varchar status
    int tender_revision
  }
  tender_work_types {
    bigint tender_id PK, FK
    bigint work_type_id PK, FK
  }
  tender_sections {
    int id PK
    bigint tender_id FK
    bigint parent_section_id FK
    varchar name
    varchar code
    int order_index
  }
  tender_items {
    int id PK
    bigint tender_id FK
    bigint section_id FK
    varchar name
    varchar unit
    numeric quantity
    bool is_removed
  }
  tender_condition_fields {
    int id PK
    bigint tender_id FK
    varchar field_name
    varchar field_type
  }
  tender_fkp_versions {
    int id PK
    bigint tender_id FK
    bigint author_id FK
    int version_number
    timestamptz notified_at
  }
  tender_protocols {
    bigint tender_id PK, FK
    bigint updated_by FK
    jsonb data
  }
  activity_log {
    int id PK
    bigint tender_id FK
    bigint user_id FK
    varchar action
    timestamptz created_at
  }
  projects {
    int id PK
  }
  work_types {
    int id PK
  }
  budget_positions {
    int id PK
  }
  suppliers {
    int id PK
  }
```

### 5.2. Приглашения, КП, раунды

Одно КП на приглашение (`quotes.tender_invitation_id` уникален). Строка КП ссылается на позицию тендера; у новой позиции поставщика ссылка пуста. Номер версии ФКП в версии КП (`fkp_version_number`) сопоставляется с `tender_fkp_versions.version_number` того же тендера логически.

```mermaid
erDiagram
  suppliers ||--o{ tender_invitations : "supplier_id"
  tenders ||--o{ tender_invitations : "tender_id"
  tender_invitations ||--o| invite_quote_drafts : "invitation_id"
  tender_invitations ||--o| quotes : "tender_invitation_id"
  quotes ||--o{ quote_versions : "quote_id"
  quote_versions ||--o{ quote_items : "quote_version_id"
  tender_items |o--o{ quote_items : "tender_item_id"
  tender_condition_fields ||--o{ quote_condition_values : "condition_field_id"
  quote_versions ||--o{ quote_condition_values : "quote_version_id"
  tenders ||--o{ tender_rounds : "tender_id"
  quote_versions |o--o{ tender_round_recipients : "quote_version_id"
  tender_rounds ||--o{ tender_round_recipients : "round_id"
  suppliers ||--o{ tender_round_recipients : "supplier_id"
  tender_invitations {
    int id PK
    bigint tender_id FK
    bigint supplier_id FK
    uuid invite_token UK
    varchar status
  }
  invite_quote_drafts {
    bigint invitation_id PK, FK
    jsonb payload
  }
  quotes {
    int id PK
    bigint tender_invitation_id FK, UK
    int created_by FK
    varchar status
  }
  quote_versions {
    int id PK
    bigint quote_id FK
    int created_by FK
    int version_number
    bool is_current
    int fkp_version_number
  }
  quote_items {
    int id PK
    bigint quote_version_id FK
    bigint tender_item_id FK
    numeric materials_price
    numeric works_price
    numeric quantity_override
    varchar review_status
  }
  quote_condition_values {
    int id PK
    bigint quote_version_id FK
    bigint condition_field_id FK
    text value_text
  }
  tender_rounds {
    int id PK
    bigint tender_id FK
    bigint created_by FK
    bigint closed_by FK
    int round_number
    varchar status
    timestamptz deadline_at
  }
  tender_round_recipients {
    int id PK
    bigint round_id FK
    bigint supplier_id FK
    bigint quote_version_id FK
    bool included
    timestamptz responded_at
  }
  tenders {
    int id PK
  }
  suppliers {
    int id PK
  }
  tender_items {
    int id PK
  }
  tender_condition_fields {
    int id PK
  }
```

### 5.3. Шаблоны и черновики мастера создания

Шаблон и тендер связаны логически в обе стороны (тендер создан из шаблона; шаблон снят с тендера). Черновик мастера ссылается на задание разбора Excel и исходный файл; после создания тендера — на результат.

```mermaid
erDiagram
  users ||--o{ tender_creation_drafts : "created_by"
  background_jobs |o--o{ tender_creation_drafts : "job_id"
  tenders |o--o{ tender_creation_drafts : "result_tender_id"
  attachments |o--o{ tender_creation_drafts : "source_attachment_id"
  tender_templates |o..o{ tenders : "template_source_id (логич.)"
  tenders |o..o{ tender_templates : "source_tender_id (логич.)"
  tender_templates {
    int id PK
    varchar name
    int version
  }
  tender_creation_drafts {
    uuid id PK
    bigint created_by FK
    uuid job_id FK
    bigint result_tender_id FK
    bigint source_attachment_id FK
    varchar status
    smallint current_step
  }
  users {
    int id PK
  }
  tenders {
    int id PK
  }
  background_jobs {
    uuid id PK
  }
  attachments {
    int id PK
  }
  project_work_type_configs {
    int id PK
  }
```

---

## 6. Бюджет

Бюджетная позиция (дерево) → бюджет на год (одна строка на позицию × год) → история изменений суммы. Тендер ссылается на бюджетную позицию; факт освоения не хранится.

```mermaid
erDiagram
  budget_positions |o--o{ budget_positions : "parent_id"
  budget_positions ||--o{ budget_period_allocations : "budget_position_id"
  budget_period_allocations ||--o{ budget_allocation_history : "allocation_id"
  users |o--o{ budget_allocation_history : "changed_by"
  budget_positions |o--o{ tenders : "budget_position_id"
  budget_positions {
    int id PK
    bigint parent_id FK
    varchar name
    varchar code
  }
  budget_period_allocations {
    int id PK
    bigint budget_position_id FK
    int period_year
    numeric allocated_amount
    varchar status
  }
  budget_allocation_history {
    int id PK
    bigint allocation_id FK
    bigint changed_by FK
    numeric old_amount
    numeric new_amount
  }
  tenders {
    int id PK
  }
  users {
    int id PK
  }
```

---

## 7. Коммуникации: комментарии, чат, уведомления

Комментарии привязаны к сущности полиморфно (`entity_type`, `entity_id`: тендер, КП, позиция, контрагент); для треда ячейки сравнения добавлен контрагент. Отметки прочтения в чате — на пользователя или (для поставщика без учётной записи) на приглашение: ровно одно из двух.

```mermaid
erDiagram
  attachments |o--o{ comments : "attachment_id"
  users |o--o{ comments : "author_id"
  users |o--o{ comments : "notify_user_id"
  comments |o--o{ comments : "parent_comment_id"
  suppliers |o--o{ comments : "supplier_id"
  comments ||--o{ comment_reads : "comment_id"
  users ||--o{ comment_reads : "user_id"
  quote_items ||--o{ position_comment_reads : "quote_item_id"
  users ||--o{ position_comment_reads : "user_id"
  suppliers ||--o{ chat_threads : "supplier_id"
  tenders ||--o{ chat_threads : "tender_id"
  attachments |o--o{ chat_messages : "attachment_id"
  users |o--o{ chat_messages : "author_user_id"
  chat_threads ||--o{ chat_messages : "thread_id"
  comments |o--o{ chat_messages : "comment_id"
  quote_items |o--o{ chat_messages : "quote_item_id"
  chat_messages |o--o{ chat_messages : "reply_to_id"
  attachments |o--o{ chat_message_revisions : "attachment_id"
  chat_messages ||--o{ chat_message_revisions : "message_id"
  users |o--o{ chat_message_revisions : "user_id"
  tender_invitations |o--o{ chat_thread_reads : "invitation_id"
  chat_threads ||--o{ chat_thread_reads : "thread_id"
  users |o--o{ chat_thread_reads : "user_id"
  suppliers |o--o{ app_notifications : "supplier_id"
  users |o--o{ app_notifications : "user_id"
  comments {
    int id PK
    bigint supplier_id FK
    bigint parent_comment_id FK
    bigint author_id FK
    bigint attachment_id FK
    bigint notify_user_id FK
    varchar entity_type
    bigint entity_id
    text comment_text
  }
  comment_reads {
    bigint user_id PK, FK
    bigint comment_id PK, FK
    timestamptz read_at
  }
  position_comment_reads {
    bigint user_id PK, FK
    bigint quote_item_id PK, FK
    timestamptz read_at
  }
  chat_threads {
    int id PK
    bigint tender_id FK
    bigint supplier_id FK
    bigint msgs_from_supplier
    bigint msgs_from_specialist
  }
  chat_messages {
    int id PK
    bigint thread_id FK
    bigint author_user_id FK
    bigint attachment_id FK
    bigint reply_to_id FK
    bigint quote_item_id FK
    bigint comment_id FK
    varchar author_role
    bigint event_seq UK
    varchar kind
    timestamptz deleted_at
  }
  chat_message_revisions {
    bigint id PK
    bigint message_id FK
    bigint attachment_id FK
    bigint user_id FK
    varchar action
  }
  chat_thread_reads {
    bigint id PK
    bigint thread_id FK
    bigint user_id FK
    bigint invitation_id FK
    bigint last_read_seq
    bigint read_count
  }
  app_notifications {
    int id PK
    bigint user_id FK
    bigint supplier_id FK
    varchar audience
    varchar kind
    bool is_read
  }
  users {
    int id PK
  }
  suppliers {
    int id PK
  }
  tenders {
    int id PK
  }
  attachments {
    int id PK
  }
  quote_items {
    int id PK
  }
  tender_invitations {
    int id PK
  }
```

---

## 8. ИИ

Сохранённые разборы СТ, кэш коротких пояснений и разговоры с ИИ привязаны к тендеру (разговоры — ещё и к пользователю). Учёт обращений к модели — таблицы биллинга `ai_operations`/`ai_calls` (показаны и здесь, и в §2.2).

```mermaid
erDiagram
  users |o--o{ ai_analysis_results : "created_by"
  tenders ||--o{ ai_analysis_results : "tender_id"
  tenders ||--o{ ai_insight_cache : "tender_id"
  tenders ||--o{ ai_chat_threads : "tender_id"
  users ||--o{ ai_chat_threads : "user_id"
  tenders ||--o{ ai_chat_messages : "tender_id"
  ai_chat_threads |o--o{ ai_chat_messages : "thread_id"
  users ||--o{ ai_chat_messages : "user_id"
  billing_clients ||--o{ ai_operations : "client_id"
  users |o--o{ ai_operations : "user_id"
  billing_clients ||--o{ ai_calls : "client_id"
  ai_operations |o--o{ ai_calls : "operation_id"
  tenders |o..o{ ai_operations : "tender_id (логич.)"
  ai_analysis_results {
    int id PK
    int tender_id FK
    int created_by FK
    varchar mode
    varchar source
    varchar snapshot_hash
  }
  ai_insight_cache {
    bigint id PK
    bigint tender_id FK
    varchar mode
    varchar snapshot_hash
  }
  ai_chat_threads {
    bigint id PK
    bigint tender_id FK
    bigint user_id FK
    varchar title
  }
  ai_chat_messages {
    bigint id PK
    bigint tender_id FK
    bigint user_id FK
    bigint thread_id FK
    varchar role
    varchar tool
  }
  ai_operations {
    bigint id PK
    bigint client_id FK
    bigint user_id FK
    varchar purpose
    varchar status
    varchar op_key
  }
  ai_calls {
    bigint id PK
    bigint client_id FK
    bigint operation_id FK
    varchar purpose
    varchar model
    numeric cost_rub
  }
  tenders {
    int id PK
  }
  users {
    int id PK
  }
  billing_clients {
    bigint id PK
  }
```

---

## 9. Безопасность

Инцидент создаётся правилом по событиям журнала; список событий инцидента — JSON-массив id (`related_event_ids`). Выгрузка в SIEM ведёт курсор по id журнала. Правила техлогов и состояние форвардера с бизнес-таблицами не связаны.

```mermaid
erDiagram
  users |o--o{ security_incidents : "assignee_id"
  users |o--o{ security_incident_timeline : "author_id"
  security_incidents ||--o{ security_incident_timeline : "incident_id"
  users |o--o{ rkn_packages : "generated_by"
  security_incidents ||--o{ rkn_packages : "incident_id"
  users |o--o{ log_pattern_rules : "created_by"
  security_incident_rules |o..o{ security_incidents : "source_rule_id (логич.)"
  security_audit_log }o..o{ security_incidents : "related_event_ids (JSON)"
  security_audit_log ||..o| siem_settings : "last_sent_event_id (курсор)"
  security_incident_rules {
    int id PK
    varchar action
    varchar severity
    bool enabled
  }
  security_incidents {
    int id PK
    bigint assignee_id FK
    varchar title
    varchar severity
    varchar status
  }
  security_incident_timeline {
    bigint id PK
    bigint incident_id FK
    bigint author_id FK
    varchar kind
  }
  rkn_packages {
    bigint id PK
    int incident_id FK
    int generated_by FK
    int version
  }
  security_audit_log {
    int id PK
    varchar action
    varchar entity_type
    bigint entity_id
    timestamptz created_at
  }
  siem_settings {
    int id PK
    bool enabled
    bigint last_sent_event_id
  }
  log_pattern_rules {
    bigint id PK
    bigint created_by FK
    varchar level
    text match
    bool enabled
  }
  log_forward_state {
    int id PK
    bigint last_sent_seq
  }
  users {
    int id PK
  }
```

---

## 10. Платформа, файлы, настройки

Вложения и незавершённые загрузки привязаны к сущностям полиморфно; при подтверждении прямой загрузки строка `pending_uploads` превращается в строку `attachments`. Однострочные настройки (`app_settings`, `smtp_settings`), кэш тяжёлых чтений и счётчики лимитов связей не имеют.

```mermaid
erDiagram
  users |o--o{ background_jobs : "created_by"
  users |o--o{ mail_templates : "updated_by"
  users ||--o{ bug_reports : "user_id"
  pending_uploads |o..o| attachments : "подтверждение (confirm)"
  attachments {
    int id PK
    varchar entity_type
    bigint entity_id
    varchar file_path
    varchar storage_backend
  }
  pending_uploads {
    uuid id PK
    varchar entity_type
    bigint entity_id
    varchar file_path
  }
  background_jobs {
    uuid id PK
    bigint created_by FK
    varchar job_type
    varchar status
  }
  heavy_read_cache {
    text key PK
    timestamptz expires_at
  }
  rate_limit_counters {
    text key PK
    int hits
    timestamptz expires_at
  }
  app_settings {
    int id PK
    bool require_questionnaire_work_experience
  }
  smtp_settings {
    int id PK
    varchar host
    bool mail_paused
  }
  mail_templates {
    varchar code PK
    bigint updated_by FK
    text subject
  }
  bug_reports {
    bigint id PK
    bigint user_id FK
    varchar status
    varchar user_status
  }
  users {
    int id PK
  }
  tender_creation_drafts {
    uuid id PK
  }
```
