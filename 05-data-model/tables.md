# Модель данных: таблицы

Полное описание схемы базы данных платформы (PostgreSQL): общие соглашения, сводный перечень всех 82 таблиц с доменом-владельцем, по каждой таблице — колонки, ключи, индексы, ограничения, перечисления/статусы и особенности; служебные объекты БД и объёмы данных демо-стенда. ER-диаграммы по доменным группам — в [erd.md](erd.md).

Связанные разделы: [архитектура](../07-architecture/architecture.md) (фоновые задания, живые события, файлы, кэш), [безопасность](../07-architecture/security.md), [масштабирование](../07-architecture/scalability.md), бизнес-логика доменов — `../02-business-logic/<домен>.md` (ссылки у каждой таблицы), [API (OpenAPI)](../04-api/openapi.json). Открытые вопросы раздела — `../_oq/DATA.md`.

Факты собраны из декларативной схемы БД, SQL-миграций и ORM-моделей текущей реализации; значения перечислений сверены с кодом и с данными демо-стенда (октябрь 2026). Имена таблиц и колонок даны как в текущей реализации — исполнитель может их сохранить (рекомендуется: на них ссылаются остальные разделы ТЗ) или переименовать, сохранив смысл и ограничения.

Нумерация требований раздела: **DM-NN**. Оценка воссоздания модели данных целиком (DDL, миграции, начальные данные, триггеры, полевое шифрование): **L**.

---

## 1. Общие соглашения

**1.1. СУБД и схема.** PostgreSQL 18 (стенд — 18.6), одна схема `public`, расширений сверх стандартных нет (`gen_random_uuid()` — встроенная функция). Всё общее состояние системы — в БД: очередь заданий, лимиты частоты, кэш тяжёлых чтений, шаг MFA, шина живых событий (`LISTEN/NOTIFY`, таблиц не требует).

- **DM-01.** Схема ведётся версионированными миграциями: декларативное описание схемы + автоматически сгенерированные SQL-миграции, а то, что декларативно не выражается (триггеры, `UNLOGGED`, именованная последовательность), — ручными SQL-миграциями. Миграции применяются автоматически при старте бэкенда до приёма запросов, поэтому дамп БД старше кода доводится до актуальной схемы. Миграции только добавляются (append-only), уже применённые не редактируются. **M**
- **DM-02.** Схема БД и модели слоя доступа к данным должны оставаться синхронными; рекомендуется автотест сверки (в текущей реализации каждая таблица имеет ORM-модель, а наборы именованных индексов схемы и моделей сверяются тестом).

**1.2. Ключи.**

| Вид первичного ключа | Таблиц | Где |
|---|---|---|
| `serial` (int4, автоинкремент) | 44 | Большинство бизнес-таблиц (исторически) |
| `bigserial` (int8, автоинкремент) | 17 | Новые таблицы: биллинг, чат-ревизии, журналы |
| `uuid` | 4 | `background_jobs`, `auth_sessions`, `pending_uploads`, `tender_creation_drafts` — id отдаётся клиенту и не должен угадываться перебором |
| Составной из двух id | 7 | Связки «многие-ко-многим» и отметки прочтения: `tender_work_types`, `supplier_work_types`, `user_portfolio_access`, `user_project_access`, `comment_reads`, `position_comment_reads`, `role_permissions` (`role_id, code`) |
| Естественный текстовый | 5 | `heavy_read_cache.key`, `rate_limit_counters.key`, `login_throttle.key`, `mail_templates.code`, `unit_aliases.alias_key` |
| Внешний ключ как PK (1:1) | 2 | `invite_quote_drafts.invitation_id`, `tender_protocols.tender_id` |
| `integer` без автоинкремента, одна строка `id = 1` | 3 | `siem_settings`, `log_forward_state` (с CHECK `id = 1`), `billing_owner_settings` |

- Ссылающиеся колонки чаще имеют тип `bigint` при `serial`-ключе родителя (97 из 133 FK) — это допустимо. **DM-03 (рекомендация):** при воссоздании использовать единый тип ключей `bigint` (`GENERATED … AS IDENTITY`) для всех суррогатных ключей; UUID сохранить там, где id виден клиенту.
- Внешних ключей — 133: `NO ACTION` — 90 (удаление родителя запрещено при наличии ссылок), `CASCADE` — 25, `SET NULL` — 18. `ON UPDATE` везде `NO ACTION`.
- **Однострочные таблицы настроек** (`id = 1`): `app_settings`, `smtp_settings`, `billing_owner_settings` (строка создаётся приложением при первом обращении/старте), `siem_settings`, `log_forward_state` (защищены CHECK `id = 1`).
- **Логические ссылки без FK** — колонки-авторы и связи, которые должны переживать удаление/архивацию родителя или образуют цикл: `created_by` у портфелей, проектов, тендеров, шаблонов, правил инцидентов; `security_audit_log.user_id`; `ai_calls.user_id/tender_id`; `ai_operations.tender_id`; `tenders.template_source_id`; `tender_templates.source_tender_id`; `suppliers.source_invitation_id` (FK дал бы цикл); `quotes.revision_requested_by`; `tender_invitations.invited_by`; `attachments.uploaded_by`, `pending_uploads.uploaded_by`; `security_incidents.source_rule_id/created_by`; `ai_analysis_results.round_id`; гранты `user_portfolio_access`/`user_project_access` (FK нет совсем). Целостность таких ссылок обеспечивает приложение.
- **Полиморфные связи** (`entity_type` + `entity_id`, без FK): `attachments`, `pending_uploads`, `comments`, `security_audit_log`, `activity_log`.

**1.3. Время.**

- **DM-04.** Все моменты времени — `timestamptz`, хранятся в UTC; `created_at` (и аналоги: `uploaded_at`, `invited_at`, `decided_at`, `checked_at`…) — `DEFAULT now()`. Даты без времени — `date` (сроки тендера, даты сбора КП, дата госрегистрации, «не приглашать до»).
- `updated_at` есть у 25 таблиц: `DEFAULT now()` при вставке, обновляется **приложением** при изменении строки (триггеров `updated_at` нет).
- Отображение — **московское время** (UTC+3, без перехода на летнее время). Где время рисует клиент, API отдаёт ISO-8601 в UTC; где строку формирует сервер (выгрузки Excel/Word, письма, имена файлов, тексты ответов) — время переводится в МСК единым модулем форматирования.
- Исключения текущей реализации: `role_permissions.granted_at` — `timestamp` без часового пояса; `supplier_work_experience.start_date/end_date` — строки `ММ.ГГГГ` (месяц и год). При воссоздании — `timestamptz` и `date` (первое число месяца) соответственно.

**1.4. Деньги и количества.**

- **DM-05.** Денежные суммы — только `numeric`, без `float`: `numeric(15,2)` — цены КП за единицу, плановая сумма тендера, лимиты, запас торга; `numeric(18,2)` — бюджет, выручка, суммы опыта работ (в тыс. ₽ — `amount_thousand_rub`). Объёмы — `numeric(15,4)`. Биллинг ИИ: себестоимость `numeric(14,8)` USD и `numeric(16,6)` ₽, курс `numeric(12,4)`, множитель `numeric(10,4)`; токены — `bigint`. Пороги аномалий — `numeric(6,2)` (проценты/коэффициенты).
- Валюта одна — рубль, в данных не хранится. Округление денег — до копейки, половина вверх; итоги строк КП не хранятся, а вычисляются (см. [`quote_items`](#quote_items)).
- `suppliers.rating` — `double precision`, не используется.

**1.5. JSONB.** 28 колонок `jsonb` — для данных с гибкой структурой: подробности журналов (`details`), слепки (версии ФКП, шаблоны, снимки для ИИ), настройки пользователя (`notification_prefs`, `lk_prefs`), черновики форм (мастер тендера, форма КП, протокол), постановка/результат фоновых заданий, условия правил инцидентов. Схема каждого JSON описана у колонки. Часть JSON в `ai_analysis_results` хранится текстом (`varchar`) и списки id — строкой через запятую (историческое решение; при воссоздании допустимо `jsonb`/массивы).

**1.6. Мягкое удаление и архив.**

| Таблица | Механизм |
|---|---|
| `suppliers` | `is_deleted` + `deleted_at`; реестр читает только живых |
| `portfolios`, `projects` | `is_archived` («удаление» = архивирование; история и ссылки журнала целы) |
| `users` | Удаления нет — деактивация `is_active = false` |
| `chat_messages` | `deleted_at` (строка остаётся в ленте, прежний текст — в ревизии) |
| `tender_items` | `is_removed` (позиция снята из формы, остаётся для истории версий) |
| `supplier_users` | `is_active` (доступ в ЛК отозван) |
| Справочники (`work_types`, `units`, `budget_positions`, `status_definitions`) | `is_active` |
| `tender_creation_drafts` | `status = 'discarded'` |

Физически удаляются: шаблоны тендеров, файлы по ссылке-приглашению до подачи, разделы/позиции при пересборке структуры тендера, служебные записи по сроку (см. 1.9).

**1.7. Полевое шифрование и хеши.**

- **DM-06.** Персональные данные и секреты, которые не участвуют в SQL-поиске/фильтрации/сортировке, шифруются на уровне приложения прозрачно для бизнес-кода (тип колонки `text`): **M**

| Таблица.колонка | Что | Причина |
|---|---|---|
| `suppliers.director_full_name` | ФИО руководителя | ПДн |
| `suppliers.phone`, `suppliers.email` | Контакты организации | ПДн |
| `supplier_contacts.full_name`, `.phone`, `.email` | Представитель контрагента | ПДн |
| `users.mfa_secret` | Секрет TOTP | Аутентификационный секрет |
| `smtp_settings.password` | Пароль SMTP | Секрет |
| `siem_settings.auth_header` | Заголовок авторизации SIEM | Секрет |
| `supplier_registration_invitations.token` | Токен ссылки на анкету | Дамп БД не должен давать рабочих ссылок |

- Алгоритм текущей реализации — Fernet (AES-128-CBC + HMAC-SHA256), ключ — переменная окружения `FIELD_ENCRYPTION_KEY` (значения по умолчанию нет; известное dev-значение отвергается проверкой безопасной конфигурации при старте). Формат шифротекста: `v1:<провайдер>:<данные>` — версия и метка провайдера для будущей ротации ключей и замены на сертифицированное СКЗИ (интерфейс провайдера выделен). Значение без префикса читается как открытый текст (наследие). Нерасшифровываемое значение (чужой ключ) приложение видит как пустую строку и при пересохранении записи возвращает в колонку без изменений; служебный скрипт-отчёт показывает такие записи.
- **Сознательно не шифруются** ИНН и ОГРН (поиск и автопроверки).
- Секреты одноразовых ссылок хранятся **хешем SHA-256** (сам токен не хранится): `password_reset_tokens.token_hash`, `login_challenges.token_hash`, `auth_sessions.refresh_hash`/`prev_refresh_hash`, `supplier_registration_invitations.token_hash`. Пароли — **bcrypt** (`users.password_hash`), история 5 последних хешей. Токен ссылки-приглашения на форму КП (`tender_invitations.invite_token`) — случайный UUID в открытом виде.

**1.8. Журналы только на дозапись (append-only).**

- **DM-07.** `security_audit_log` и `supplier_qualification_history` защищены на уровне БД: триггеры `BEFORE UPDATE` и `BEFORE DELETE` на каждую строку вызывают функцию, выбрасывающую исключение. Триггеры и функции создаются ручной SQL-миграцией. **S**
- По правилу приложения (без триггеров) только дописываются: `billing_entries` (ошибка исправляется корректировкой), `supplier_status_events`, `chat_message_revisions`, `budget_allocation_history`, `activity_log`, `security_incident_timeline`.

**1.9. UNLOGGED-таблицы и очистка по сроку.**

- **DM-08.** `heavy_read_cache` и `rate_limit_counters` создаются как `UNLOGGED` (не пишутся в журнал WAL, быстрее; после аварийного рестарта СУБД очищаются — для кэша и счётчиков окна это допустимо; на реплики не попадают). Признак задаётся ручной SQL-миграцией.
- Очистку выполняют периодические задачи фонового воркера (подробно — [архитектура, §3.4](../07-architecture/architecture.md#34-периодические-задачи)):

| Задача | Период | Что удаляет/стирает |
|---|---|---|
| `shared-store-purge` | 1 ч | Просроченные строки `rate_limit_counters`, `heavy_read_cache` |
| `auth-purge` | 1 ч | `auth_sessions`, истёкшие/отозванные > 7 дней назад; `login_challenges` через 1 ч после истечения; `login_throttle` с истёкшим сроком |
| `pending-uploads-cleanup` | 1 ч | `pending_uploads` старше 24 ч вместе с объектом в хранилище |
| `ai-prompt-purge` | 6 ч | Тексты `ai_calls.prompt_text/response_text` старше 30 дней (числа остаются) |

- Таблицы без очистки (растут неограниченно): `security_audit_log`, `activity_log`, `app_notifications`, `background_jobs`, `ai_analysis_results`, `ai_insight_cache`, `chat_*`. Политика хранения — см. открытые вопросы.

**1.10. Индексы — соглашения.** Префиксы имён: `idx_` — обычный, `uq_`/`ux_` — уникальный индекс, `ix_` — обычный (в новых модулях); ограничения уникальности `<таблица>_<колонка>_key`, CHECK — `ck_…`/`…_chk`. Всего 120 индексов, из них 43 частичных (`WHERE … IS NOT NULL`, `WHERE NOT is_deleted`, `WHERE status = 'queued'` и т.п.) и 12 уникальных, плюс 17 ограничений уникальности и 11 CHECK. На nullable-FK заводятся частичные индексы, чтобы удаление родителя не сканировало таблицу. Полнотекстовый поиск — один GIN-индекс `to_tsvector('russian', body)` по сообщениям чата.

**1.11. Перечисления.** Статусы и виды хранятся строками `varchar` (без типов `ENUM` PostgreSQL); допустимые значения проверяет приложение, CHECK есть только у `tender_creation_drafts.status` и таблиц биллинга. Это позволяет добавлять значения без миграции. Перевод и смысл значений — у каждой таблицы ниже. Колонка «По умолчанию» показывает значение по умолчанию в БД; пометка *(прил.)* — значение подставляет приложение при вставке, в БД умолчания нет.

**1.12. Начальные данные.** Миграции создают: роли и матрицу прав по умолчанию (см. [`roles`](#roles), [`role_permissions`](#role_permissions)). Клиент биллинга и строка настроек владельца создаются приложением при старте; справочники (виды работ, единицы, статусы, бюджетные позиции) и демонстрационное наполнение стенда загружаются из дампа демо-данных. Учётная запись владельца площадки и роль `owner` создаются служебной консольной командой.

---

## 2. Сводная таблица таблиц

Колонка «Домен-владелец» — модуль бэкенда, которому принадлежит таблица (ORM-модель и логика); «группа раздела» — где таблица описана ниже, если отличается.

| № | Таблица | Домен-владелец | Назначение |
|---|---|---|---|
| 1 | [security_audit_log](#security_audit_log) | audit | Защищённый журнал событий безопасности (append-only) |
| 2 | [security_incident_rules](#security_incident_rules) | audit | Правила автосоздания инцидентов ИБ по событиям журнала |
| 3 | [security_incidents](#security_incidents) | audit | Инциденты информационной безопасности |
| 4 | [users](#users) | identity | Учётные записи всех пользователей (сотрудники, поставщики, лиды, владелец) |
| 5 | [roles](#roles) | identity | Справочник ролей |
| 6 | [role_permissions](#role_permissions) | identity | Матрица прав «роль × действие» (выданные галочки) |
| 7 | [supplier_users](#supplier_users) | identity | Привязка пользователя ЛК к контрагенту |
| 8 | [auth_sessions](#auth_sessions) | identity | Сессии входа (refresh-токены по устройствам) |
| 9 | [login_challenges](#login_challenges) | identity | Шаг второго фактора между паролем и сессией |
| 10 | [login_throttle](#login_throttle) | identity | Нарастающая пауза входа по паре «учётка + IP» |
| 11 | [password_reset_tokens](#password_reset_tokens) | identity | Одноразовые токены восстановления пароля и активации |
| 12 | [billing_clients](#billing_clients) | billing | Клиент установки для биллинга ИИ (баланс токенов) |
| 13 | [billing_owner_settings](#billing_owner_settings) | billing | Настройки владельца площадки (курс USD, реквизиты оплаты); одна строка |
| 14 | [ai_operations](#ai_operations) | billing | ИИ-операции — единицы списания токенов |
| 15 | [ai_calls](#ai_calls) | billing | Журнал всех обращений к ИИ-модели (стоимость, токены) |
| 16 | [topup_requests](#topup_requests) | billing | Заявки клиента на пополнение баланса токенов |
| 17 | [billing_entries](#billing_entries) | billing | Записи баланса токенов (списания, начисления, корректировки) |
| 18 | [work_types](#work_types) | catalog | Справочник видов работ (дерево) |
| 19 | [units](#units) | catalog | Справочник единиц измерения |
| 20 | [unit_aliases](#unit_aliases) | catalog | Запомненные написания единиц измерения («м3» → «м³») |
| 21 | [status_definitions](#status_definitions) | catalog | Пользовательские статусы тендеров и КП |
| 22 | [portfolios](#portfolios) | portfolios | Портфели (верхний уровень дерева объектов) |
| 23 | [projects](#projects) | portfolios | Проекты (объекты) и подпроекты |
| 24 | [user_portfolio_access](#user_portfolio_access) | portfolios | Гранты доступа пользователя на портфель |
| 25 | [user_project_access](#user_project_access) | portfolios | Гранты доступа пользователя на проект |
| 26 | [app_notifications](#app_notifications) | notifications | Центр уведомлений (колокольчик) сотрудников и поставщиков |
| 27 | [attachments](#attachments) | files | Метаданные вложений (файлы в хранилище) |
| 28 | [suppliers](#suppliers) | suppliers | Контрагенты (поставщики/подрядчики): карточка, анкета, статусы проверок |
| 29 | [supplier_contacts](#supplier_contacts) | suppliers | Контактные лица (представители) контрагента — ПДн |
| 30 | [supplier_qualification_history](#supplier_qualification_history) | suppliers | История решений по квалификации контрагента (append-only) |
| 31 | [supplier_status_events](#supplier_status_events) | suppliers | Лента смен частных статусов контрагента |
| 32 | [supplier_work_types](#supplier_work_types) | suppliers | Специализация контрагента: связь с видами работ |
| 33 | [supplier_revenues](#supplier_revenues) | suppliers | Выручка контрагента по годам |
| 34 | [supplier_work_experience](#supplier_work_experience) | suppliers | Опыт работ контрагента (объекты-референсы) |
| 35 | [supplier_checks](#supplier_checks) | suppliers | Результаты автопроверок контрагента по открытым реестрам |
| 36 | [supplier_registration_invitations](#supplier_registration_invitations) | suppliers | Ссылки-приглашения на публичную анкету саморегистрации |
| 37 | [tenders](#tenders) | tenders | Тендеры (шапка, статус, победитель, пакет документов) |
| 38 | [tender_work_types](#tender_work_types) | tenders | Виды работ тендера (многие-ко-многим) |
| 39 | [tender_sections](#tender_sections) | tenders | Разделы ФКП тендера (дерево) |
| 40 | [tender_items](#tender_items) | tenders | Позиции тендера (строки ФКП) |
| 41 | [tender_condition_fields](#tender_condition_fields) | tenders | Дополнительные условия тендера (вопросы подвала ФКП) |
| 42 | [tender_fkp_versions](#tender_fkp_versions) | tenders | Версии шаблона ФКП тендера (слепки структуры) |
| 43 | [tender_invitations](#tender_invitations) | tenders | Приглашения контрагентов в тендер (ссылка на форму КП) |
| 44 | [invite_quote_drafts](#invite_quote_drafts) | tenders | Черновик формы КП по ссылке-приглашению |
| 45 | [quotes](#quotes) | tenders | Коммерческие предложения (КП) поставщиков |
| 46 | [quote_versions](#quote_versions) | tenders | Версии (редакции) КП |
| 47 | [quote_items](#quote_items) | tenders | Строки КП: цены по позициям, корректировки, новые позиции |
| 48 | [quote_condition_values](#quote_condition_values) | tenders | Ответы КП на дополнительные условия |
| 49 | [tender_rounds](#tender_rounds) | tenders | Раунды запроса КП (переторжка/уточнение) |
| 50 | [tender_round_recipients](#tender_round_recipients) | tenders | Адресаты раунда и их ответ |
| 51 | [tender_templates](#tender_templates) | tenders | Шаблоны тендеров (структура для повторного использования) |
| 52 | [tender_creation_drafts](#tender_creation_drafts) | tenders | Черновики мастера создания тендера |
| 53 | [project_work_type_configs](#project_work_type_configs) | tenders | Настройки «проект × вид работ»: даты сбора КП и шаблон по умолчанию |
| 54 | [tender_protocols](#tender_protocols) | tenders | Черновик тендерного протокола (один JSON на тендер) |
| 55 | [activity_log](#activity_log) | tenders | Журнал событий тендера (лента) |
| 56 | [budget_positions](#budget_positions) | budget | Бюджетные позиции (статьи, дерево) |
| 57 | [budget_period_allocations](#budget_period_allocations) | budget | Бюджет позиции на год (распределение по периодам) |
| 58 | [budget_allocation_history](#budget_allocation_history) | budget | История изменений бюджета периода |
| 59 | [ai_analysis_results](#ai_analysis_results) | insights | Сохранённые результаты анализа СТ (правила и ИИ) |
| 60 | [ai_insight_cache](#ai_insight_cache) | insights | Кэш коротких ИИ-пояснений по снимку тендера |
| 61 | [ai_chat_threads](#ai_chat_threads) | insights | Разговоры специалиста с ИИ по таблице сравнения |
| 62 | [ai_chat_messages](#ai_chat_messages) | insights | Реплики разговора с ИИ |
| 63 | [comments](#comments) | comments | Внутренние комментарии (обсуждения) к тендеру, КП, позиции, контрагенту |
| 64 | [comment_reads](#comment_reads) | comments | Отметки «прочитано» комментариев по пользователям |
| 65 | [position_comment_reads](#position_comment_reads) | comments | Просмотр комментария поставщика к позиции |
| 66 | [chat_threads](#chat_threads) | chat | Чаты «тендер × контрагент» |
| 67 | [chat_messages](#chat_messages) | chat | Сообщения чата |
| 68 | [chat_message_revisions](#chat_message_revisions) | chat | Прежние версии сообщений чата (до правки/удаления) |
| 69 | [chat_thread_reads](#chat_thread_reads) | chat | Персональные отметки «докуда прочитал» в чатах |
| 70 | [security_incident_timeline](#security_incident_timeline) | security | Лента расследования инцидента ИБ |
| 71 | [rkn_packages](#rkn_packages) | security | Версионируемые пакеты сведений для РКН по инциденту |
| 72 | [siem_settings](#siem_settings) | security | Настройка выгрузки журнала ИБ во внешний SIEM; одна строка |
| 73 | [log_pattern_rules](#log_pattern_rules) | security | Правила-паттерны техлогов → оповещение в Telegram |
| 74 | [app_settings](#app_settings) | settings (модель — suppliers) | Общесистемные настройки: анкета, пороги аномалий; одна строка |
| 75 | [smtp_settings](#smtp_settings) | settings (модель — platform) | Настройки исходящей почты (SMTP) и выключатель почты; одна строка |
| 76 | [mail_templates](#mail_templates) | settings (модель — platform) | Пользовательские тексты писем вместо стандартных |
| 77 | [bug_reports](#bug_reports) | bug_reports | Баг-репорты пользователей (связка с внешним трекером) |
| 78 | [background_jobs](#background_jobs) | platform | Очередь фоновых заданий |
| 79 | [heavy_read_cache](#heavy_read_cache) | platform | Кэш готовых ответов тяжёлых чтений (UNLOGGED) |
| 80 | [rate_limit_counters](#rate_limit_counters) | platform | Счётчики ограничения частоты запросов (UNLOGGED) |
| 81 | [pending_uploads](#pending_uploads) | files (технич.); группа раздела — platform | Незавершённые прямые загрузки файлов в S3 (presigned) |
| 82 | [log_forward_state](#log_forward_state) | security (технич.); группа раздела — platform | Состояние форвардера техлогов во внешний приёмник; одна строка |

Служебная таблица инструмента миграций `atlas_schema_revisions` в этот перечень не входит — см. [§4](#4-служебные-объекты-бд-вне-перечня-82-таблиц).

## 3. Таблицы по доменам

Разделы сгруппированы по доменам бэкенда в порядке их зависимостей. У каждой таблицы: назначение, домен-владелец, ссылка на бизнес-логику, колонки (тип, допустимость NULL, значение по умолчанию, описание), первичный и внешние ключи с поведением при удалении, уникальные ограничения, индексы, CHECK, перечисления с переводом и смыслом, триггеры и особенности. Стрелка `→ таблица.колонка` в описании колонки — внешний ключ.

## 3.1. Домен audit — журнал событий ИБ и инциденты

Самый нижний домен: любой другой домен пишет события ИБ через единую функцию аудита. Здесь же — правила и сами инциденты; их ведение (лента расследования, пакеты РКН, SIEM) — в домене security.

### security_audit_log

**Назначение.** Журнал событий информационной безопасности: входы, отказы в доступе, смена ролей и прав, выгрузки отчётов, решения по контрагентам, правки справочников, биллинг и т.п. Источник для экрана «Журнал безопасности», правил инцидентов, выгрузки в SIEM и пакетов для РКН. Пишется только кодом через единую функцию записи события; код события обязан входить в закрытый словарь (см. ниже).

**Домен-владелец:** audit. **Бизнес-логика:** [audit.md](../02-business-logic/audit.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ; также курсор выгрузки в SIEM (`siem_settings.last_sent_event_id`) |
| `user_id` | `bigint` | да |  | Кто совершил действие (логическая ссылка на `users.id`, без FK — запись журнала переживает удаление пользователя). NULL — система или анонимный посетитель |
| `action` | `varchar(100)` | нет |  | Код события из закрытого словаря (77 кодов, см. «Перечисления») |
| `entity_type` | `varchar(100)` | да |  | Тип затронутой сущности: `user`, `supplier`, `tender`, `quote`, `report`, `project`, `portfolio`, `budget_position`, `tender_template`, `role`, `security_incident`, `billing_client` и др. |
| `entity_id` | `bigint` | да |  | Идентификатор затронутой сущности |
| `ip_address` | `inet` | да |  | IP-адрес клиента (тип `inet`) |
| `details` | `jsonb` | да |  | Подробности события в JSON (например, `{поле: {from, to}}` для правки реквизитов, `{added, removed}` для видов работ, `reason` для отказа в файле) |
| `created_at` | `timestamptz` | нет | `now()` | Серверное время события |
| `correlation_id` | `varchar(64)` | да |  | Сквозной идентификатор запроса (заголовок `X-Request-Id`): связывает события одного действия между собой и с техлогами |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_security_audit_log_action_created` | `action, created_at DESC` | Фильтр по коду/категории события (`action IN (…)`) с сортировкой по времени |
| `idx_security_audit_log_correlation` | `correlation_id` | Поиск всех событий одного запроса |
| `idx_security_audit_log_created` | `created_at DESC, id DESC` | Листание журнала с конца без фильтров |
| `idx_security_audit_log_entity` | `entity_type, entity_id` | История событий конкретной сущности |
| `idx_security_audit_log_user_created` | `user_id, created_at DESC` | События конкретного пользователя |

**Перечисления и статусы.**

**Категория, критичность и результат события не хранятся в колонках** — они выводятся из кода `action` по словарю (правка словаря не требует перезаписи append-only журнала). Шкала критичности: `low` → `medium` → `high` → `critical`. Результат (`outcome`) = `failure` для кодов `login_failed`, `account_locked`, `permission_denied`, `session_idle_timeout`, `session_reuse_detected`, `incident_created`, `chat_rate_limited`, `chat_thread_cap_reached`, `file_upload_rejected`; для остальных — `success`. Код вне словаря при чтении показывается как «Неизвестное событие», при записи — отвергается.

| Категория | Коды событий (критичность, если не `low`) |
|---|---|
| Аутентификация и доступ | `login_success` Успешный вход; `login_failed` Неуспешный вход (medium); `account_locked` Блокировка учётной записи (high); `permission_denied` Отказ в доступе (medium); `mfa_enabled` Включена MFA; `mfa_disabled` Отключена MFA (high); `password_changed` Смена пароля; `password_reset_requested` Запрос восстановления пароля; `password_reset_completed` Пароль восстановлен (medium); `initial_password_set` Первый пароль установлен; `mfa_reset` Сброс MFA администратором (high); `session_idle_timeout` Выход по неактивности; `logout` Выход; `sessions_revoked_all` Завершены все сессии (medium); `session_reuse_detected` Повторное использование токена сессии (high) |
| Пользователи и роли | `role_changed` Изменение роли (high); `user_created` Создание пользователя (medium); `lead_submitted` Заявка с лендинга (medium); `user_activation_reissued` Повторная выдача ссылки активации; `user_deactivated` / `user_reactivated` (medium); `project_access_changed` Изменён доступ по проектам (medium); `account_unlocked` Снятие блокировки (medium); `finance_access_changed` Изменено право видеть финансы (high); `role_permissions_changed` Изменены права роли (high) |
| Отчёты и чувствительные просмотры | `report_export` Выгрузка отчёта (medium); `finance_viewed` Просмотр финансового блока (medium) |
| Шифрование и ИБ-операции | `crypto_selftest` Самопроверка шифрования; `incident_created` Создан инцидент ИБ (high); `rkn_draft_exported` Пакет сведений для РКН (high); `mail_switch_changed` Изменён выключатель почты (medium) |
| Контрагенты и тендеры | `supplier_registered`; `supplier_checks_run`; `supplier_qualification_decided` (medium); `anomaly_thresholds_changed` (medium); `supplier_requisites_changed` (medium); `supplier_work_types_changed` (medium); `mail_template_changed` (medium); `participant_access_restored` / `participant_access_revoked` (medium); `winner_non_accredited_confirmed` (medium); `quote_status_changed`; `chat_message_created`; `chat_message_edited`; `chat_message_deleted` (medium); `chat_thread_exported` (medium); `chat_cell_bridged` (medium); `chat_rate_limited` (medium); `chat_thread_cap_reached` (medium); `comment_created`; `file_uploaded`; `file_presigned`; `file_upload_rejected` (medium) |
| ИИ и автоматизация | `ai_prompt_changed` (medium); `ai_analysis_run`; `ai_analysis_transition` |
| Справочники и бюджет | `budget_position_created`; `budget_position_changed`; `budget_transfer_undone` (medium); `budget_allocation_changed` (medium); `work_type_created`; `work_type_changed` |
| Портфели и проекты | `portfolio_created`; `portfolio_changed`; `portfolio_archived` (medium); `project_created`; `project_changed`; `project_archived` (medium) |
| Шаблоны | `template_created`; `template_changed`; `template_deleted` (medium) |
| Биллинг | `billing_entry_created`; `billing_client_updated` (medium); `billing_owner_settings_updated` (medium); `billing_topup_fulfilled` (medium); `billing_topup_rejected` |
| Наблюдаемость | `log_pattern_rule_changed` (medium) |

**Триггеры.** `prevent_update_security_audit_log` (BEFORE UPDATE) и `prevent_delete_security_audit_log` (BEFORE DELETE), FOR EACH ROW, функция `trg_prevent_audit_log_modification()` — выбрасывает исключение «security_audit_log is append-only: UPDATE/DELETE запрещены». Создаются вручную написанной SQL-миграцией (декларативная схема их не описывает).

**Особенности.**

- **Append-only на уровне БД**: правка и удаление строк запрещены триггерами, а не только приложением. Очистки по сроку нет — журнал растёт неограниченно (на демо-стенде ≈9,6 тыс. строк, из них ≈94 % — `session_idle_timeout`); политика хранения/архивации — см. открытые вопросы.
- После записи события синхронно применяются правила `security_incident_rules` (создание/дополнение инцидента).
- Выгрузка во внешний SIEM — периодическая задача `siem-drain` по курсору `siem_settings.last_sent_event_id`.

### security_incident_rules

**Назначение.** Правило «при появлении события `action` (с условиями и порогом N событий за M минут) — создать инцидент или дополнить открытый инцидент этого правила». Настраивается в разделе ИБ.

**Домен-владелец:** audit. **Бизнес-логика:** [audit.md](../02-business-logic/audit.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `action` | `varchar(64)` | нет |  | Код события журнала, на которое реагирует правило (из словаря `security_audit_log.action`; `incident_created` правил не порождает — защита от цикла) |
| `severity` | `varchar(20)` | нет | `'medium'` *(прил.)* | Критичность создаваемого инцидента: `low` / `medium` / `high` / `critical` |
| `title` | `varchar(500)` | нет |  | Название правила; оно же — заголовок создаваемого инцидента (до 500 символов) |
| `enabled` | `boolean` | нет | `true` *(прил.)* | Правило включено |
| `created_by` | `integer` | да |  | Автор правила (логическая ссылка на `users.id`, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `conditions` | `jsonb` | да |  | Плоский И-список условий `[{field, op, value}]` по полям события. Поля: `action`, `entity_type`, `entity_id`, `ip_address`, `user_id`; операции: `eq`, `ne`, `in`. Незнакомое поле/операция — правило не срабатывает |
| `window_minutes` | `integer` | да |  | Окно «N событий за M минут», минуты. Пусто/0 — срабатывать на каждое событие |
| `threshold` | `integer` | да |  | Порог числа событий в окне |
| `recipients` | `jsonb` | да |  | Список дополнительных адресов e-mail для уведомления о новом инциденте (для адресатов, не являющихся пользователями системы — дежурная почта, внешний SOC) |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Перечисления и статусы.**

`severity`: `low` — низкая, `medium` — средняя (по умолчанию в приложении), `high` — высокая, `critical` — критическая.

**Особенности.**

- Пока по правилу есть открытый инцидент, новые совпавшие события дописываются в его `related_event_ids`, второй инцидент не создаётся.
- Внешних ключей нет; на демо-стенде правил нет (0 строк).

### security_incidents

**Назначение.** Инцидент ИБ: создаётся правилом журнала автоматически или оператором вручную; ведётся в разделе ИБ (статус, ответственный, лента расследования, пакеты для РКН).

**Домен-владелец:** audit. **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `title` | `varchar(500)` | нет |  | Заголовок (до 500 символов) |
| `severity` | `varchar(20)` | нет | `'medium'` *(прил.)* | Критичность: `low` / `medium` / `high` / `critical` |
| `status` | `varchar(20)` | нет | `'open'` *(прил.)* | Статус: `open` / `investigating` / `closed` |
| `description` | `text` | да |  | Описание; у автоинцидента — «Создано правилом «…» (#id) по событию #id.» |
| `related_event_ids` | `jsonb` | да |  | JSON-массив id событий `security_audit_log`, относящихся к инциденту |
| `source_rule_id` | `integer` | да |  | Правило, создавшее инцидент (логическая ссылка на `security_incident_rules.id`, без FK); NULL — создан вручную |
| `created_by` | `integer` | да |  | Кто создал (логическая ссылка на `users.id`); у автоинцидента — автор события |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `closed_at` | `timestamptz` | да |  | Время закрытия |
| `assignee_id` | `bigint` | да |  | Ответственный (назначается человеком) → `users.id` |
| `affected_resources` | `jsonb` | да |  | JSON-массив затронутых ресурсов (объекты `{…}`, вводит оператор) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `assignee_id` | `users(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_security_incidents_assignee` | `assignee_id` | Инциденты, назначенные пользователю |

**Перечисления и статусы.**

`status`: `open` — открыт (по умолчанию), `investigating` — расследуется, `closed` — закрыт. `severity` — как у правил.

**Особенности.**

- Создание инцидента публикует доменное событие → письмо ответственным фоновым заданием `security_incident_email` и уведомление в центр уведомлений (тип `security`).
- Смены статуса, назначения и комментарии пишутся в `security_incident_timeline`.

## 3.2. Домен identity — пользователи, роли, права, сессии

Пользователи всех типов, роли и матрица прав, привязка пользователя к контрагенту, сессии на HttpOnly-cookie с ротацией refresh-токена, шаг MFA, защита входа от перебора, одноразовые ссылки.

### users

**Назначение.** Учётная запись пользователя: сотрудник Заказчика, пользователь контрагента (ЛК поставщика), лид с лендинга или владелец площадки. Одна роль на пользователя (`role_id`). Статус учётки (активна / приглашена / заблокирована / отключена) не хранится колонкой, а выводится из полей.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `full_name` | `varchar(255)` | нет |  | ФИО (до 255 символов) |
| `email` | `varchar(255)` | нет |  | E-mail — логин; уникален |
| `phone` | `varchar(20)` | да |  | Телефон (до 20 символов) |
| `password_hash` | `varchar(255)` | нет |  | Хеш пароля (bcrypt). У приглашённого, ещё не задавшего пароль, — заведомо несовпадающая заглушка |
| `password_history` | `jsonb` | нет | `[]` *(прил.)* | JSON-массив последних 5 хешей паролей — запрет повторного использования |
| `role_id` | `integer` | нет |  | Роль пользователя → `roles.id` |
| `source` | `varchar(32)` | да |  | Откуда пришёл: `landing` — заявка с формы демо на лендинге (роль `lead`); NULL — заведён администратором/по приглашению |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Учётка активна; `false` — отключена администратором (вход запрещён) |
| `can_view_finance` | `boolean` | нет | `false` *(прил.)* | Персональное право видеть цены и финансовый блок поверх роли (администраторам ставится при создании) |
| `is_superadmin` | `boolean` | нет | `false` | Владелец площадки (раздел биллинга ИИ `/owner`); ставится только служебной консольной командой, вне ролей клиента |
| `mfa_enabled` | `boolean` | нет | `false` *(прил.)* | Включён второй фактор (TOTP) |
| `mfa_secret` | `text` | да |  | Секрет TOTP — **шифруется** полевым шифрованием (тип колонки `text`) |
| `mfa_last_counter` | `bigint` | да |  | Номер 30-секундного шага последнего принятого TOTP-кода — защита от повторного использования кода |
| `failed_login_attempts` | `integer` | нет | `0` *(прил.)* | Счётчик неудачных входов подряд (порог блокировки — `MAX_FAILED_LOGIN_ATTEMPTS`, 5) |
| `locked_until` | `timestamptz` | да |  | Учётка заблокирована до этого момента; снимает администратор или таймер |
| `last_login_at` | `timestamptz` | да |  | Время последнего успешного входа |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время последнего изменения |
| `last_activity` | `timestamptz` | да |  | Время последнего аутентифицированного запроса — серверный тайм-аут неактивности |
| `notification_prefs` | `jsonb` | нет | `'{}'` | Галочки e-mail-уведомлений сотрудника: `{код_типа: true\|false}`; отсутствие ключа = включено. Коды — из каталога уведомлений (см. `app_notifications`) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `role_id` | `roles(id)` | NO ACTION |

**Уникальные ограничения:** `users_email_key` (email).

**Перечисления и статусы.**

Производный статус учётки (по старшинству): `deactivated` — `is_active=false`; `locked` — `locked_until` в будущем; `invited` — пароль ещё не задан (активация не пройдена); `active` — иначе.

`source`: `landing` — заявка с лендинга; NULL — прочие.

**Особенности.**

- Шифруемое поле: `mfa_secret`.
- Начальные данные: демо-учётки создаются начальной миграцией (без паролей в ТЗ); владелец площадки и его роль `owner` — консольной командой.
- Удаления пользователей нет — только деактивация.

### roles

**Назначение.** Роли пользователей. Набор фиксирован кодом; редактируется только матрица прав ролей (`role_permissions`).

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `code` | `varchar(50)` | нет |  | Код роли (уникален) |
| `name` | `varchar(150)` | нет |  | Название роли для интерфейса |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Уникальные ограничения:** `roles_code_key` (code).

**Перечисления и статусы.**

| `code` | Название | Смысл |
|---|---|---|
| `admin` | Администратор | Полный доступ, матрицу прав обходит |
| `tender_specialist` | Тендерный специалист | Тендеры, КП, портфели; доступ только к выданным портфелям/проектам |
| `security` | Служба безопасности | Реестр контрагентов, очередь проверок СБ, раздел ИБ |
| `lawyer` | Юрист | Реестр контрагентов, очередь юридических проверок |
| `supplier` | Поставщик | Пользователь контрагента (ЛК); матрица прав к нему не применяется |
| `lead` | Лид | Заявка с лендинга; прав нет, ждёт назначения роли |
| `owner` | Владелец площадки | Создаётся консольной командой; доступ решает `users.is_superadmin` |

**Особенности.**

- Начальные данные (миграция): роли `admin`, `tender_specialist`, `supplier`, `security`, `lawyer`, позже `lead`.

### role_permissions

**Назначение.** Одна строка — одно выданное роли право. Список возможных прав — константа в коде (право без проверки в API ничего не даёт); в БД хранятся только выданные.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `role_id` (PK) | `integer` | нет |  | Роль (часть PK); удаление роли каскадно удаляет её права → `roles.id` |
| `code` (PK) | `varchar(50)` | нет |  | Код права (часть PK), см. перечень |
| `granted_at` | `timestamp` | нет | `now()` | Когда выдано. **Тип `timestamp` без часового пояса** — единственное исключение из правила timestamptz; при воссоздании использовать `timestamptz` |

**Первичный ключ:** (`role_id`, `code`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `role_id` | `roles(id)` | CASCADE |

**Перечисления и статусы.**

| Код права | Что разрешает | Начальные данные (кому выдано) |
|---|---|---|
| `portfolios.manage` | Создавать и изменять портфели и объекты | tender_specialist |
| `tenders.manage` | Создавать тендеры и приглашать контрагентов | tender_specialist |
| `quotes.manage` | Обрабатывать КП | tender_specialist |
| `suppliers.manage` | Удалять/восстанавливать контрагентов, выдавать доступ в ЛК | — |
| `suppliers.contacts.view` | Видеть полные ИНН, почту, телефон контрагента (иначе маска) | tender_specialist, security, lawyer |
| `reports.view` | Смотреть и выгружать отчёты | tender_specialist |
| `finance.view` | Видеть цены и финансовые данные | — |
| `budget.view` | Смотреть бюджет | tender_specialist |
| `budget.manage` | Распределять бюджет, корректировки | — |
| `catalogs.manage` | Править справочники | — |
| `users.manage` | Управлять пользователями и доступом к объектам | — |
| `security.manage` | Журнал безопасности, инциденты, РКН | security |
| `settings.manage` | Настройки системы: почта и уведомления | — |
| `billing.view` | Расходы на ИИ: баланс, история, заявки | — |

Строки для `admin` не нужны (обходит матрицу); для `supplier`, `lead`, `owner` строк нет намеренно.

### supplier_users

**Назначение.** Связь «пользователь с ролью `supplier` ↔ контрагент»: отвечает на вопрос «чей это аккаунт» при каждом запросе ЛК поставщика. Один пользователь — не более одного контрагента; у контрагента может быть несколько пользователей.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `user_id` | `bigint` | нет |  | Пользователь (уникален) → `users.id` |
| `supplier_id` | `bigint` | нет |  | Контрагент → `suppliers.id` |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Доступ в ЛК активен; `false` — доступ отозван без удаления связи |
| `created_at` | `timestamptz` | нет | `now()` | Время создания связи |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | NO ACTION |
| `user_id` | `users(id)` | NO ACTION |

**Уникальные ограничения:** `supplier_users_user_id_key` (user_id).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_users_supplier` | `supplier_id` | Пользователи контрагента |

### auth_sessions

**Назначение.** Сессия одного устройства/браузера. Refresh-токен живёт в HttpOnly-cookie, в БД — только его хеш. Токен вращается при каждом обновлении; каждый запрос сверяет идентификатор сессии из access-токена с этой таблицей, поэтому выход и «выйти на всех устройствах» действуют сразу.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `uuid` | нет | `gen_random_uuid()` | UUID сессии (генерируется БД `gen_random_uuid()`); кладётся в claim `sid` access-токена |
| `user_id` | `integer` | нет |  | Владелец сессии; удаление пользователя каскадно удаляет сессии → `users.id` |
| `refresh_hash` | `varchar(64)` | нет |  | SHA-256 текущего refresh-токена (уникален) |
| `prev_refresh_hash` | `varchar(64)` | да |  | Хеш предыдущего refresh-токена — принимается ещё 30 с после ротации (гонка вкладок); позже — признак кражи |
| `rotated_at` | `timestamptz` | да |  | Время последней ротации |
| `created_at` | `timestamptz` | нет | `now()` | Время входа |
| `last_used_at` | `timestamptz` | нет | `now()` | Время последнего обновления |
| `idle_expires_at` | `timestamptz` | нет |  | Скользящий срок: сдвигается при каждом обновлении (`SESSION_IDLE_DAYS`, 14 дней) |
| `expires_at` | `timestamptz` | нет |  | Абсолютный потолок (`SESSION_ABSOLUTE_DAYS`, 30 дней); дальше только новый вход |
| `revoked_at` | `timestamptz` | да |  | Когда сессия закрыта |
| `revoked_reason` | `varchar(32)` | да |  | Причина закрытия (см. перечень) |
| `ip` | `varchar(45)` | да |  | IP при входе (до 45 символов — IPv6) |
| `user_agent` | `varchar(255)` | да |  | User-Agent браузера (до 255 символов) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `user_id` | `users(id)` | CASCADE |

**Уникальные ограничения:** `auth_sessions_refresh_hash_key` (refresh_hash).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_auth_sessions_user` | `user_id` | Сессии пользователя («выйти везде», список устройств) |
| `idx_auth_sessions_prev_refresh_hash` | `prev_refresh_hash` | Поиск по предыдущему хешу — обнаружение повторного предъявления |

**Перечисления и статусы.**

`revoked_reason`: `logout` — выход; `logout_all` — «выйти на всех устройствах»; `password_changed` — смена/первичная установка пароля; `password_reset` — восстановление пароля; `reuse` — повторное предъявление старого refresh-токена (кража); `expired` — истёк срок; `user_inactive` — учётка отключена.

**Особенности.**

- Очистка: периодическая задача `auth-purge` (раз в час) удаляет сессии, истёкшие или отозванные более 7 дней назад.

### login_challenges

**Назначение.** Одноразовый челлендж MFA: создаётся после верного пароля у пользователя с MFA, cookie челленджа ведёт на шаг ввода TOTP-кода. Хранится в БД (а не в памяти процесса), т.к. процессов API несколько.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `user_id` | `integer` | нет |  | Пользователь; каскадное удаление → `users.id` |
| `token_hash` | `varchar(64)` | нет |  | SHA-256 токена челленджа из cookie (уникален) |
| `attempts` | `integer` | нет | `0` | Число неверных кодов (лимит 5) |
| `expires_at` | `timestamptz` | нет |  | Срок действия (5 минут) |
| `consumed_at` | `timestamptz` | да |  | Когда использован (успешный ввод кода) |
| `ip` | `varchar(45)` | да |  | IP клиента |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `user_id` | `users(id)` | CASCADE |

**Уникальные ограничения:** `login_challenges_token_hash_key` (token_hash).

**Особенности.**

- Очистка: `auth-purge` удаляет челленджи через 1 час после истечения.

### login_throttle

**Назначение.** Защита от перебора паролей без блокировки самой учётки: после 5 неудач подряд пара «e-mail + IP» получает паузу 1, 2, 4… минут (до `LOGIN_THROTTLE_MAX_MINUTES`, 60). Чужой адрес не может запереть владельца.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `key` (PK) | `text` | нет |  | Ключ `<email в нижнем регистре>\|<ip>` |
| `failures` | `integer` | нет | `0` | Число неудач подряд |
| `blocked_until` | `timestamptz` | да |  | Пауза до этого момента |
| `expires_at` | `timestamptz` | нет |  | Когда запись можно удалить |

**Первичный ключ:** (`key`).

**Внешние ключи:** нет.

**Особенности.**

- Очистка: `auth-purge` удаляет записи с истёкшим `expires_at`; успешный вход удаляет запись пары.

### password_reset_tokens

**Назначение.** Одноразовая ссылка из письма: восстановление пароля (срок 2 ч) и первичная активация учётки/установка первого пароля (срок 72 ч) — один механизм.

**Домен-владелец:** identity. **Бизнес-логика:** [identity.md](../02-business-logic/identity.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `user_id` | `integer` | нет |  | Пользователь → `users.id` |
| `token_hash` | `varchar(64)` | нет |  | SHA-256 токена из ссылки (уникален); сам токен не хранится |
| `expires_at` | `timestamptz` | нет |  | Срок действия |
| `used_at` | `timestamptz` | да |  | Когда использован; повторно не принимается |
| `created_at` | `timestamptz` | нет | `now()` | Время выдачи |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `user_id` | `users(id)` | NO ACTION |

**Уникальные ограничения:** `password_reset_tokens_token_hash_key` (token_hash).

## 3.3. Домен billing — биллинг ИИ (токены)

Учёт расходов на ИИ в токенах: клиент установки, операции (единицы списания), журнал обращений к модели, заявки на пополнение и книга баланса. Включается `BILLING_ENABLED`; журнал обращений ведётся всегда.

### billing_clients

**Назначение.** Клиент, для которого развёрнута установка; сейчас один на установку (создаётся на старте приложения по `CLIENT_CODE`). Параметры тарификации ИИ задаёт владелец площадки.

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `code` | `varchar(64)` | нет |  | Код клиента (уникален), из настройки `CLIENT_CODE` (по умолчанию `demo`) |
| `name` | `varchar(255)` | нет |  | Название клиента |
| `multiplier` | `numeric(10,4)` | нет | `1` | Множитель списания (> 0): токены = ⌈себестоимость ₽ × 100 × множитель⌉ |
| `block_at_zero` | `boolean` | нет | `false` | Блокировать запуск ИИ-операций при балансе ≤ 0 (ответ «Баланс токенов исчерпан — обратитесь к администратору») |
| `low_balance_threshold` | `bigint` | нет | `1000` | Порог «низкий баланс», токены (по умолчанию 1000) |
| `low_balance_notified` | `boolean` | нет | `false` | Уведомление о низком балансе уже отправлено (сбрасывается, когда баланс снова выше порога) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время последнего изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `uq_billing_clients_code` | `code` | UNIQUE; Уникальность кода клиента |

**CHECK-ограничения:**

- `billing_clients_multiplier_chk`: `(multiplier > (0)::numeric)` — Множитель строго больше нуля

**Особенности.**

- При первом создании клиента начисляется приветственный бонус (`billing_entries.kind='welcome'`), если `BILLING_WELCOME_BONUS` > 0.
- Баланс не хранится — это сумма `billing_entries.tokens` по клиенту.

### billing_owner_settings

**Назначение.** Настройки уровня установки: курс пересчёта себестоимости ИИ из USD в рубли и реквизиты для оплаты пополнений. Ровно одна строка `id = 1` (создаётся на старте).

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `integer` | нет |  | Всегда 1 (одна строка) |
| `usd_rate` | `numeric(12,4)` | нет | `100` | Курс ₽ за 1 USD для пересчёта себестоимости обращений к ИИ (по умолчанию 100) |
| `payment_details` | `text` | нет | `''` | Реквизиты для оплаты пополнения (свободный текст) |
| `updated_at` | `timestamptz` | нет | `now()` | Время последнего изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

### ai_operations

**Назначение.** Одна ИИ-операция = одно действие пользователя (распознавание файла, анализ тендера, разбор пары, письмо поставщику, вопрос к таблице, сопоставление позиций). Внутри операции может быть много обращений к модели (`ai_calls`); по завершении операции её себестоимость списывается одной записью `billing_entries`.

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `client_id` | `bigint` | нет |  | Клиент биллинга → `billing_clients.id` |
| `op_key` | `varchar(100)` | да |  | Ключ идемпотентности: у фонового задания — `job:<id>`, повтор задания продолжает ту же операцию (уникален среди непустых) |
| `purpose` | `varchar(40)` | нет |  | Вид операции (см. перечень) |
| `status` | `varchar(20)` | нет | `'running'` | Статус: `running` / `succeeded` / `failed` |
| `user_id` | `bigint` | да |  | Инициатор; при удалении пользователя — NULL → `users.id` |
| `tender_id` | `bigint` | да |  | Тендер (логическая ссылка, без FK) |
| `round_no` | `integer` | да |  | Номер раунда, если операция относится к раунду |
| `started_at` | `timestamptz` | нет | `now()` | Начало |
| `finished_at` | `timestamptz` | да |  | Окончание |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `client_id` | `billing_clients(id)` | NO ACTION |
| `user_id` | `users(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `uq_ai_operations_op_key` | `op_key` WHERE `(op_key IS NOT NULL)` | UNIQUE; Идемпотентность операции фонового задания |

**CHECK-ограничения:**

- `ai_operations_status_chk`: `((status)::text = ANY ((ARRAY['running'::character varying, 'succeeded'::character varying, 'failed'::character varying])::text[]))` — Допустимые статусы операции

**Перечисления и статусы.**

`purpose`: `file_recognition` — распознавание файла ведомости; `tender_analysis` — анализ тендера; `pair_analysis` — разбор пары подрядчиков; `supplier_letter` — письмо поставщику; `table_question` — вопрос к таблице сравнения; `position_matching` — сопоставление позиций.

`status`: `running` — выполняется (по умолчанию), `succeeded` — успешно, `failed` — ошибка.

### ai_calls

**Назначение.** Каждое обращение к модели (чат-запрос или эмбеддинг) — ведётся всегда, независимо от включённости биллинга. Источник отчётов о расходах на ИИ у владельца и клиента.

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `client_id` | `bigint` | нет |  | Клиент биллинга → `billing_clients.id` |
| `operation_id` | `bigint` | да |  | ИИ-операция; NULL — системное обращение вне операции (себестоимость владельца, без списания) → `ai_operations.id` |
| `purpose` | `varchar(40)` | нет |  | Вид операции (как в `ai_operations.purpose`) или `system` |
| `kind` | `varchar(10)` | нет |  | Тип обращения: `chat` / `embed` |
| `model` | `varchar(120)` | нет |  | Имя модели у провайдера (до 120 символов) |
| `ok` | `boolean` | нет |  | Обращение успешно |
| `error` | `text` | да |  | Текст ошибки |
| `duration_ms` | `integer` | нет |  | Длительность, мс |
| `prompt_tokens` | `integer` | да |  | Токены запроса (по данным провайдера) |
| `output_tokens` | `integer` | да |  | Токены ответа |
| `cost_usd` | `numeric(14,8)` | да |  | Себестоимость, USD (по данным провайдера), 8 знаков после запятой |
| `usd_rate` | `numeric(12,4)` | нет |  | Курс USD на момент обращения (из `billing_owner_settings`) |
| `cost_rub` | `numeric(16,6)` | да |  | Себестоимость, ₽ = `cost_usd × usd_rate` |
| `user_id` | `bigint` | да |  | Инициатор (логическая ссылка, без FK) |
| `tender_id` | `bigint` | да |  | Тендер (логическая ссылка, без FK) |
| `supplier_code` | `varchar(255)` | да |  | Чьё КП читал шаг пайплайна (код/имя участника в снимке), до 255 символов |
| `round_no` | `integer` | да |  | Номер раунда |
| `prompt_text` | `text` | да |  | Текст запроса — сохраняется, только если включено `AI_PROMPT_LOG_ENABLED` |
| `response_text` | `text` | да |  | Текст ответа — на тех же условиях |
| `created_at` | `timestamptz` | нет | `now()` | Время обращения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `client_id` | `billing_clients(id)` | NO ACTION |
| `operation_id` | `ai_operations(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_ai_calls_created` | `created_at DESC` | Журнал обращений по времени |
| `idx_ai_calls_operation` | `operation_id` | Обращения операции |
| `idx_ai_calls_prompt_kept` | `created_at` WHERE `(prompt_text IS NOT NULL)` | Поиск строк с сохранённым текстом для очистки |

**Перечисления и статусы.**

`kind`: `chat` — генерация, `embed` — эмбеддинг. `purpose`: значения `ai_operations.purpose` + `system`.

**Особенности.**

- Очистка: периодическая задача `ai-prompt-purge` (раз в 6 ч) стирает `prompt_text`/`response_text` старше `AI_PROMPT_LOG_DAYS` (30 дней); числовые поля остаются.

### topup_requests

**Назначение.** Заявка на пополнение: оплата происходит вне площадки, решение (зачислить/отклонить) принимает владелец площадки.

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `client_id` | `bigint` | нет |  | Клиент биллинга → `billing_clients.id` |
| `tokens` | `bigint` | нет |  | Запрошенное число токенов (> 0, не более 100 000 000) |
| `comment` | `text` | нет | `''` | Комментарий заявителя |
| `status` | `varchar(20)` | нет | `'new'` | Статус: `new` / `fulfilled` / `rejected` |
| `requested_by` | `bigint` | нет |  | Кто подал → `users.id` |
| `decided_by` | `bigint` | да |  | Кто решил (владелец) → `users.id` |
| `decided_at` | `timestamptz` | да |  | Когда решено |
| `reject_reason` | `text` | да |  | Причина отказа |
| `created_at` | `timestamptz` | нет | `now()` | Время подачи |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `client_id` | `billing_clients(id)` | NO ACTION |
| `requested_by` | `users(id)` | NO ACTION |
| `decided_by` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_topup_requests_client_created` | `client_id, created_at DESC` | Заявки клиента по времени |

**CHECK-ограничения:**

- `topup_requests_status_chk`: `((status)::text = ANY ((ARRAY['new'::character varying, 'fulfilled'::character varying, 'rejected'::character varying])::text[]))` — Допустимые статусы заявки
- `topup_requests_tokens_chk`: `(tokens > 0)` — Число токенов положительно

**Перечисления и статусы.**

`status`: `new` — новая (по умолчанию), `fulfilled` — зачислено (создаётся `billing_entries.kind='topup'`), `rejected` — отклонено. Цена покупки: 100 токенов = 1 ₽.

### billing_entries

**Назначение.** Бухгалтерская книга токенов клиента. Баланс = сумма `tokens`. Записи не правятся и не удаляются — ошибка исправляется корректировкой.

**Домен-владелец:** billing. **Бизнес-логика:** [billing.md](../02-business-logic/billing.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `client_id` | `bigint` | нет |  | Клиент биллинга → `billing_clients.id` |
| `kind` | `varchar(20)` | нет |  | Вид записи (см. перечень) |
| `tokens` | `bigint` | нет |  | Число токенов со знаком: списание и корректировка ≤ 0, начисления > 0 |
| `operation_id` | `bigint` | да |  | ИИ-операция списания (не более одной записи на операцию) → `ai_operations.id` |
| `topup_request_id` | `bigint` | да |  | Заявка на пополнение (для `topup`) → `topup_requests.id` |
| `multiplier` | `numeric(10,4)` | да |  | Множитель клиента на момент списания |
| `cost_rub` | `numeric(16,6)` | да |  | Себестоимость операции, ₽ |
| `comment` | `text` | да |  | Комментарий |
| `created_by` | `bigint` | да |  | Автор ручной записи → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время записи |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `client_id` | `billing_clients(id)` | NO ACTION |
| `operation_id` | `ai_operations(id)` | NO ACTION |
| `topup_request_id` | `topup_requests(id)` | NO ACTION |
| `created_by` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `uq_billing_entries_operation` | `operation_id` WHERE `(operation_id IS NOT NULL)` | UNIQUE; Одно списание на операцию |
| `idx_billing_entries_client_created` | `client_id, created_at DESC` | История баланса клиента |

**CHECK-ограничения:**

- `billing_entries_kind_chk`: `((kind)::text = ANY ((ARRAY['charge'::character varying, 'welcome'::character varying, 'topup'::character varying, 'manual'::character varying, 'correction'::character varying])::text[]))` — Допустимые виды записи
- `billing_entries_sign_chk`: `((((kind)::text = ANY ((ARRAY['charge'::character varying, 'correction'::character varying])::text[])) AND (tokens <= 0)) OR (((kind)::text = ANY ((ARRAY['welcome'::character varying, 'topup'::character varying, 'manual'::character varying])::text[])) AND (tokens > 0)))` — Знак `tokens` соответствует виду: списание/корректировка ≤ 0, начисления > 0

**Перечисления и статусы.**

`kind`: `charge` — списание за ИИ-операцию (≤ 0); `welcome` — приветственный бонус (> 0); `topup` — зачисленное пополнение (> 0); `manual` — ручное начисление владельцем (> 0); `correction` — корректировка владельцем (< 0; вводится положительным числом, пишется со знаком минус).

**Особенности.**

- Фактически журнал только на дозапись (правило приложения, триггера нет).

## 3.4. Домен catalog — справочники

Справочники, общие для всех: виды работ (дерево), единицы измерения и их запомненные написания, пользовательские статусы тендеров и КП.

### work_types

**Назначение.** Виды работ — неограниченная иерархия (`parent_id`). Используются в тендерах, специализации и опыте контрагентов, настройках проектов.

**Домен-владелец:** catalog. **Бизнес-логика:** [catalog.md](../02-business-logic/catalog.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `parent_id` | `bigint` | да |  | Родительский вид работ; NULL — корневая категория → `work_types.id` |
| `name` | `varchar(255)` | нет |  | Название (уникально во всём справочнике) |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Активен (неактивный не предлагается в выборе, ссылки сохраняются) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `parent_id` | `work_types(id)` | NO ACTION |

**Уникальные ограничения:** `work_types_name_key` (name).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_work_types_parent` | `parent_id` WHERE `(parent_id IS NOT NULL)` | Дочерние элементы узла |

### units

**Назначение.** Единицы измерения (шт., м², компл. …) — подсказка и сверка единиц при разборе Excel-ведомости в мастере тендера.

**Домен-владелец:** catalog. **Бизнес-логика:** [catalog.md](../02-business-logic/catalog.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `name` | `varchar(50)` | нет |  | Название единицы (уникально) |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Активна |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Уникальные ограничения:** `units_name_key` (name).

### unit_aliases

**Назначение.** Сопоставление нестандартного написания единицы с записью справочника: выбор оператора или ответ ИИ запоминается и применяется при следующих импортах. Одно сопоставление на всю установку.

**Домен-владелец:** catalog. **Бизнес-логика:** [catalog.md](../02-business-logic/catalog.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `alias_key` (PK) | `varchar(50)` | нет |  | Нормализованный ключ написания (PK): нижний регистр, без точек, пробелов, скобок, `²`/`³` → `2`/`3`, «кв.м»/«м.кв» → `м2`, «куб.м» → `м3`; «м» и «м.п.» остаются разными |
| `unit_id` | `integer` | нет |  | Единица справочника; удаление единицы удаляет её алиасы → `units.id` |
| `created_at` | `timestamptz` | нет | `now()` | Когда запомнено |

**Первичный ключ:** (`alias_key`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `unit_id` | `units(id)` | CASCADE |

### status_definitions

**Назначение.** Дополнительные (пользовательские) статусы-пометки тендеров и КП поверх системных. Код статуса записан строкой в `tenders.custom_status_code` / `quotes.custom_status_code`.

**Домен-владелец:** catalog. **Бизнес-логика:** [catalog.md](../02-business-logic/catalog.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `entity_type` | `varchar(20)` | нет |  | Чей статус: `tender` / `quote` |
| `code` | `varchar(50)` | нет |  | Код (латиница/цифры/`_`, выводится транслитерацией из названия; при коллизии `_2`, `_3`…). **Никогда не меняется и не переиспользуется** |
| `label` | `varchar(255)` | нет |  | Название для интерфейса (переименование — только здесь) |
| `color` | `varchar(30)` | да |  | Смысловой тон: `info` / `success` / `warning` / `danger` / `neutral` (старые hex-значения показываются нейтральным тоном) |
| `is_system` | `boolean` | нет | `false` *(прил.)* | Системный статус (не удаляется) |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Активен; снятый статус остаётся на помеченных сущностях |
| `order_index` | `integer` | нет | `0` *(прил.)* | Порядок в списке |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_status_definitions_entity_code` | `entity_type, code` | Поиск статуса по типу сущности и коду |

**Перечисления и статусы.**

`entity_type`: `tender` — тендер, `quote` — КП. На демо-стенде: по одному статусу каждого типа (например, `clarification` у КП).

**Особенности.**

- Уникальность `(entity_type, code)` обеспечивается приложением (индекс не уникальный).

## 3.5. Домен portfolios — портфели, проекты и доступ

Дерево «портфель → проект (объект) → подпроект» и гранты доступа тендерных специалистов. Тендеры под узлами дерева — домен tenders.

### portfolios

**Назначение.** Портфель — группа проектов (объектов строительства). Верхний уровень дерева «портфель → проект → подпроект».

**Домен-владелец:** portfolios. **Бизнес-логика:** [portfolios.md](../02-business-logic/portfolios.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `name` | `varchar(255)` | нет |  | Название |
| `description` | `text` | да |  | Описание |
| `created_by` | `integer` | да |  | Автор (логическая ссылка на `users.id`, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `is_archived` | `boolean` | нет | `false` | Мягкое удаление: архивный портфель скрыт из выдачи, история и ссылки журнала сохраняются |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_portfolios_archived` | `is_archived` WHERE `(NOT is_archived)` | Частичный индекс по неархивным портфелям |

### projects

**Назначение.** Проект (объект строительства) внутри портфеля; подпроекты — через `parent_project_id` (произвольная глубина). К проекту привязываются тендеры; доступ тендерного специалиста определяется грантами на портфель/проект.

**Домен-владелец:** portfolios. **Бизнес-логика:** [portfolios.md](../02-business-logic/portfolios.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `portfolio_id` | `bigint` | нет |  | Портфель → `portfolios.id` |
| `parent_project_id` | `bigint` | да |  | Родительский проект; NULL — проект верхнего уровня → `projects.id` |
| `name` | `varchar(255)` | нет |  | Название |
| `description` | `text` | да |  | Описание |
| `address` | `varchar(500)` | да |  | Адрес стройплощадки (до 500 символов) |
| `photo_path` | `varchar(500)` | да |  | Ключ фото объекта в файловом хранилище |
| `created_by` | `integer` | да |  | Автор (логическая ссылка, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `is_archived` | `boolean` | нет | `false` | Мягкое удаление (архив) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `parent_project_id` | `projects(id)` | NO ACTION |
| `portfolio_id` | `portfolios(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_projects_archived` | `is_archived` WHERE `(NOT is_archived)` | Частичный индекс по неархивным |
| `idx_projects_parent` | `parent_project_id` WHERE `(parent_project_id IS NOT NULL)` | Подпроекты |
| `idx_projects_portfolio` | `portfolio_id` | Проекты портфеля |

### user_portfolio_access

**Назначение.** Грант на весь портфель — даёт доступ ко всем его проектам и подпроектам.

**Домен-владелец:** portfolios. **Бизнес-логика:** [portfolios.md](../02-business-logic/portfolios.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `user_id` (PK) | `integer` | нет |  | Пользователь (часть PK; логическая ссылка, FK нет) |
| `portfolio_id` (PK) | `integer` | нет |  | Портфель (часть PK; логическая ссылка, FK нет) |
| `granted_at` | `timestamptz` | нет | `now()` | Когда выдан |

**Первичный ключ:** (`user_id`, `portfolio_id`).

**Внешние ключи:** нет.

**Особенности.**

- Внешних ключей нет — целостность обеспечивает приложение; при воссоздании допустимо добавить FK с каскадным удалением.
- Правило доступа: явные гранты на проекты + все проекты портфелей из грантов + все потомки доступных проектов (рекурсивно) + путь вверх до корня (для отображения дерева).

### user_project_access

**Назначение.** Грант на конкретный проект (и его подпроекты).

**Домен-владелец:** portfolios. **Бизнес-логика:** [portfolios.md](../02-business-logic/portfolios.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `user_id` (PK) | `integer` | нет |  | Пользователь (часть PK; логическая ссылка, FK нет) |
| `project_id` (PK) | `integer` | нет |  | Проект (часть PK; логическая ссылка, FK нет) |
| `granted_at` | `timestamptz` | нет | `now()` | Когда выдан |

**Первичный ключ:** (`user_id`, `project_id`).

**Внешние ключи:** нет.

**Особенности.**

- Внешних ключей нет (как у `user_portfolio_access`).

## 3.6. Домен notifications — центр уведомлений

Центр уведомлений. Каталог типов уведомлений (кому, какая галочка) — константа в коде, в БД только записи и галочки пользователей.

### app_notifications

**Назначение.** Запись центра уведомлений. Создаётся всегда на каждое событие и каждому получателю (отключить нельзя); e-mail — надстройка, управляемая галочками. Новая запись публикуется в живой поток адресата (realtime) в той же транзакции.

**Домен-владелец:** notifications. **Бизнес-логика:** [notifications.md](../02-business-logic/notifications.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `audience` | `varchar(20)` | нет |  | Аудитория: `staff` (сотрудники) / `supplier` (поставщики) |
| `user_id` | `bigint` | да |  | Персональный адресат; для `staff` NULL — общее для всех сотрудников → `users.id` |
| `supplier_id` | `bigint` | да |  | Контрагент-адресат (для `supplier`) → `suppliers.id` |
| `kind` | `varchar(30)` | нет | `'system'` *(прил.)* | Вид уведомления (см. перечень); по нему же — отключение галочкой ЛК поставщика |
| `title` | `varchar(500)` | нет |  | Заголовок (до 500 символов) |
| `body` | `text` | нет | `''` *(прил.)* | Текст |
| `href` | `varchar(500)` | да |  | Ссылка внутри приложения (deep link), например `/tenders/{id}` или `/chats?tender_id=…&supplier_id=…` |
| `is_read` | `boolean` | нет | `false` *(прил.)* | Прочитано |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | NO ACTION |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_app_notifications_audience_created` | `audience, created_at DESC` | Лента аудитории по времени |
| `idx_app_notifications_audience_user_created` | `audience, user_id, created_at DESC` | Лента сотрудника: свои и общие (`audience='staff' AND (user_id IS NULL OR user_id=…)`) |
| `idx_app_notifications_supplier` | `supplier_id` WHERE `(supplier_id IS NOT NULL)` | Уведомления контрагента |
| `idx_app_notifications_user` | `user_id` WHERE `(user_id IS NOT NULL)` | Уведомления пользователя |

**Перечисления и статусы.**

`audience`: `staff` — сотрудники, `supplier` — поставщики.

`kind`: `system` — системное (по умолчанию); `message` — сообщение чата; `tender` — событие тендера (приглашение, раунд, изменения ФКП); `quote` — КП; `revision` — запрос доработки/дозаполнения КП; `comment` — комментарий («+ Уведомить»); `security` — инцидент ИБ. У поставщика виды `message`, `revision`, `tender` отключаются галочками ЛК (`suppliers.lk_prefs`: `notify_chat`, `notify_revision`, `notify_tender`) — при выключенной галочке запись не создаётся.

Каталог e-mail-галочек сотрудников (ключи `users.notification_prefs`): `questionnaire_submitted_security`, `questionnaire_submitted_lawyer`, `questionnaire_submitted_specialist` (новая анкета — для СБ, юриста, специалиста), `sb_check_closed` (проверка СБ закрыта), `legal_check_closed` (юридическая проверка закрыта), `supplier_checks_completed` (контрагент проверен СБ и юристом — можно принимать решение) — для специалиста, `security_incident_created` (инциденты ИБ, роль security). Администратор видит все галочки.

**Особенности.**

- Очистки по сроку нет — см. открытые вопросы.

## 3.7. Домен files — вложения

Метаданные вложений; сами файлы — в хранилище (локальный диск или S3). Техническая таблица незавершённых прямых загрузок `pending_uploads` описана в группе platform.

### attachments

**Назначение.** Файл, привязанный к сущности другого домена полиморфно (`entity_type`, `entity_id`). Сам файл — в хранилище (локальный диск или S3), здесь — метаданные и ключ объекта.

**Домен-владелец:** files. **Бизнес-логика:** [files.md](../02-business-logic/files.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `entity_type` | `varchar(20)` | нет |  | Тип владельца: `supplier`, `tender`, `quote`, `tender_invite`, `chat_message`, `project`, `creation_draft` |
| `entity_id` | `bigint` | да |  | Id владельца; NULL — ещё не привязано (исходная ведомость черновика мастера; ссылку держит сам черновик) |
| `doc_type` | `varchar(100)` | да |  | Слот документа (см. перечень); NULL — без слота |
| `file_name` | `varchar(500)` | нет |  | Исходное имя файла (до 500 символов) |
| `file_path` | `varchar(1000)` | нет |  | Ключ объекта в хранилище: относительный путь для `local`, ключ в бакете для `s3` (формат одинаковый — перенос диска в бакет ключи не меняет) |
| `content_type` | `varchar(100)` | да |  | MIME-тип |
| `file_size` | `integer` | да |  | Размер, байт |
| `uploaded_by` | `integer` | да |  | Кто загрузил (логическая ссылка, без FK) |
| `uploaded_at` | `timestamptz` | нет | `now()` | Время загрузки |
| `sha256` | `char(64)` | да |  | SHA-256 содержимого (hex, нижний регистр) — тот же, что считает фронт; дедупликация/проверка |
| `storage_backend` | `varchar(10)` | нет | `'local'` | Где лежит: `local` / `s3` |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_attachments_entity` | `entity_type, entity_id` | Файлы сущности (единственный путь к файлам) |
| `idx_attachments_sha256` | `sha256` WHERE `(sha256 IS NOT NULL)` | Поиск по хешу содержимого |

**Перечисления и статусы.**

`entity_type`: `supplier` — документы анкеты контрагента; `tender` — пакет документов тендера; `quote` — файлы КП; `tender_invite` — файлы, загруженные по ссылке-приглашению; `chat_message` — вложение чата (`entity_id` = id треда); `project` — фото/файлы объекта; `creation_draft` — исходный файл мастера создания тендера.

`doc_type` (анкета контрагента): `charter` — Устав; `inn_extract` — Выписка ЕГРЮЛ / свидетельство ИНН; `authority` — документ о полномочиях подписанта (три обязательных); `sro` — допуск СРО; `license_mchs` — лицензия МЧС; `license_minculture` — лицензия Минкультуры; `iso` — сертификат ISO (принимается, но не запрашивается); `other` — прочее. Слоты пакета тендера/КП: `rd_url`, `tz_file` (ТЗ), `smr_contract_file` (шаблон договора СМР), `surety_file` (договор поручительства). Вложения чата: `chat`.

`storage_backend`: `local` — локальный диск, `s3` — S3-совместимое хранилище.

**Особенности.**

- Нет FK на владельца (полиморфная связь). Строка появляется только для принятого и проверенного файла; незавершённые прямые загрузки — в `pending_uploads`.

## 3.8. Домен suppliers — контрагенты и аккредитация

Контрагенты: карточка, контакты (ПДн шифруются), статусная модель аккредитации (оси СБ / юрист / финансы → сводный статус, пул), история решений, специализация, выручка, опыт, автопроверки по реестрам, ссылки на саморегистрацию.

### suppliers

**Назначение.** Карточка контрагента: реквизиты, контакты организации, анкетные данные, единый статус квалификации, три независимые оси проверок (СБ, юрист, финансы), вычисляемый сводный статус аккредитации и рабочий пул. Заводится саморегистрацией по ссылке-приглашению (анкета) или вручную сотрудником.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `legal_name` | `varchar(500)` | нет |  | Полное наименование (до 500 символов) |
| `short_name` | `varchar(255)` | да |  | Краткое наименование |
| `inn` | `varchar(12)` | нет |  | ИНН (10 цифр — юрлицо, 12 — ИП; проверка контрольной суммы). Уникален. **Не шифруется** — участвует в поиске и автопроверках |
| `kpp` | `varchar(20)` | да |  | КПП |
| `ogrn` | `varchar(15)` | да |  | ОГРН (до 15 символов). **Не шифруется** |
| `legal_address` | `varchar(500)` | да |  | Юридический адрес |
| `actual_address` | `varchar(500)` | да |  | Фактический адрес |
| `region` | `varchar(255)` | да |  | Регион |
| `category` | `varchar(255)` | да |  | **Устарело**: вид деятельности строкой; канонический источник — `supplier_work_types`, писать запрещено |
| `director_full_name` | `text` | да |  | ФИО руководителя — **шифруется** (ПДн) |
| `phone` | `text` | да |  | Телефон организации — **шифруется** |
| `email` | `text` | да |  | E-mail организации — **шифруется**; адрес для писем-приглашений |
| `website` | `varchar(255)` | да |  | Сайт |
| `tax_system` | `varchar(100)` | да |  | Система налогообложения (строка, например «ОСН (НДС 20 %)») |
| `revenue_history` | `jsonb` | да |  | Устаревший JSON-список выручки (до появления `supplier_revenues`); показывается только при праве видеть финансы |
| `staff_count` | `integer` | да |  | Численность персонала |
| `experience` | `text` | да |  | Опыт работы — свободный текст |
| `sro` | `varchar(255)` | да |  | Сведения о допуске СРО |
| `iso` | `varchar(255)` | да |  | Сертификат ISO (снят с карточки, колонка сохранена) |
| `license_mchs` | `varchar(255)` | да |  | Лицензия МЧС |
| `license_minculture` | `varchar(255)` | да |  | Лицензия Минкультуры |
| `comment` | `text` | да |  | Комментарий сотрудника |
| `user_tags` | `varchar[]` | да |  | Пользовательские метки (массив строк `varchar[]`) |
| `is_manual` | `boolean` | нет | `false` *(прил.)* | Заведён вручную сотрудником (не через анкету) |
| `status` | `varchar(40)` | нет | `'new'` *(прил.)* | Единый статус квалификации (см. перечень); пишется только через блок решения (с историей) |
| `rating` | `double precision` | да |  | Рейтинг (`double precision`); кодом не используется — резерв |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Активен; `false` — организация приостановлена (вход её пользователей запрещён: «Доступ организации приостановлен») |
| `is_deleted` | `boolean` | нет | `false` *(прил.)* | Мягкое удаление (реестр читает только `NOT is_deleted`) |
| `deleted_at` | `timestamptz` | да |  | Когда удалён |
| `lk_prefs` | `jsonb` | да | `{}` *(прил.)* | Настройки ЛК поставщика: `{notify_chat, notify_revision, notify_tender: bool}` |
| `finance_status` | `varchar(40)` | нет | `'pending'` *(прил.)* | Ось «Финансовый блок» (см. перечень) (в схеме помечено deprecated) |
| `finance_limit_amount` | `numeric(15,2)` | да |  | Лимит на один договор СМР при одобрении с лимитом, ₽ |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `no_invite_until` | `date` | да |  | «Не приглашать повторно до …» — отказ со сроком (обычно +365 дней) вместо чёрного списка |
| `source_invitation_id` | `bigint` | да |  | Ссылка-приглашение, по которой подана анкета (логическая ссылка на `supplier_registration_invitations.id`, FK намеренно нет — иначе цикл FK) |
| `sb_status` | `varchar(40)` | нет | `'pending'` | Ось «Проверка СБ» (см. перечень) |
| `legal_status` | `varchar(40)` | нет | `'pending'` | Ось «Юридическая проверка» (см. перечень) |
| `sb_status_comment` | `text` | да |  | Комментарий к решению СБ |
| `legal_status_comment` | `text` | да |  | Комментарий юриста |
| `sb_status_changed_at` | `timestamptz` | да |  | Когда менялся статус СБ |
| `legal_status_changed_at` | `timestamptz` | да |  | Когда менялся юридический статус |
| `sb_status_changed_by` | `integer` | да |  | Кто менял статус СБ → `users.id` |
| `legal_status_changed_by` | `integer` | да |  | Кто менял юридический статус → `users.id` |
| `accredited_at` | `timestamptz` | да |  | Дата выдачи аккредитации (при отзыве не затирается — период «с — по») |
| `questionnaire_submitted_at` | `timestamptz` | да |  | Когда анкета отправлена |
| `finance_status_comment` | `text` | да |  | Комментарий к финансовому решению |
| `finance_status_changed_at` | `timestamptz` | да |  | Когда менялся финансовый статус |
| `finance_status_changed_by` | `integer` | да |  | Кто менял финансовый статус → `users.id` |
| `overall_status` | `varchar(40)` | нет | `'draft'` | Сводный статус аккредитации — **вычисляется** из трёх осей при каждом их изменении, через API не пишется |
| `pool` | `varchar(40)` | нет | `'none'` | Рабочий пул компании (меняется вручную с обязательным комментарием) |
| `pool_comment` | `text` | да |  | Комментарий к переводу в пул |
| `pool_changed_at` | `timestamptz` | да |  | Когда менялся пул |
| `pool_changed_by` | `integer` | да |  | Кто менял пул → `users.id` |
| `pool_is_manual` | `boolean` | нет | `false` | Пул задан человеком — автоправило «с замечаниями → для переторгов» больше не применяется |
| `accreditation_revoked_at` | `timestamptz` | да |  | Дата снятия аккредитации |
| `founded_at` | `date` | да |  | Дата госрегистрации организации (ЕГРЮЛ/ЕГРИП) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `finance_status_changed_by` | `users(id)` | SET NULL |
| `legal_status_changed_by` | `users(id)` | SET NULL |
| `pool_changed_by` | `users(id)` | SET NULL |
| `sb_status_changed_by` | `users(id)` | SET NULL |

**Уникальные ограничения:** `suppliers_inn_key` (inn).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_suppliers_founded_at` | `founded_at` | Фильтр по сроку работы компании |
| `idx_suppliers_legal_status` | `legal_status` WHERE `(NOT is_deleted)` | Очередь юридических проверок |
| `idx_suppliers_live_status` | `status, id` WHERE `(NOT is_deleted)` | Реестр: `WHERE NOT is_deleted [AND status IN …] ORDER BY id` |
| `idx_suppliers_overall_status` | `overall_status` WHERE `(NOT is_deleted)` | Фильтр реестра и экрана приглашения по сводному статусу |
| `idx_suppliers_pool` | `pool` WHERE `(NOT is_deleted)` | Фильтр по пулу |
| `idx_suppliers_sb_status` | `sb_status` WHERE `(NOT is_deleted)` | Очередь проверок СБ |

**Перечисления и статусы.**

`status` — единый статус квалификации (5 рабочих + 3 решения тендерного комитета):

| Код | Подпись | Смысл |
|---|---|---|
| `new` | Новая анкета | Анкета поступила |
| `questionnaire_pending` | Анкета не заполнена | Ждём анкету |
| `review` | На согласовании | |
| `check` | Проверяется | |
| `revision` | На доработке | Анкета возвращена |
| `approved` | Одобрен | Решение |
| `approved_with_limit` | Одобрен с лимитом | Решение; лимит в `finance_limit_amount` |
| `rejected` | Отклонён | Решение; возможен `no_invite_until` |

`sb_status`, `legal_status` (общая шкала): `not_required` — Не требуется; `pending` — Ожидает (по умолчанию); `in_review` — На проверке; `passed` — Пройдена; `passed_with_remarks` — Пройдена с замечаниями (автоматически переводит в пул «Для переторгов», если пул не задан вручную); `failed` — Не пройдена.

`finance_status`: `pending` — Ожидает; `approved` — Одобрено; `approved_with_limit` — Одобрено с лимитом; `rejected` — Отказ.

`overall_status` (вычисляется по порядку правил): любая ось `failed`/`rejected` → `rejected` (Отклонён); все оси `passed`/`passed_with_remarks`/`not_required`/`approved`/`approved_with_limit` → `accredited` (Аккредитован); заведён вручную и анкета не отправлена → `questionnaire_missing` (Анкета не заполнена); анкета не отправлена и ни одна ось не тронута → `draft` (Черновик анкеты, по умолчанию); иначе → `in_review` (На проверке).

`pool`: `main` — Основной пул; `negotiation` — Для переторгов; `none` — Вне пулов (по умолчанию).

**Особенности.**

- Шифруемые поля: `director_full_name`, `phone`, `email` (тип колонки `text`, шифротекст `v1:<провайдер>:<данные>`). ИНН и ОГРН сознательно не шифруются.
- Мягкое удаление: `is_deleted` + `deleted_at`.
- Каждое изменение оси пишет `supplier_status_events` и `supplier_qualification_history`; переход сводного статуса в/из `accredited` — события `accreditation_granted` / `accreditation_revoked`.

### supplier_contacts

**Назначение.** Представители контрагента. Персональные данные шифруются. Контакт может быть связан с учётной записью пользователя ЛК (не более одного контакта на пользователя).

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `integer` | нет |  | Контрагент → `suppliers.id` |
| `full_name` | `text` | нет |  | ФИО — **шифруется** |
| `job_title` | `varchar(255)` | да |  | Должность |
| `phone` | `text` | да |  | Телефон — **шифруется** |
| `email` | `text` | да |  | E-mail — **шифруется** |
| `is_primary` | `boolean` | нет | `false` *(прил.)* | Основной контакт |
| `user_id` | `integer` | да |  | Связанный пользователь ЛК (уникален; при удалении пользователя — NULL, контакт остаётся) → `users.id` |
| `pdn_consent_given` | `boolean` | нет | `false` *(прил.)* | Дано согласие на обработку ПДн |
| `pdn_consent_at` | `timestamptz` | да |  | Когда дано согласие |
| `pdn_consent_document_ref` | `varchar(500)` | да |  | Редакция текста согласия, под которой оно дано (например, `pdn-consent-v1.0-2026-08-01`) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | NO ACTION |
| `user_id` | `users(id)` | SET NULL |

**Уникальные ограничения:** `supplier_contacts_user_id_key` (user_id).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_contacts_supplier` | `supplier_id` | Контакты контрагента |

**Особенности.**

- Шифруемые поля: `full_name`, `phone`, `email`.

### supplier_qualification_history

**Назначение.** Аудит-след решений тендерного комитета и проверяющих: кто, когда (серверное время), с каким комментарием перевёл квалификацию или ось проверки. Правка задним числом запрещена на уровне БД.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `integer` | нет |  | Контрагент (каскадное удаление) → `suppliers.id` |
| `prev_status` | `varchar(40)` | да |  | Прежнее значение: статус квалификации или `<ось>:<статус>` (например, `sb_status:pending`) |
| `new_status` | `varchar(40)` | нет |  | Новое значение в том же формате (`approved`, `finance_status:approved_with_limit` …) |
| `limit_amount` | `numeric(15,2)` | да |  | Лимит на договор при одобрении с лимитом, ₽ |
| `comment` | `text` | да |  | Комментарий решения |
| `no_invite_until` | `date` | да |  | Срок запрета повторного приглашения (при отказе) |
| `decided_by` | `integer` | да |  | Кто решил (при удалении пользователя — NULL) → `users.id` |
| `decided_at` | `timestamptz` | нет | `now()` | Серверное время решения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `decided_by` | `users(id)` | SET NULL |
| `supplier_id` | `suppliers(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_qualification_history_supplier` | `supplier_id, decided_at DESC` | История контрагента от новых к старым |

**Триггеры.** `prevent_update_supplier_qualification_history` (BEFORE UPDATE) и `prevent_delete_supplier_qualification_history` (BEFORE DELETE), FOR EACH ROW, функция `trg_prevent_qualification_history_modification()` — исключение «supplier_qualification_history is append-only: UPDATE/DELETE запрещены». Создаются SQL-миграцией вручную.

**Особенности.**

- Append-only на уровне БД. Каскадное удаление по `supplier_id` формально объявлено, но триггер DELETE его заблокирует — физическое удаление контрагента с историей невозможно (контрагенты удаляются только мягко).

### supplier_status_events

**Назначение.** Append-only (правилом приложения) лента событий статусной модели: смены осей СБ/юрист/финансы, пула, выдача/отзыв аккредитации, отправка анкеты, завершение автопроверок. Источник вкладки «Активность» карточки.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `integer` | нет |  | Контрагент (каскадное удаление) → `suppliers.id` |
| `kind` | `varchar(40)` | нет |  | Вид события (см. перечень) |
| `old_status` | `varchar(40)` | да |  | Прежнее значение |
| `new_status` | `varchar(40)` | нет |  | Новое значение |
| `comment` | `text` | да |  | Комментарий (для автоматических — пояснение, например «Автоматически: проверка пройдена с замечаниями») |
| `user_id` | `integer` | да |  | Автор; NULL — автоматическое событие → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время события |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | CASCADE |
| `user_id` | `users(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_status_events_supplier` | `supplier_id, created_at DESC` | Лента контрагента от новых к старым |

**Перечисления и статусы.**

`kind`: `sb_status`, `legal_status`, `finance_status` — смена оси проверки; `pool` — смена пула (ручная или автоправило); `accreditation_granted` — аккредитация выдана; `accreditation_revoked` — аккредитация снята; `accreditation_date` — правка даты аккредитации; `questionnaire_submitted` — анкета отправлена; `checks_completed` — автопроверки завершены.

### supplier_work_types

**Назначение.** Связь «многие-ко-многим» контрагента с видами работ (специализация из анкеты или правка сотрудником). Канонический источник видов работ контрагента.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `supplier_id` (PK) | `integer` | нет |  | Контрагент (часть PK, каскадное удаление) → `suppliers.id` |
| `work_type_id` (PK) | `integer` | нет |  | Вид работ (часть PK, каскадное удаление) → `work_types.id` |

**Первичный ключ:** (`supplier_id`, `work_type_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | CASCADE |
| `work_type_id` | `work_types(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_work_types_work_type` | `work_type_id` | Контрагенты по виду работ (подбор участников тендера) |

### supplier_revenues

**Назначение.** Выручка за календарный год (анкета/карточка). Один год у контрагента — одна строка.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `integer` | нет |  | Контрагент (каскадное удаление) → `suppliers.id` |
| `year` | `integer` | нет |  | Год |
| `amount` | `numeric(18,2)` | да |  | Сумма выручки, ₽ |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | CASCADE |

**Уникальные ограничения:** `supplier_revenues_supplier_id_year_key` (supplier_id, year).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_revenues_supplier` | `supplier_id` | Выручка контрагента |

### supplier_work_experience

**Назначение.** Опыт работы по заявленным видам деятельности: заказчик, объект, объём, роль, сроки, сумма. Обязательность в анкете — настройка `app_settings.require_questionnaire_work_experience`.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md). **Комментарий в схеме:** «Опыт работы контрагента по заявленным видам деятельности (эпик 39, П14)».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `integer` | нет |  | Контрагент (каскадное удаление) → `suppliers.id` |
| `work_type_id` | `integer` | нет |  | Вид работ (каскадное удаление) → `work_types.id` |
| `customer_name` | `varchar(500)` | нет |  | Заказчик (до 500 символов) |
| `object_type` | `varchar(255)` | нет |  | Тип объекта |
| `object_address` | `varchar(500)` | нет |  | Адрес объекта |
| `work_scope` | `text` | нет |  | Объём/состав работ (текст) |
| `role` | `varchar(50)` | нет |  | Роль: `генподрядчик` / `подрядчик` (хранится по-русски) |
| `start_date` | `varchar(10)` | нет |  | Месяц и год начала работ строкой в формате `ММ.ГГГГ` |
| `end_date` | `varchar(10)` | да |  | Месяц и год окончания `ММ.ГГГГ`; пусто — договор не завершён (работы идут) |
| `amount_thousand_rub` | `numeric(18,2)` | нет |  | Сумма договора, **тыс. ₽** |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | CASCADE |
| `work_type_id` | `work_types(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_work_experience_supplier` | `supplier_id` | Опыт контрагента |
| `idx_supplier_work_experience_work_type` | `work_type_id` | Опыт по виду работ |

**Перечисления и статусы.**

`role`: `генподрядчик`, `подрядчик` (на входе принимаются и латинские синонимы, например `general_contractor`).

**Особенности.**

- Сроки хранятся строками `ММ.ГГГГ` (точность — месяц); при воссоздании допустим тип `date` (первое число месяца) с тем же форматом ввода/вывода.

### supplier_checks

**Назначение.** Результат автопроверки по внешнему реестру (ИНН/ЕГРЮЛ, СРО, арбитраж, банкротство, чёрные списки). Каждый запуск добавляет строки; актуальна последняя по типу.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `supplier_id` | `bigint` | нет |  | Контрагент → `suppliers.id` |
| `check_type` | `varchar(50)` | нет |  | Тип проверки (см. перечень) |
| `result` | `varchar(20)` | нет | `'pend'` *(прил.)* | Результат (см. перечень) |
| `details` | `jsonb` | да |  | Подробности в JSON: краткое резюме, ссылка для ручной проверки, ответ источника, текст ошибки |
| `performed_by` | `bigint` | да |  | Кто запустил; NULL — автоматически → `users.id` |
| `checked_at` | `timestamptz` | нет | `now()` | Время проверки |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `performed_by` | `users(id)` | NO ACTION |
| `supplier_id` | `suppliers(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_supplier_checks_supplier_type` | `supplier_id, check_type, checked_at DESC` | Последний результат по типу проверки |

**Перечисления и статусы.**

`check_type`: `inn` — ИНН/ЕГРЮЛ (включая контрольную сумму); `sro` — членство в СРО; `arbitrage` — арбитражные дела; `bankruptcy` — банкротство; `blacklist` — реестры недобросовестных поставщиков.

`result`: `ok` — норма; `warn` — внимание; `fail` — нарушение; `pend` — не проверено (по умолчанию); `unavailable` — источник недоступен (ссылка на ручную проверку).

### supplier_registration_invitations

**Назначение.** Уникальная ссылка на публичную анкету регистрации контрагента. Одноразовая (сгорает после первой регистрации) или многоразовая.

**Домен-владелец:** suppliers. **Бизнес-логика:** [suppliers.md](../02-business-logic/suppliers.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `token_hash` | `varchar(64)` | нет |  | SHA-256 токена ссылки (уникален) — по нему проверяется ссылка при открытии формы |
| `status` | `varchar(20)` | нет | `'pending'` *(прил.)* | Статус: `pending` / `used` / `revoked` |
| `mode` | `varchar(20)` | нет | `'single'` *(прил.)* | Режим: `single` / `multi` |
| `use_count` | `integer` | нет | `0` *(прил.)* | Число регистраций по ссылке |
| `created_by` | `bigint` | да |  | Кто выпустил → `users.id` |
| `supplier_id` | `bigint` | да |  | Последний зарегистрировавшийся контрагент → `suppliers.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время выпуска |
| `used_at` | `timestamptz` | да |  | Время (последнего) использования |
| `token` | `text` | да |  | Сам токен — **шифруется** полевым шифрованием (не ПДн; чтобы дамп БД не давал рабочих ссылок), нужен для повторного копирования ссылки; NULL у старых строк |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |
| `supplier_id` | `suppliers(id)` | NO ACTION |

**Уникальные ограничения:** `supplier_registration_invitations_token_hash_key` (token_hash).

**Перечисления и статусы.**

`status`: `pending` — действует (по умолчанию), `used` — использована (для `single`), `revoked` — отозвана. `mode`: `single` — одноразовая (по умолчанию), `multi` — многоразовая.

**Особенности.**

- Шифруемое поле: `token`.

## 3.9. Домен fkp_import — импорт ФКП из Excel

Собственных таблиц нет. Разбор Excel-ведомости (ФКП) выполняется фоновым заданием `parse_excel` ([`background_jobs`](#background_jobs)), результат сохраняется в черновик мастера ([`tender_creation_drafts`](#tender_creation_drafts)); при разборе используются справочники [`units`](#units) и [`unit_aliases`](#unit_aliases), обращения к ИИ учитываются в [`ai_calls`](#ai_calls). Бизнес-логика: [fkp-import.md](../02-business-logic/fkp-import.md).

## 3.10. Домен tenders — тендеры, ФКП, приглашения, КП, раунды

Ядро системы. Агрегат «Тендер»: разделы ФКП → позиции, дополнительные условия, версии шаблона ФКП, приглашения → КП → версии КП → строки КП и ответы на условия, раунды и их адресаты, протокол, журнал событий; а также шаблоны, черновики мастера создания и настройки «проект × вид работ». Схема связей — [erd.md, тендеры](erd.md#5-тендеры-ядро).

### tenders

**Назначение.** Тендер — запрос коммерческих предложений по объекту (проекту) и видам работ. Корень агрегата: разделы → позиции, доп. условия, версии ФКП, приглашения → КП → версии, раунды, протокол, журнал.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `project_id` | `bigint` | нет |  | Проект (объект) → `projects.id` |
| `work_type_id` | `bigint` | да |  | **Устарело** (оставлено для отката): единственный вид работ; источник истины — `tender_work_types` → `work_types.id` |
| `budget_position_id` | `bigint` | да |  | Бюджетная позиция (задаётся в мастере) → `budget_positions.id` |
| `budget_amount` | `numeric(15,2)` | да |  | Плановая сумма тендера, ₽ (факт не хранится — считается из КП) |
| `tender_number` | `varchar(100)` | нет |  | Номер тендера `T-<год>-<id с ведущими нулями до 3 знаков>` (например, `T-2026-105`), уникален |
| `title` | `varchar(500)` | нет |  | Название (до 500 символов) |
| `status` | `varchar(20)` | нет | `'draft'` *(прил.)* | Системный статус (см. перечень) |
| `custom_status_code` | `varchar(50)` | да |  | Пользовательский статус-пометка (код из `status_definitions`, `entity_type='tender'`) |
| `needs_attention` | `boolean` | нет | `false` *(прил.)* | Флаг «требует внимания»: ставится при правке состава ФКП (замена структуры, правка позиций/объёмов) и при подаче КП с предложенным изменением объёма; автоматически не сбрасывается (см. открытые вопросы BL) |
| `responsible_user_id` | `bigint` | да |  | Ответственный тендерный специалист → `users.id` |
| `template_source_id` | `integer` | да |  | Шаблон, из которого создан тендер (логическая ссылка на `tender_templates.id`, без FK) |
| `start_date` | `date` | да |  | Начало сбора КП (дата); также период тендера для бюджета (иначе — `created_at`) |
| `end_date` | `date` | да |  | Срок подачи КП (дата) — основа «соблюдения сроков» |
| `created_by` | `integer` | да |  | Автор (логическая ссылка, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `tender_revision` | `integer` | нет | `1` *(прил.)* | Ревизия тендера: +1 при правке структуры (разделов/позиций/условий), подаче КП и решениях по корректировкам; входит в ключи кэша тяжёлых чтений и отпечатки ИИ-разборов |
| `winner_supplier_id` | `bigint` | да |  | Победитель (фиксируется при завершении тендера) → `suppliers.id` |
| `winner_accreditation_status` | `varchar(40)` | да |  | Сводный статус аккредитации победителя на момент выбора, если он не аккредитован; NULL — аккредитован |
| `closed_at` | `timestamptz` | да |  | Когда тендер завершён или отменён |
| `status_changed_at` | `timestamptz` | нет | `now()` | Когда статус менялся последний раз (колонка реестра) |
| `doc_rd_url` | `varchar(1000)` | да |  | Пакет документов: ссылка на рабочую документацию (РД) |
| `doc_tz_attachment_id` | `bigint` | да |  | Пакет документов: файл ТЗ → `attachments.id` |
| `doc_smr_attachment_id` | `bigint` | да |  | Пакет документов: шаблон договора СМР → `attachments.id` |
| `doc_surety_attachment_id` | `bigint` | да |  | Пакет документов: договор личного поручительства → `attachments.id` |
| `staff_notes` | `text` | да |  | Рабочая заметка специалиста (подрядчикам не отдаётся) |
| `staff_notes_by` | `bigint` | да |  | Автор заметки → `users.id` |
| `staff_notes_at` | `timestamptz` | да |  | Время заметки |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `budget_position_id` | `budget_positions(id)` | NO ACTION |
| `doc_smr_attachment_id` | `attachments(id)` | NO ACTION |
| `doc_surety_attachment_id` | `attachments(id)` | NO ACTION |
| `staff_notes_by` | `users(id)` | NO ACTION |
| `doc_tz_attachment_id` | `attachments(id)` | NO ACTION |
| `project_id` | `projects(id)` | NO ACTION |
| `responsible_user_id` | `users(id)` | NO ACTION |
| `winner_supplier_id` | `suppliers(id)` | NO ACTION |
| `work_type_id` | `work_types(id)` | NO ACTION |

**Уникальные ограничения:** `tenders_tender_number_key` (tender_number).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tenders_budget_position` | `budget_position_id` WHERE `(budget_position_id IS NOT NULL)` | Тендеры бюджетной позиции |
| `idx_tenders_created_at` | `created_at DESC` | Сортировка реестра по дате создания |
| `idx_tenders_project_status` | `project_id, status` | Реестр с ограничением доступа: `project_id IN (…) [AND status]` |
| `idx_tenders_responsible_user` | `responsible_user_id` WHERE `(responsible_user_id IS NOT NULL)` | Тендеры ответственного |
| `idx_tenders_start_date` | `start_date` WHERE `(start_date IS NOT NULL)` | Отбор по периоду |
| `idx_tenders_winner_supplier` | `winner_supplier_id` WHERE `(winner_supplier_id IS NOT NULL)` | Победы контрагента |
| `idx_tenders_work_type` | `work_type_id` WHERE `(work_type_id IS NOT NULL)` | Legacy-фильтр по виду работ |

**Перечисления и статусы.**

`status`: `draft` — Черновик (по умолчанию); `open` — Сбор предложений (переход при первой рассылке приглашений); `closed` — Завершён (терминальный, фиксируется победитель, его КП → `accepted`); `cancelled` — Отменён (терминальный). В терминальных статусах состав тендера заморожен; возврат — только административным восстановлением статуса.

Производный признак «соблюдение сроков» (не хранится): `on_time` — В срок, `late` — С опозданием (завершён позже `end_date` по московскому дню), `overdue` — Просрочен (открыт после `end_date`).

**Особенности.**

- Смена статуса всегда вместе с `status_changed_at`; в `closed`/`cancelled` — также `closed_at`.

### tender_work_types

**Назначение.** Один тендер — один или несколько видов работ. Источник истины вместо `tenders.work_type_id`.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `tender_id` (PK) | `bigint` | нет |  | Тендер (часть PK, каскадное удаление) → `tenders.id` |
| `work_type_id` (PK) | `bigint` | нет |  | Вид работ (часть PK) → `work_types.id` |

**Первичный ключ:** (`tender_id`, `work_type_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | CASCADE |
| `work_type_id` | `work_types(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_work_types_work_type` | `work_type_id` | Тендеры по виду работ |

### tender_sections

**Назначение.** Раздел формы КП (ФКП) тендера; иерархия через `parent_section_id`. Позиции и подразделы внутри раздела могут идти вперемешку (порядок задаёт `items_before`).

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `parent_section_id` | `bigint` | да |  | Родительский раздел; NULL — раздел верхнего уровня → `tender_sections.id` |
| `name` | `varchar(500)` | нет |  | Название раздела (до 500 символов) |
| `order_index` | `integer` | нет | `0` *(прил.)* | Порядок среди разделов |
| `code` | `varchar(100)` | да |  | Номер раздела из ФКП («1.2», «IV»), показывается рядом с названием |
| `unit` | `varchar(50)` | да |  | Единица объёма подраздела из ФКП (справочно, в итоги не входит) |
| `quantity` | `numeric(15,4)` | да |  | Объём подраздела из ФКП (справочно), 4 знака после запятой |
| `items_before` | `integer` | да |  | Сколько позиций родителя стоит выше этого подраздела; NULL — после всех позиций родителя |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `parent_section_id` | `tender_sections(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_sections_tender_order` | `tender_id, order_index, id` | Разделы тендера по порядку |
| `idx_tender_sections_parent` | `parent_section_id` WHERE `(parent_section_id IS NOT NULL)` | Подразделы |

### tender_items

**Назначение.** Позиция (строка работ/материалов) формы КП. Поставщики дают по ней цены материалов и работ за единицу. Снятые позиции остаются (флаг `is_removed`) — для истории версий.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ (в API и ИИ — `work_id`) |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `section_id` | `bigint` | нет |  | Раздел → `tender_sections.id` |
| `name` | `varchar(1000)` | нет |  | Наименование (до 1000 символов) |
| `unit` | `varchar(50)` | да |  | Единица измерения |
| `quantity` | `numeric(15,4)` | да |  | Текущий объём (после принятых корректировок), 4 знака после запятой |
| `original_quantity` | `numeric(15,4)` | да |  | Исходный объём из ФКП — заполняется при первом изменении объёма |
| `order_index` | `integer` | нет | `0` *(прил.)* | Порядок внутри раздела |
| `is_removed` | `boolean` | нет | `false` *(прил.)* | Позиция снята из формы (в КП не заполняется) |
| `is_corrected` | `boolean` | нет | `false` *(прил.)* | Объём позиции изменён (принятая корректировка или правка шаблона) |
| `item_number` | `varchar(100)` | да |  | Номер позиции из ФКП (без хвостовой точки) |
| `budget_code` | `varchar(100)` | да |  | Код бюджета/статьи (атрибут ФКП) |
| `mark` | `varchar(500)` | да |  | Марка |
| `equipment_code` | `varchar(100)` | да |  | Код оборудования |
| `manufacturer` | `varchar(500)` | да |  | Производитель |
| `attrs_json` | `jsonb` | нет | `{}` *(прил.)* | Прочие атрибуты строки ФКП, не имеющие колонок: `{имя: значение}` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `section_id` | `tender_sections(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_items_section` | `section_id` | Позиции раздела |
| `idx_tender_items_tender_order` | `tender_id, order_index, id` | Позиции тендера по порядку |

### tender_condition_fields

**Назначение.** Поле дополнительных условий предложения (срок выполнения, аванс, гарантия и т.п.) — поставщик отвечает в КП (`quote_condition_values`).

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `field_name` | `varchar(255)` | нет |  | Формулировка условия (до 255 символов) |
| `field_type` | `varchar(20)` | нет | `'text'` *(прил.)* | Тип ответа: `text` / `number` / `boolean` / `date` |
| `order_index` | `integer` | нет | `0` *(прил.)* | Порядок |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_condition_fields_tender_order` | `tender_id, order_index, id` | Условия тендера по порядку |

**Перечисления и статусы.**

`field_type`: `text` — текст (по умолчанию); `number` — число; `boolean` — да/нет; `date` — дата. При импорте ФКП тип угадывается по формулировке.

### tender_fkp_versions

**Назначение.** Версия формы КП: слепок разделов, позиций (включая снятые) и условий после каждой значимой правки структуры. КП фиксирует, по какой версии подано (`quote_versions.fkp_version_number`).

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-rounds.md](../02-business-logic/tenders-rounds.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `version_number` | `integer` | нет |  | Номер версии в тендере |
| `author_id` | `bigint` | да |  | Автор правки → `users.id` |
| `change_summary` | `text` | да |  | Краткое описание изменений (служебные «тихие» версии участникам не рассылаются) |
| `snapshot` | `jsonb` | нет | `{}` *(прил.)* | Слепок: `{sections:[{id,name,code,parent_section_id,order_index}], items:[{id,section_id,name,unit,quantity,is_removed,is_corrected,order_index}], condition_fields:[{id,field_name,field_type,order_index}]}` |
| `notified_at` | `timestamptz` | да |  | Когда участники уведомлены об этой версии; NULL — ещё не уведомлены |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `author_id` | `users(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_fkp_versions_tender_number` | `tender_id, version_number DESC` | Последняя версия тендера |

### tender_invitations

**Назначение.** Участие контрагента в тендере: персональная ссылка (magic-link по UUID-токену) на форму КП без учётной записи. Одно КП на приглашение.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-invitations.md](../02-business-logic/tenders-invitations.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `supplier_id` | `bigint` | нет |  | Контрагент → `suppliers.id` |
| `invite_token` | `uuid` | нет |  | UUID-токен ссылки на форму КП (уникален) |
| `status` | `varchar(20)` | нет | `'invited'` *(прил.)* | Стадия участия (см. перечень) |
| `invited_by` | `integer` | да |  | Кто пригласил (логическая ссылка, без FK) |
| `invited_at` | `timestamptz` | нет | `now()` | Время приглашения |
| `corrections_acked_at` | `timestamptz` | да |  | Когда поставщик подтвердил ознакомление с правками шаблона ФКП; сбрасывается при новой правке |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Уникальные ограничения:** `tender_invitations_invite_token_key` (invite_token).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_invitations_supplier_tender` | `supplier_id, tender_id` | Тендеры контрагента |
| `idx_tender_invitations_tender_supplier` | `tender_id, supplier_id` | Участники тендера |

**Перечисления и статусы.**

`status`: `invited` — отправлено (по умолчанию); `opened` — доставлено (открыл ссылку); `filling` — заполняет (сохранён черновик / запрошена доработка); `submitted` — подано; `declined` — отказ (значение предусмотрено, кодом не выставляется); `revoked` — доступ отозван (ссылка не действует; может быть восстановлен).

**Особенности.**

- Повторное приглашение того же контрагента в тот же тендер не создаёт второй строки (правило приложения; индекс не уникальный).

### invite_quote_drafts

**Назначение.** Последнее сохранённое состояние формы КП поставщика до подачи — одно на приглашение. Версий КП не создаёт; сохранение — upsert по PK.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-quotes.md](../02-business-logic/tenders-quotes.md). **Комментарий в схеме:** «Черновик формы КП по invite-токену: один на приглашение, версии КП не создаёт.».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `invitation_id` (PK) | `bigint` | нет |  | Приглашение (PK; каскадное удаление) → `tender_invitations.id` |
| `payload` | `jsonb` | нет |  | Состояние формы в JSON (цены по позициям, условия, комментарии) |
| `updated_at` | `timestamptz` | нет | `now()` | Время последнего сохранения |

**Первичный ключ:** (`invitation_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `invitation_id` | `tender_invitations(id)` | CASCADE |

### quotes

**Назначение.** КП контрагента по тендеру — ровно одно на приглашение. Содержимое — в версиях (`quote_versions`); здесь статус обработки и запрос доработки.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-quotes.md](../02-business-logic/tenders-quotes.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_invitation_id` | `bigint` | нет |  | Приглашение (уникально) → `tender_invitations.id` |
| `status` | `varchar(20)` | нет | `'draft'` *(прил.)* | Статус КП (см. перечень) |
| `custom_status_code` | `varchar(50)` | да |  | Пользовательский статус-пометка (`status_definitions`, `entity_type='quote'`) |
| `revision_comment` | `text` | да |  | Комментарий запроса доработки/дозаполнения |
| `revision_requested_at` | `timestamptz` | да |  | Когда запрошена доработка |
| `revision_requested_by` | `integer` | да |  | Кто запросил (логическая ссылка, без FK) |
| `created_by` | `integer` | да |  | Автор КП (пользователь ЛК); неизменяем. NULL — подано по ссылке без учётки/неизвестен → `users.id` |
| `inbox_viewed_at` | `timestamptz` | да |  | Когда специалист открыл КП во «Входящих» (бейдж «новых») |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |
| `tender_invitation_id` | `tender_invitations(id)` | NO ACTION |

**Уникальные ограничения:** `quotes_tender_invitation_id_key` (tender_invitation_id).

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_quotes_created_by` | `created_by` WHERE `(created_by IS NOT NULL)` | КП автора |
| `idx_quotes_new_inbox` | `inbox_viewed_at` WHERE `((status)::text = 'submitted'::text)` | Счётчик новых поданных КП |
| `idx_quotes_status` | `status` | Фильтр по статусу |

**Перечисления и статусы.**

`status`: `draft` — Черновик (по умолчанию); `submitted` — Подано; `revision_requested` — На доработке; `accepted` — Принят (КП победителя при завершении тендера); `rejected` — Отклонено (целиком, с обязательной причиной); `withdrawn` — Отозвано (есть в интерфейсе, кодом не выставляется). «Поданными» для сравнения и раундов считаются `submitted`, `revision_requested`, `accepted`.

### quote_versions

**Назначение.** Каждая подача КП (первичная, после доработки, в новом раунде) — новая версия; текущая помечена `is_current`. Строки цен и ответы на условия принадлежат версии.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-quotes.md](../02-business-logic/tenders-quotes.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `quote_id` | `bigint` | нет |  | КП → `quotes.id` |
| `version_number` | `integer` | нет |  | Номер версии в КП (1, 2, …) |
| `is_current` | `boolean` | нет | `true` *(прил.)* | Текущая версия (ровно одна на КП — правило приложения) |
| `submitted_at` | `timestamptz` | нет | `now()` | Время подачи |
| `fkp_version_number` | `integer` | да |  | По какой версии ФКП подано (сравнение с `tender_fkp_versions`) |
| `vat_confirmed_at` | `timestamptz` | да |  | Когда поставщик подтвердил «цены указаны с НДС N %» при этой подаче; NULL — подача до введения подтверждения |
| `vat_confirmed_rate` | `integer` | да |  | Подтверждённая ставка НДС, % (настройка `VAT_RATE_PCT`, по умолчанию 22) |
| `created_by` | `integer` | да |  | Кто подал именно эту редакцию → `users.id` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |
| `quote_id` | `quotes(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_quote_versions_created_by` | `created_by` | Версии, поданные пользователем |
| `idx_quote_versions_quote_current` | `quote_id` WHERE `is_current` | Текущая версия КП — самый частый запрос |
| `idx_quote_versions_quote_number` | `quote_id, version_number DESC` | Последняя по номеру версия |
| `idx_quote_versions_submitted` | `submitted_at DESC` | Лента подач по времени |

### quote_items

**Назначение.** Строка версии КП: цены материалов и работ за единицу по позиции тендера, отказ от позиции, предложение другого объёма или новой позиции (корректировки) и решение специалиста по ним, отметки ячейки сравнения.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-quotes.md](../02-business-logic/tenders-quotes.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `quote_version_id` | `bigint` | нет |  | Версия КП → `quote_versions.id` |
| `tender_item_id` | `bigint` | да |  | Позиция тендера; NULL — новая позиция, предложенная поставщиком → `tender_items.id` |
| `is_new_position` | `boolean` | нет | `false` *(прил.)* | Новая позиция от поставщика (корректировка) |
| `is_declined_by_supplier` | `boolean` | нет | `false` *(прил.)* | Поставщик отказался давать цену по позиции (комментарий обязателен; цены = 0) |
| `custom_name` | `varchar(1000)` | да |  | Наименование новой позиции |
| `review_status` | `varchar(20)` | да |  | Решение специалиста по корректировке: `pending` / `accepted` / `rejected` |
| `review_comment` | `text` | да |  | Комментарий решения (обязателен при отклонении) |
| `quantity_override` | `numeric(15,4)` | да |  | Предложенный поставщиком объём вместо объёма тендера (корректировка объёма) |
| `materials_price` | `numeric(15,2)` | нет | `0` *(прил.)* | Цена материалов **за единицу**, ₽ (2 знака) |
| `works_price` | `numeric(15,2)` | нет | `0` *(прил.)* | Цена работ **за единицу**, ₽ (2 знака) |
| `price_basis` | `varchar(20)` | нет | `'unit'` *(прил.)* | База цен: `unit` — за единицу; `legacy_line` — старые данные, цены уже за строку |
| `supplier_comment` | `text` | да |  | Комментарий поставщика к позиции (история по версиям — «комментарий к позиции») |
| `negotiation_potential` | `numeric(15,2)` | да |  | Отметка специалиста: заявленный запас торга, ₽/ед. |
| `awaiting_reply` | `boolean` | нет | `false` *(прил.)* | Отметка специалиста «ждём ответ по ячейке» |
| `refill_required` | `boolean` | нет | `false` | Строка открыта поставщику на дозаполнение (после решения по корректировке объёма); КП при этом остаётся `submitted` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `quote_version_id` | `quote_versions(id)` | NO ACTION |
| `tender_item_id` | `tender_items(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_quote_items_tender_item` | `tender_item_id` WHERE `(tender_item_id IS NOT NULL)` | Строки по позиции (сравнение) |
| `idx_quote_items_version` | `quote_version_id` | Строки версии |

**CHECK-ограничения:**

- `ck_quote_items_review_target`: `((review_status IS NULL) OR (is_new_position = true) OR (quantity_override IS NOT NULL))` — Решение специалиста бывает только у новой позиции или корректировки объёма

**Перечисления и статусы.**

`review_status`: NULL — решения не требуется; `pending` — ждёт решения специалиста; `accepted` — принято (для корректировки объёма объём позиции тендера заменяется на `quantity_override`, `tender_items.is_corrected = true`; новая позиция принимается); `rejected` — отклонено с обязательной причиной (для корректировки объёма строка открывается поставщику на дозаполнение). О решении поставщик получает уведомление.

`price_basis`: `unit` (по умолчанию), `legacy_line`.

**Особенности.**

- Итог строки **не хранится**: материалы = М × объём (до копейки, половина вверх), работы = Р × объём, итог = их сумма; объём = `quantity_override`, если корректировка принята, иначе объём позиции.

### quote_condition_values

**Назначение.** Значение поставщика по полю доп. условий тендера в конкретной версии КП.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-quotes.md](../02-business-logic/tenders-quotes.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `quote_version_id` | `bigint` | нет |  | Версия КП → `quote_versions.id` |
| `condition_field_id` | `bigint` | нет |  | Поле условий тендера → `tender_condition_fields.id` |
| `value_text` | `text` | да |  | Значение (хранится текстом при любом `field_type`) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `condition_field_id` | `tender_condition_fields(id)` | NO ACTION |
| `quote_version_id` | `quote_versions(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_quote_condition_values_version` | `quote_version_id` | Ответы версии |
| `idx_quote_condition_values_field` | `condition_field_id` | Ответы по полю (сравнение условий) |

### tender_rounds

**Назначение.** Раунд — повторный запрос уточнённых КП у выбранных участников с общим сообщением и сроком. Активный раунд — `sent` с максимальным номером; отправка следующего закрывает предыдущий.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-rounds.md](../02-business-logic/tenders-rounds.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `round_number` | `integer` | нет |  | Номер раунда в тендере |
| `status` | `varchar(20)` | нет | `'draft'` *(прил.)* | Статус: `draft` / `sent` / `closed` |
| `message_all` | `text` | да |  | Общее сообщение всем адресатам (по умолчанию — стандартный текст «Просим подать уточнённое КП…») |
| `deadline_at` | `timestamptz` | да |  | Срок подачи в раунде |
| `created_by` | `bigint` | да |  | Автор → `users.id` |
| `sent_at` | `timestamptz` | да |  | Когда отправлен |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `closed_at` | `timestamptz` | да |  | Когда закрыт (вручную или отправкой следующего раунда) |
| `closed_by` | `bigint` | да |  | Кто закрыл вручную → `users.id` |
| `close_reason` | `text` | да |  | Причина ручного закрытия |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `closed_by` | `users(id)` | NO ACTION |
| `created_by` | `users(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_rounds_tender_status_number` | `tender_id, status, round_number DESC` | Активный раунд тендера |

**Перечисления и статусы.**

`status`: `draft` — Черновик (по умолчанию); `sent` — В процессе; `closed` — Завершён.

### tender_round_recipients

**Назначение.** Участник раунда: включён ли, персональный комментарий, доставка письма, ответ (поданная версия КП).

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-rounds.md](../02-business-logic/tenders-rounds.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `round_id` | `bigint` | нет |  | Раунд → `tender_rounds.id` |
| `supplier_id` | `bigint` | нет |  | Контрагент → `suppliers.id` |
| `included` | `boolean` | нет | `true` *(прил.)* | Включён в рассылку раунда |
| `personal_comment` | `text` | да |  | Персональное сообщение адресату |
| `delivered_at` | `timestamptz` | да |  | Когда письмо доставлено (SMTP принял) |
| `delivery_error` | `text` | да |  | Ошибка доставки |
| `responded_at` | `timestamptz` | да |  | Когда участник подал КП в раунде |
| `quote_version_id` | `bigint` | да |  | Версия КП, поданная в ответ на раунд → `quote_versions.id` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `quote_version_id` | `quote_versions(id)` | NO ACTION |
| `round_id` | `tender_rounds(id)` | NO ACTION |
| `supplier_id` | `suppliers(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_tender_round_recipients_round_supplier` | `round_id, supplier_id` | Адресат раунда |
| `idx_tender_round_recipients_supplier` | `supplier_id` | Раунды контрагента |
| `idx_tender_round_recipients_quote_version` | `quote_version_id` WHERE `(quote_version_id IS NOT NULL)` | Раунд по версии КП |

### tender_templates

**Назначение.** Шаблон структуры тендера (разделы, позиции, доп. условия, сопоставление колонок импорта) для быстрого создания нового тендера.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `name` | `varchar(255)` | нет |  | Название (уникально без учёта регистра) |
| `source_tender_id` | `bigint` | да |  | Тендер-источник (логическая ссылка, без FK) |
| `structure_snapshot` | `jsonb` | нет |  | Структура: `{work_type, sections:[{name, items, children}], condition_fields:[{field_name, field_type}], column_map?}`; старые записи — плоский `{positions:[…], conditions}` |
| `created_by` | `integer` | да |  | Автор (логическая ссылка, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `version` | `integer` | нет | `1` | Растёт на каждом сохранении: признак правки и защита от гонки вкладок (оптимистическая блокировка) |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `uq_tender_templates_name` | `lower((name))` | UNIQUE; Уникальность названия без учёта регистра (`lower(name)`) |

### tender_creation_drafts

**Назначение.** Состояние мастера создания тендера до его сохранения: шапка, разобранный состав, подсказки ИИ, ссылка на задание разбора Excel и исходный файл. Позволяет вернуться к мастеру после закрытия вкладки.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md). **Комментарий в схеме:** «Черновик мастера /tenders/new до POST /tenders; дедуп ИИ — по file_hash в background_jobs.».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `uuid` | нет | UUID v4 *(прил.)* | UUID черновика |
| `created_by` | `bigint` | нет |  | Владелец черновика (чужие черновики не видны, кроме администратора) → `users.id` |
| `file_hash` | `char(64)` | да |  | SHA-256 исходного файла — дедупликация ИИ-разбора |
| `file_name` | `text` | да |  | Имя исходного файла |
| `file_size` | `integer` | да |  | Размер файла, байт |
| `job_id` | `uuid` | да |  | Задание разбора Excel (`background_jobs`) → `background_jobs.id` |
| `status` | `varchar(20)` | нет | `'draft'` | Статус: `draft` / `converted` / `discarded` |
| `current_step` | `smallint` | нет | `1` | Текущий шаг мастера (1…) |
| `header` | `jsonb` | нет | `'{}'` | Шапка тендера (JSON): название, проект, виды работ, даты, бюджет… |
| `structure` | `jsonb` | нет | `'{}'` | Разобранный состав (JSON): разделы, позиции, условия |
| `enrich` | `jsonb` | да |  | Подсказки ИИ (JSON) |
| `result_tender_id` | `bigint` | да |  | Созданный тендер (для `converted`) → `tenders.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |
| `source_attachment_id` | `bigint` | да |  | Исходный файл шага 1 (вложение `entity_type='creation_draft'`); при создании тендера перевешивается на тендер → `attachments.id` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |
| `job_id` | `background_jobs(id)` | NO ACTION |
| `result_tender_id` | `tenders(id)` | NO ACTION |
| `source_attachment_id` | `attachments(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `ix_tender_creation_drafts_file_hash` | `file_hash` WHERE `(file_hash IS NOT NULL)` | Поиск готового разбора по хешу файла |
| `ix_tender_creation_drafts_owner_status_updated` | `created_by, status, updated_at DESC` | Список черновиков пользователя |

**CHECK-ограничения:**

- `tender_creation_drafts_status_chk`: `((status)::text = ANY ((ARRAY['draft'::character varying, 'converted'::character varying, 'discarded'::character varying])::text[]))` — Допустимые статусы

**Перечисления и статусы.**

`status`: `draft` — черновик (по умолчанию); `converted` — превращён в тендер; `discarded` — удалён пользователем.

**Особенности.**

- Черновики старше `CREATION_DRAFT_TTL_DAYS` (30 дней с последнего изменения) не показываются в списке; физически не удаляются.

### project_work_type_configs

**Назначение.** Подсказки мастеру создания тендера: даты сбора КП по умолчанию и привязанный шаблон для пары «проект + вид работ».

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `project_id` | `bigint` | нет |  | Проект → `projects.id` |
| `work_type_id` | `bigint` | нет |  | Вид работ → `work_types.id` |
| `collect_from` | `date` | да |  | Начало сбора КП по умолчанию |
| `collect_to` | `date` | да |  | Окончание сбора КП по умолчанию |
| `template_id` | `bigint` | да |  | Шаблон тендера по умолчанию → `tender_templates.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `project_id` | `projects(id)` | NO ACTION |
| `template_id` | `tender_templates(id)` | NO ACTION |
| `work_type_id` | `work_types(id)` | NO ACTION |

**Уникальные ограничения:** `uq_pwtc_project_work_type` (project_id, work_type_id).

### tender_protocols

**Назначение.** Реквизиты протокола заседания, которых нет в системе (номер, дата, место, председатель, голосующие, секретарь, кворум, статьи бюджета, приложения, вопросы, решения, аванс, срок работ), — вводятся перед выгрузкой протокола. Всё не введённое берётся из данных системы и настроек.

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders-documents.md](../02-business-logic/tenders-documents.md). **Комментарий в схеме:** «Черновик тендерного протокола: один JSON-документ на тендер (реквизиты, которых нет в системе).».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `tender_id` (PK) | `bigint` | нет |  | Тендер (PK; каскадное удаление) → `tenders.id` |
| `data` | `jsonb` | нет |  | Документ протокола (JSON, ≤ 64 КБ, строгая схема без лишних полей; пустые поля-переопределения = «сгенерировать из данных») |
| `updated_by` | `bigint` | да |  | Кто сохранил (при удалении пользователя — NULL) → `users.id` |
| `updated_at` | `timestamptz` | нет | `now()` | Время сохранения |

**Первичный ключ:** (`tender_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | CASCADE |
| `updated_by` | `users(id)` | SET NULL |

**Особенности.**

Структура `data`: `meeting_number` (≤50), `meeting_date`, `place` (≤300; пусто — из настройки `PROTOCOL_PLACE`), `topic`, `chairman` (пусто — `PROTOCOL_CHAIRMAN`), `chairman_signature`, `voters[]` (≤30), `secretary`, `quorum`, `budget_lines[] {label, amount}` (≤20), `attachments[]` (≤20; NULL — по умолчанию), `presenter`, `invited_text` / `received_text` / `comparison_text` (переопределения), `questions[] {text, speaker}` (≤30), `advance`, `work_term`, `decision_text`, `decisions[] {text, executor}` (≤20). Строки — до 200 символов, длинные тексты — до 4000.

### activity_log

**Назначение.** Лента событий тендера: приглашения, подачи КП, корректировки, раунды, сообщения, смены статуса. Видна сотрудникам; поставщику — только события по нему и общие (белый список действий и ключей `details`).

**Домен-владелец:** tenders. **Бизнес-логика:** [tenders.md](../02-business-logic/tenders.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `user_id` | `bigint` | да |  | Автор; NULL — система или сам контрагент (для `quote_submitted`, `chat_message`) → `users.id` |
| `action` | `varchar(255)` | нет |  | Код события (см. перечень) |
| `entity_type` | `varchar(50)` | да |  | Тип объекта события: `invitation`, `supplier`, `quote`, `tender`, `quote_item`, `tender_round`, … |
| `entity_id` | `bigint` | да |  | Id объекта |
| `details` | `jsonb` | да |  | Подробности (JSON): `supplier_id`, `round_number`, `version_number`, `from_status`/`to_status`, `comment`, `from_quantity`/`to_quantity` и др. |
| `created_at` | `timestamptz` | нет | `now()` | Время события |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | NO ACTION |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_activity_log_action` | `action` | Фильтр по типу события |
| `idx_activity_log_details_supplier_id` | `((details ->> 'supplier_id'))` | Агрегат событий по контрагенту (выражение `details->>'supplier_id'`) |
| `idx_activity_log_entity` | `entity_type, entity_id` | События объекта |
| `idx_activity_log_tender_recent` | `tender_id, created_at DESC, id DESC` | Лента тендера от новых к старым |
| `idx_activity_log_user` | `user_id` WHERE `(user_id IS NOT NULL)` | События пользователя |

**Перечисления и статусы.**

Группы фильтра ленты и коды `action`:
- **КП** (`quote`): `quote_submitted`, `revision_requested`, `quote_status_changed`, `correction_decided`;
- **Раунды** (`round`): `round_sent`, `round_invited`, `round_deadline_extended`, `round_closed`, `fkp_version_notified`;
- **Сообщения** (`message`): `chat_message`;
- **Тендер** (`tender`): `structure_updated`, `structure_rebuilt`, `documents_updated`, `tender_status_changed`, `winner_non_accredited_confirmed`, `tender_custom_status_changed`, `tender_responsible_changed`, `tender_details_changed`;
- **Участники** (`participant`): `invitation_created`, `invitation_non_approved_confirmed`, `invitation_regenerated`, `participant_revoked`, `participant_access_restored`.

Поставщику видны: действия `structure_updated`, `documents_updated` (общие) и события, где `entity_type='supplier' AND entity_id=<он>` или `details.supplier_id=<он>`; из `details` отдаются только ключи белого списка.

## 3.11. Домен budget — бюджет

Бюджетные позиции (дерево), распределение по годам и история изменений. Факт освоения не хранится — считается по КП тендеров позиции.

### budget_positions

**Назначение.** Статья бюджета (иерархия через `parent_id`). Тендер привязывается к бюджетной позиции с плановой суммой; факт считается по КП.

**Домен-владелец:** budget. **Бизнес-логика:** [budget.md](../02-business-logic/budget.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `name` | `varchar(255)` | нет |  | Название (до 255 символов) |
| `code` | `varchar(64)` | да |  | Код статьи (произвольная строка, например «5.7.3») |
| `parent_id` | `bigint` | да |  | Родительская позиция; NULL — корень → `budget_positions.id` |
| `is_active` | `boolean` | нет | `true` *(прил.)* | Активна |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `parent_id` | `budget_positions(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_budget_positions_parent` | `parent_id` WHERE `(parent_id IS NOT NULL)` | Дочерние позиции |

### budget_period_allocations

**Назначение.** Выделенная сумма по бюджетной позиции на календарный год. Одна строка на пару «позиция × год».

**Домен-владелец:** budget. **Бизнес-логика:** [budget.md](../02-business-logic/budget.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `budget_position_id` | `bigint` | нет |  | Бюджетная позиция → `budget_positions.id` |
| `period_year` | `integer` | нет |  | Год периода |
| `allocated_amount` | `numeric(18,2)` | нет | `0` *(прил.)* | Выделенная сумма, ₽ (≥ 0) |
| `status` | `varchar(32)` | нет | `'open'` *(прил.)* | Статус периода: `open` / `closed` |
| `comment` | `text` | да |  | Комментарий |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `budget_position_id` | `budget_positions(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_budget_period_allocations_period` | `period_year, budget_position_id` | Бюджет года по позициям |
| `uq_budget_period_allocations_position_year` | `budget_position_id, period_year` | UNIQUE; Одна сумма на позицию × год |

**Перечисления и статусы.**

`status`: `open` — открыт (по умолчанию); `closed` — закрыт (сумму менять нельзя: «Закрытый период нельзя менять», пока не переоткрыт).

### budget_allocation_history

**Назначение.** Каждое изменение суммы бюджета периода: было/стало, комментарий, автор.

**Домен-владелец:** budget. **Бизнес-логика:** [budget.md](../02-business-logic/budget.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `allocation_id` | `bigint` | нет |  | Бюджет периода → `budget_period_allocations.id` |
| `old_amount` | `numeric(18,2)` | да |  | Прежняя сумма, ₽ |
| `new_amount` | `numeric(18,2)` | нет |  | Новая сумма, ₽ |
| `comment` | `text` | да |  | Комментарий (по умолчанию «Изменение суммы») |
| `changed_by` | `bigint` | да |  | Кто изменил → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `allocation_id` | `budget_period_allocations(id)` | NO ACTION |
| `changed_by` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_budget_allocation_history_alloc` | `allocation_id, id DESC` | История периода от новых к старым |

## 3.12. Домен analytics — отчёты

Собственных таблиц нет: отчёты (реестр тендеров, КП, контрагентов, финансы) строятся запросами по таблицам доменов tenders, suppliers, budget; каждая выгрузка журналируется в [`security_audit_log`](#security_audit_log) (`report_export`). Бизнес-логика: [analytics.md](../02-business-logic/analytics.md).

## 3.13. Домен insights — анализ СТ и ИИ-разговоры

Сохранённые разборы сравнительной таблицы (на правилах и ИИ), кэш коротких ИИ-пояснений и личные разговоры специалиста с ИИ. Модуль ИИ сам в БД не обращается — получает снимок данных.

### ai_analysis_results

**Назначение.** Сохранённый разбор сравнительной таблицы тендера: детерминированные сценарии «на правилах» и результаты ИИ-разбора (фоновое задание). Хранит снимок входных данных и отпечатки для определения устаревания. Текстовые поля с JSON хранятся как `varchar`, не `jsonb`.

**Домен-владелец:** insights. **Бизнес-логика:** [insights.md](../02-business-logic/insights.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `integer` | нет |  | Тендер → `tenders.id` |
| `mode` | `varchar(64)` | нет |  | Вид разбора / сценарий (см. перечень) |
| `supplier_ids` | `varchar(512)` | нет | `''` *(прил.)* | Срез участников: id контрагентов через запятую (пусто — все) |
| `prompt_title` | `varchar(255)` | нет | `''` *(прил.)* | Название сценария для интерфейса |
| `prompt_scope` | `varchar(32)` | нет | `'system'` *(прил.)* | Область промпта: `system` |
| `prompt_body` | `varchar` | нет | `''` *(прил.)* | Текст пользовательского промпта (сейчас пусто) |
| `source` | `varchar(32)` | нет | `'rules'` *(прил.)* | Источник результата: `rules` — правила / `llm` — модель |
| `result_json` | `varchar` | нет | `'[]'` *(прил.)* | Результат (JSON-строка): список выводов или структурированный `{format: 'structured_v1', summary_pins, sections, insights, summary, disclaimer, round}` |
| `created_by` | `integer` | да |  | Кто запустил → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `work_ids` | `varchar(1024)` | нет | `''` *(прил.)* | Срез позиций: id позиций через запятую |
| `tender_revision` | `integer` | да |  | Ревизия тендера на момент разбора |
| `formula_version` | `varchar(64)` | да |  | Версия формул аналитики |
| `ai_contract_version` | `varchar(32)` | да |  | Версия контракта ответа ИИ |
| `thresholds_json` | `varchar` | нет | `'{}'` *(прил.)* | Пороги аномалий на момент разбора (JSON-строка) |
| `snapshot_hash` | `varchar(64)` | да |  | Хеш снимка входных данных |
| `quotes_stamp` | `varchar(128)` | да |  | Отпечаток состояния КП тендера на момент разбора — устаревание при новом КП |
| `snapshot_json` | `varchar` | нет | `'{}'` *(прил.)* | Снимок входной таблицы (JSON-строка) |
| `validation_status` | `varchar(32)` | нет | `'valid'` *(прил.)* | Результат проверки ответа: `valid` / `partial` / `invalid` |
| `validation_errors` | `varchar` | нет | `'[]'` *(прил.)* | Ошибки проверки (JSON-массив строкой) |
| `processing_profile` | `varchar(32)` | нет | `'off'` *(прил.)* | Куда уходили данные КП: `openrouter` — внешний ИИ-шлюз, `off` — ИИ выключен |
| `round_id` | `integer` | да |  | Раунд (логическая ссылка, без FK) |
| `round_number` | `integer` | да |  | Номер раунда |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_ai_analysis_results_tender_mode` | `tender_id, mode, created_at DESC` | Последний разбор тендера данного вида |

**Перечисления и статусы.**

`mode`: сценарии на правилах — `tender_summary` (Анализ тендера), `important` (Что важно), `bidding` (Точки торгов), `compare` (Сравнить подрядчиков), `breakdown` (Разобрать подрядчика), `why_expensive` (Почему дороже), `why_cheaper` (Почему дешевле), `anomalies` (Риски и аномалии), `rounds` (Сравнить раунды), `selection` (Анализировать выбранное); ИИ-разборы — `tender_analysis` (разбор тендера одним прогоном), `pair_duel` (разбор пары финалистов), `negotiation_agenda` (повестка торга прежней схемы, только чтение старых).

`source`: `rules`, `llm`. `validation_status`: `valid` — корректен; `partial` — частично (деградация ИИ); `invalid` — не прошёл проверку. `processing_profile`: `openrouter`, `off`.

**Особенности.**

- Самая «тяжёлая» таблица на строку (снимки целиком): на демо-стенде 7 строк ≈ 2,7 МБ. Очистки нет — см. открытые вопросы.

### ai_insight_cache

**Назначение.** Готовые короткие разборы, зависящие только от снимка тендера («из чего собрана цена», «условия») — генерация детерминирована, поэтому результат кэшируется по хешу снимка.

**Домен-владелец:** insights. **Бизнес-логика:** [insights.md](../02-business-logic/insights.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `mode` | `varchar(32)` | нет |  | Вид пояснения: `structure` / `terms` |
| `snapshot_hash` | `varchar(64)` | нет |  | Хеш снимка |
| `payload` | `jsonb` | нет | `'{}'` | Готовый ответ (JSON) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `ux_ai_insight_cache_key` | `tender_id, mode, snapshot_hash` | UNIQUE; Одно пояснение на тендер × вид × снимок |

**Перечисления и статусы.**

`mode`: `structure` — из чего собрана цена; `terms` — сравнение условий предложений.

**Особенности.**

- Устаревшие записи не удаляются (новый снимок — новый ключ).

### ai_chat_threads

**Назначение.** Ветка разговора с ИИ по СТ тендера. Личная: привязана к тендеру и пользователю; веток на тендер может быть несколько. Заголовок — первый вопрос.

**Домен-владелец:** insights. **Бизнес-логика:** [insights.md](../02-business-logic/insights.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `user_id` | `bigint` | нет |  | Владелец разговора → `users.id` |
| `title` | `varchar(200)` | нет | `''` | Заголовок (первый вопрос, до 200 символов) |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время последней реплики (сортировка списка веток) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | CASCADE |
| `user_id` | `users(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `ix_ai_chat_threads_owner` | `tender_id, user_id, updated_at DESC` | Ветки пользователя по тендеру, свежие первыми |

### ai_chat_messages

**Назначение.** Реплика пользователя или ответ ИИ в ветке разговора, включая предложенный моделью инструмент (кнопку) и черновик письма.

**Домен-владелец:** insights. **Бизнес-логика:** [insights.md](../02-business-logic/insights.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `user_id` | `bigint` | нет |  | Владелец разговора → `users.id` |
| `role` | `varchar(16)` | нет |  | Автор реплики: `user` / `bot` |
| `body` | `text` | нет | `''` | Текст |
| `bullets` | `jsonb` | нет | `'[]'` | Строки подтверждения ответа (JSON-массив) — как их отдала модель после сверки чисел |
| `subject` | `varchar(255)` | нет | `''` | Тема черновика письма |
| `draft` | `text` | нет | `''` | Текст черновика письма |
| `created_at` | `timestamptz` | нет | `now()` | Время |
| `thread_id` | `bigint` | да |  | Ветка разговора; NULL — реплики до появления веток → `ai_chat_threads.id` |
| `tool` | `varchar(32)` | нет | `''` | Инструмент, предложенный моделью (см. перечень); пусто — обычный ответ |
| `tool_args` | `jsonb` | нет | `'{}'` | Аргументы инструмента (JSON) |
| `notice` | `text` | нет | `''` | Что убрано из ответа и почему (например, выброшен черновик с именами чужих участников) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `tender_id` | `tenders(id)` | CASCADE |
| `thread_id` | `ai_chat_threads(id)` | CASCADE |
| `user_id` | `users(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `ix_ai_chat_messages_by_thread` | `thread_id, id` | Реплики ветки по порядку |
| `ix_ai_chat_messages_thread` | `tender_id, user_id, id` | Реплики тендера × пользователя |

**Перечисления и статусы.**

`role`: `user` — человек, `bot` — ИИ. `tool`: `focus` (показать позицию), `filter` (пресет таблицы), `pair` (разбор пары), `request_split` (черновик письма о составе цены), `message` (черновик письма участнику), `volume_correction` (к полю корректировки объёма), `export` (выгрузка), `explain_cell` (пояснить ячейку). Ни один инструмент не меняет данные без действия человека.

## 3.14. Домен comments — внутренние обсуждения

Внутренние обсуждения сотрудников (поставщик — только по своему КП) и отметки прочтения.

### comments

**Назначение.** Комментарий-обсуждение, привязанный полиморфно (`entity_type`, `entity_id`). Ответы — через `parent_comment_id`. Для ячейки сравнения тред определяется парой «позиция × контрагент».

**Домен-владелец:** comments. **Бизнес-логика:** [comments.md](../02-business-logic/comments.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `entity_type` | `varchar(20)` | нет |  | Что обсуждается: `tender` / `quote` / `tender_item` / `supplier` |
| `entity_id` | `bigint` | нет |  | Id обсуждаемой сущности |
| `supplier_id` | `bigint` | да |  | Якорь треда ячейки сравнения (для `tender_item`): контрагент; у прочих типов — NULL → `suppliers.id` |
| `parent_comment_id` | `bigint` | да |  | Комментарий, на который отвечают → `comments.id` |
| `author_id` | `bigint` | да |  | Автор → `users.id` |
| `comment_text` | `text` | нет |  | Текст |
| `created_at` | `timestamptz` | нет | `now()` | Время |
| `attachment_id` | `bigint` | да |  | Один файл к комментарию → `attachments.id` |
| `notify_user_id` | `bigint` | да |  | Адресат «+ Уведомить» (персональное уведомление и письмо) → `users.id` |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `attachment_id` | `attachments(id)` | NO ACTION |
| `author_id` | `users(id)` | NO ACTION |
| `notify_user_id` | `users(id)` | NO ACTION |
| `parent_comment_id` | `comments(id)` | NO ACTION |
| `supplier_id` | `suppliers(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_comments_author` | `author_id` WHERE `(author_id IS NOT NULL)` | Комментарии автора |
| `idx_comments_entity` | `entity_type, entity_id` | Комментарии сущности |
| `idx_comments_parent` | `parent_comment_id` WHERE `(parent_comment_id IS NOT NULL)` | Ответы |
| `idx_comments_supplier` | `supplier_id` WHERE `(supplier_id IS NOT NULL)` | Треды ячеек по контрагенту |

**Перечисления и статусы.**

`entity_type`: `tender` — тендер; `quote` — КП (единственный тип, доступный поставщику, и только по своему КП); `tender_item` — позиция/ячейка сравнения; `supplier` — карточка контрагента.

### comment_reads

**Назначение.** Персональная отметка прочтения комментария.

**Домен-владелец:** comments. **Бизнес-логика:** [comments.md](../02-business-logic/comments.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `user_id` (PK) | `bigint` | нет |  | Пользователь (часть PK) → `users.id` |
| `comment_id` (PK) | `bigint` | нет |  | Комментарий (часть PK) → `comments.id` |
| `read_at` | `timestamptz` | нет | `now()` | Когда прочитан |

**Первичный ключ:** (`user_id`, `comment_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `comment_id` | `comments(id)` | NO ACTION |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_comment_reads_comment` | `comment_id` | Отметки комментария (PK начинается с user_id) |

### position_comment_reads

**Назначение.** Отметка, что пользователь видел последнюю запись истории комментария поставщика к позиции (история — проекция `quote_items.supplier_comment` по версиям КП). Новая отличающаяся запись снова подсвечивается.

**Домен-владелец:** comments. **Бизнес-логика:** [comments.md](../02-business-logic/comments.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `user_id` (PK) | `bigint` | нет |  | Пользователь (часть PK) → `users.id` |
| `quote_item_id` (PK) | `bigint` | нет |  | Строка КП — последняя просмотренная запись истории (часть PK) → `quote_items.id` |
| `read_at` | `timestamptz` | нет | `now()` | Когда просмотрено |

**Первичный ключ:** (`user_id`, `quote_item_id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `quote_item_id` | `quote_items(id)` | CASCADE |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_position_comment_reads_quote_item` | `quote_item_id` | Отметки по строке КП |

## 3.15. Домен chat — чат с контрагентами

Переписка специалистов с контрагентом по тендеру: треды, сообщения с глобальным номером события (курсор живого потока), ревизии правок, персональные отметки прочтения.

### chat_threads

**Назначение.** Переписка специалистов Заказчика с контрагентом по тендеру — ровно один тред на пару «тендер × контрагент». Хранит счётчики реплик сторон для быстрого расчёта непрочитанного.

**Домен-владелец:** chat. **Бизнес-логика:** [chat.md](../02-business-logic/chat.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `tender_id` | `bigint` | нет |  | Тендер → `tenders.id` |
| `supplier_id` | `bigint` | нет |  | Контрагент → `suppliers.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время последней активности (сортировка списка чатов) |
| `specialist_last_read_seq` | `bigint` | нет | `0` | До какого события (`event_seq`) прочитала сторона специалистов — для пометки «прочитано» у отправителя (по стороне, имена сотрудников поставщику не раскрываются) |
| `supplier_last_read_seq` | `bigint` | нет | `0` | До какого события прочитала сторона поставщика |
| `msgs_from_supplier` | `bigint` | нет | `0` | Монотонный счётчик реплик поставщика (без системных; удаление не уменьшает) |
| `msgs_from_specialist` | `bigint` | нет | `0` | Монотонный счётчик реплик специалистов |
| `specialist_read_base` | `bigint` | нет | `0` | Откат для пользователя без своей записи «докуда прочитал»: сколько реплик поставщика сторона прочитала на момент перехода на персональные отметки |
| `supplier_read_base` | `bigint` | нет | `0` | То же для стороны поставщика |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `supplier_id` | `suppliers(id)` | NO ACTION |
| `tender_id` | `tenders(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_chat_threads_supplier_updated` | `supplier_id, updated_at DESC` | Чаты контрагента по свежести |
| `idx_chat_threads_tender_updated` | `tender_id, updated_at DESC` | Чаты тендера по свежести |
| `uq_chat_threads_tender_supplier` | `tender_id, supplier_id` | UNIQUE; Один тред на пару «тендер × контрагент» |

**Особенности.**

- Непрочитанное зрителя = счётчик реплик **другой** стороны − его `chat_thread_reads.read_count` (без перебора истории).

### chat_messages

**Назначение.** Реплика в треде: от специалиста, поставщика или системы; может нести вложение, ответ на реплику, ссылку на ячейку КП (мост «ячейка ↔ чат»). Каждое создание/правка получает номер события из глобальной последовательности — курсор живого потока.

**Домен-владелец:** chat. **Бизнес-логика:** [chat.md](../02-business-logic/chat.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Суррогатный ключ |
| `thread_id` | `bigint` | нет |  | Тред → `chat_threads.id` |
| `author_user_id` | `bigint` | да |  | Автор-пользователь; NULL — поставщик по ссылке без учётки или система → `users.id` |
| `author_role` | `varchar(20)` | нет |  | Сторона автора: `specialist` / `supplier` / `system` |
| `body` | `text` | нет | `''` *(прил.)* | Текст (полнотекстовый поиск с русской морфологией) |
| `attachment_id` | `bigint` | да |  | Вложение (одно на сообщение) → `attachments.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `event_seq` | `bigserial` | нет | `nextval('chat_event_seq')` | Номер события из глобальной последовательности `chat_event_seq` (по умолчанию `nextval`); уникален |
| `client_msg_id` | `uuid` | да |  | UUID отправки от клиента — ключ идемпотентности повтора (уникален в треде) |
| `reply_to_id` | `bigint` | да |  | Реплика, на которую отвечают (при удалении оригинала — NULL) → `chat_messages.id` |
| `quote_item_id` | `bigint` | да |  | Ячейка КП, из которой пришла реплика (мост «ячейка ↔ чат»); при удалении строки КП — NULL → `quote_items.id` |
| `comment_id` | `bigint` | да |  | Комментарий ячейки, продублированный в чат (не более одного дубля на комментарий) → `comments.id` |
| `kind` | `varchar(20)` | да |  | Вид: NULL — обычная реплика; `cell_quote` / `cell_thread` / `review` |
| `edited_at` | `timestamptz` | да |  | Когда отредактировано автором |
| `deleted_at` | `timestamptz` | да |  | Когда удалено автором (мягкое удаление; текст и вложение уходят в ревизию) |
| `edited_seq` | `bigint` | да |  | Номер события последней правки/удаления (из той же последовательности) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `attachment_id` | `attachments(id)` | NO ACTION |
| `author_user_id` | `users(id)` | NO ACTION |
| `thread_id` | `chat_threads(id)` | NO ACTION |
| `comment_id` | `comments(id)` | SET NULL |
| `quote_item_id` | `quote_items(id)` | SET NULL |
| `reply_to_id` | `chat_messages(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_chat_messages_attachment` | `attachment_id` WHERE `(attachment_id IS NOT NULL)` | FK на вложение |
| `idx_chat_messages_author_user` | `author_user_id` WHERE `(author_user_id IS NOT NULL)` | FK на автора |
| `idx_chat_messages_edited_seq` | `edited_seq` WHERE `(edited_seq IS NOT NULL)` | Дельта правок после курсора |
| `idx_chat_messages_body_fts` | `to_tsvector('russian', body)` | GIN; Полнотекстовый поиск `to_tsvector('russian', body)` (GIN) |
| `idx_chat_messages_reply_to` | `reply_to_id` WHERE `(reply_to_id IS NOT NULL)` | Ответы на реплику |
| `idx_chat_messages_thread_created` | `thread_id, created_at, id` | Лента треда |
| `idx_chat_messages_thread_seq` | `thread_id, event_seq` | Дельта треда после курсора |
| `uq_chat_messages_bridge_comment` | `comment_id` WHERE `(comment_id IS NOT NULL)` | UNIQUE; Один дубль на комментарий ячейки |
| `uq_chat_messages_client_msg` | `thread_id, client_msg_id` WHERE `(client_msg_id IS NOT NULL)` | UNIQUE; Идемпотентная отправка |
| `uq_chat_messages_event_seq` | `event_seq` | UNIQUE; Уникальность номера события |
| `idx_chat_messages_quote_item` | `quote_item_id` WHERE `(quote_item_id IS NOT NULL)` | FK на строку КП |

**Перечисления и статусы.**

`author_role`: `specialist` — сотрудник Заказчика; `supplier` — контрагент; `system` — системное (не учитывается в счётчиках). `kind` (мост «ячейка ↔ чат», дубль отправляет сотрудник из панели ячейки «подрядчик × позиция»): `cell_quote` — цитата ячейки КП (комментарий поставщика к позиции); `cell_thread` — дубль комментария из обсуждения ячейки; `review` — «Решение по позиции». По `quote_item_id`/`comment_id` строится бейдж позиции и переход к ячейке.

**Особенности.**

- Последовательность `chat_event_seq` (bigint) создаётся SQL-миграцией и принадлежит колонке `event_seq` (`OWNED BY`).
- Мягкое удаление (`deleted_at`): строка остаётся местом в ленте; прежний текст — в `chat_message_revisions`.

### chat_message_revisions

**Назначение.** Правка или удаление реплики не стирает сказанное: прежний текст и вложение сохраняются здесь и попадают в выгрузку переписки.

**Домен-владелец:** chat. **Бизнес-логика:** [chat.md](../02-business-logic/chat.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `message_id` | `bigint` | нет |  | Сообщение (каскадное удаление) → `chat_messages.id` |
| `action` | `varchar(10)` | нет |  | Что произошло: `edit` / `delete` |
| `body` | `text` | нет |  | Текст до изменения |
| `attachment_id` | `bigint` | да |  | Вложение до изменения (при удалении файла — NULL) → `attachments.id` |
| `user_id` | `bigint` | да |  | Кто правил; NULL — поставщик по ссылке → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `attachment_id` | `attachments(id)` | SET NULL |
| `message_id` | `chat_messages(id)` | CASCADE |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_chat_message_revisions_message` | `message_id` | Ревизии сообщения |

**Перечисления и статусы.**

`action`: `edit` — правка, `delete` — удаление.

### chat_thread_reads

**Назначение.** «Докуда прочитал» — на человека (а не на сторону). Субъект — пользователь либо (для поставщика по ссылке без учётки) приглашение.

**Домен-владелец:** chat. **Бизнес-логика:** [chat.md](../02-business-logic/chat.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `thread_id` | `bigint` | нет |  | Тред (каскадное удаление) → `chat_threads.id` |
| `user_id` | `bigint` | да |  | Пользователь (взаимоисключимо с `invitation_id`) → `users.id` |
| `invitation_id` | `bigint` | да |  | Приглашение — субъект для поставщика без учётной записи → `tender_invitations.id` |
| `last_read_seq` | `bigint` | нет | `0` | Номер последнего прочитанного события |
| `read_count` | `bigint` | нет | `0` | Сколько реплик другой стороны было в треде до `last_read_seq` |
| `updated_at` | `timestamptz` | нет | `now()` | Время обновления |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `invitation_id` | `tender_invitations(id)` | NO ACTION |
| `thread_id` | `chat_threads(id)` | CASCADE |
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `uq_chat_thread_reads_invitation` | `thread_id, invitation_id` WHERE `(invitation_id IS NOT NULL)` | UNIQUE; Одна запись на тред × приглашение |
| `uq_chat_thread_reads_user` | `thread_id, user_id` WHERE `(user_id IS NOT NULL)` | UNIQUE; Одна запись на тред × пользователя |

**CHECK-ограничения:**

- `ck_chat_thread_reads_subject`: `((user_id IS NOT NULL) <> (invitation_id IS NOT NULL))` — Ровно один субъект: пользователь или приглашение

## 3.16. Домен security — инциденты, SIEM, техлоги

Ведение инцидентов (лента, пакеты РКН), выгрузка журнала в SIEM, правила-паттерны техлогов. Состояние форвардера техлогов `log_forward_state` описано в группе platform.

### security_incident_timeline

**Назначение.** События расследования инцидента: смены статуса, назначения, комментарии оператора, формирование пакета РКН. Отдельная таблица, чтобы параллельные правки не затирали друг друга.

**Домен-владелец:** security. **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `incident_id` | `bigint` | нет |  | Инцидент (каскадное удаление) → `security_incidents.id` |
| `kind` | `varchar(32)` | нет |  | Вид записи: `status_changed` / `assigned` / `comment` / `rkn_package` |
| `author_id` | `bigint` | да |  | Автор (при удалении — NULL) → `users.id` |
| `text` | `text` | да |  | Текст комментария |
| `payload` | `jsonb` | да |  | Данные события (JSON: from/to статуса, назначенный и т.п.) |
| `created_at` | `timestamptz` | нет | `now()` | Время |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `author_id` | `users(id)` | SET NULL |
| `incident_id` | `security_incidents(id)` | CASCADE |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_incident_timeline_incident` | `incident_id, created_at` | Лента инцидента по времени |

**Перечисления и статусы.**

`kind`: `status_changed` — смена статуса; `assigned` — назначение ответственного; `comment` — комментарий; `rkn_package` — сформирован пакет для РКН.

### rkn_packages

**Назначение.** Пакет сведений для уведомления Роскомнадзора по 152-ФЗ: версионируемый снимок, привязанный к инциденту. Подача — вручную, система только готовит комплект.

**Домен-владелец:** security. **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `incident_id` | `integer` | нет |  | Инцидент (каскадное удаление) → `security_incidents.id` |
| `version` | `integer` | нет |  | Номер версии пакета в инциденте |
| `payload` | `jsonb` | нет |  | Содержимое пакета (JSON) |
| `generated_by` | `integer` | да |  | Кто сформировал (при удалении — NULL) → `users.id` |
| `generated_at` | `timestamptz` | нет | `now()` | Время формирования |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `generated_by` | `users(id)` | SET NULL |
| `incident_id` | `security_incidents(id)` | CASCADE |

**Уникальные ограничения:** `rkn_packages_incident_id_version_key` (incident_id, version).

### siem_settings

**Назначение.** Однострочная настройка передачи `security_audit_log` в SIEM по HTTP пачками и курсор выгрузки. Недоступный коллектор не теряет события: курсор двигается только после успешной отправки.

**Домен-владелец:** security. **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `integer` | нет | `1` | Всегда 1 |
| `enabled` | `boolean` | нет | `false` | Выгрузка включена |
| `url` | `text` | да |  | URL коллектора |
| `auth_header` | `text` | да |  | Значение заголовка авторизации коллектора (Bearer/API-ключ) — **шифруется**; наружу отдаётся только признак «задан» |
| `batch_size` | `integer` | нет | `200` | Размер пачки (по умолчанию 200) |
| `last_sent_event_id` | `bigint` | нет | `0` | Курсор: id последнего отправленного события журнала |
| `last_sent_at` | `timestamptz` | да |  | Время последней успешной отправки |
| `last_error` | `text` | да |  | Текст последней ошибки |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**CHECK-ограничения:**

- `siem_settings_singleton`: `(id = 1)` — Одна строка

**Особенности.**

- Шифруемое поле: `auth_header`.
- Отправка — периодическая задача `siem-drain` (интервал `SIEM_DRAIN_INTERVAL`, 15 с).

### log_pattern_rules

**Назначение.** Правило «подстрока/регулярное выражение в строке технического лога уровня ≥ N, M совпадений за K минут → алерт в Telegram». Редактируется без кода; правка журналируется.

**Домен-владелец:** security. **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ |
| `name` | `varchar(200)` | нет |  | Название (до 200 символов) |
| `level` | `varchar(20)` | нет | `'ERROR'` | Минимальный уровень записи: `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL` (`WARNING` = «WARNING и выше») |
| `match` | `text` | нет |  | Подстрока или регулярное выражение |
| `is_regex` | `boolean` | нет | `false` | `match` — регулярное выражение |
| `window_minutes` | `integer` | нет | `5` | Окно, минуты (по умолчанию 5) |
| `threshold` | `integer` | нет | `1` | Порог совпадений в окне (1 — каждое) |
| `dedupe_key` | `varchar(200)` | да |  | Ключ антиспама алертов; пусто — `pattern:<id>` |
| `enabled` | `boolean` | нет | `true` | Включено |
| `created_by` | `bigint` | да |  | Автор правила (при удалении пользователя — NULL) → `users.id` |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | SET NULL |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_log_pattern_rules_enabled` | `enabled` | Выборка включённых правил |

**Особенности.**

- Применяется периодической задачей `log-patterns` в каждом процессе; канал оповещения — всегда Telegram.

## 3.17. Домен settings — системные настройки

Домен settings своих ORM-моделей не объявляет: однострочные настройки объявлены в suppliers (`app_settings`) и platform (`smtp_settings`, `mail_templates`), а читаются и правятся сервисами settings (экран «Настройки системы»).

### app_settings

**Назначение.** Единственная строка (`id = 1`) общесистемных настроек: обязательность опыта работ в анкете контрагента и пороги аномалий сравнения (общие для всех тендеров). Правится экраном «Настройки системы».

**Домен-владелец:** settings (модель — suppliers). **Бизнес-логика:** [settings.md](../02-business-logic/settings.md). **Комментарий в схеме:** «Общесистемные настройки приложения (эпик 39, П14)».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | `1` *(прил.)* | Всегда 1 |
| `require_questionnaire_work_experience` | `boolean` | нет | `false` | Опыт работ обязателен в анкете контрагента (по умолчанию нет) |
| `anomaly_spread_notice` | `numeric(6,2)` | нет | `15` | Порог разброса цен «заметно», % (по умолчанию 15) |
| `anomaly_spread_high` | `numeric(6,2)` | нет | `40` | Порог разброса «сильно», % (по умолчанию 40) |
| `anomaly_k` | `numeric(6,2)` | нет | `3` | Коэффициент выброса k (по умолчанию 3) |
| `anomaly_key_share` | `numeric(6,2)` | нет | `80` | Доля ключевых работ, % суммы (по умолчанию 80) |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Особенности.**

- Изменение порогов журналируется (`anomaly_thresholds_changed`) и меняет разметку сравнения во всех тендерах сразу.

### smtp_settings

**Назначение.** Строка `id = 1`: SMTP-сервер, заданный из интерфейса (используется только при `SMTP_SETTINGS_IN_UI=true`, иначе — из окружения), и выключатель «Почта включена». `MAIL_ENABLED=false` в окружении главнее выключателя.

**Домен-владелец:** settings (модель — platform). **Бизнес-логика:** [settings.md](../02-business-logic/settings.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `serial` | нет | автоинкремент | Всегда 1 (ограничения нет — правило приложения) |
| `host` | `varchar(255)` | да |  | SMTP-хост |
| `port` | `integer` | нет | `587` *(прил.)* | Порт (по умолчанию 587) |
| `username` | `varchar(255)` | да |  | Логин SMTP |
| `password` | `text` | да |  | Пароль SMTP — **шифруется**, наружу не отдаётся |
| `use_tls` | `boolean` | нет | `true` *(прил.)* | Использовать TLS (по умолчанию да) |
| `from_email` | `varchar(255)` | да |  | Адрес отправителя |
| `notification_email` | `varchar(255)` | да |  | **Устарело**: бывшая общая почта уведомлений, не читается |
| `mail_paused` | `boolean` | нет | `false` | Почта на паузе (выключатель в интерфейсе); строки нет — не на паузе |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Особенности.**

- Шифруемое поле: `password`.

### mail_templates

**Назначение.** Свой текст письма вместо встроенного. Нет строки — письмо уходит со стандартным текстом из кода.

**Домен-владелец:** settings (модель — platform). **Бизнес-логика:** [settings.md](../02-business-logic/settings.md). **Комментарий в схеме:** «Свой текст письма вместо стандартного из кода (настройки → шаблоны писем)».

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `code` (PK) | `varchar(40)` | нет |  | Код шаблона (PK): `invitation` / `round` / `fkp_notice` |
| `subject` | `text` | нет |  | Тема с переменными `{…}` |
| `body` | `text` | нет |  | Тело с переменными `{номер}`, `{название}`, `{объект}`, `{вид_работ}`, `{срок}`, `{ндс}`, `{приглашающий}`, `{ссылка}`, `{бренд}` и т.п. (`{ссылка}` обязательна) |
| `updated_by` | `bigint` | да |  | Кто изменил → `users.id` |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`code`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `updated_by` | `users(id)` | NO ACTION |

**Перечисления и статусы.**

`code`: `invitation` — приглашение к тендеру; `round` — приглашение в раунд (переторжка); `fkp_notice` — уведомление об изменении формы КП.

## 3.18. Домен bug_reports — баг-репорты

Баг-репорты из виджета интерфейса, отправляемые во внешний трекер задач.

### bug_reports

**Назначение.** Сообщение об ошибке из виджета в интерфейсе: связка «пользователь → задача во внешнем трекере», а не собственная тикет-система. Отправка — фоновое задание `bug_report_dispatch`.

**Домен-владелец:** bug_reports. **Бизнес-логика:** [bug-reports.md](../02-business-logic/bug-reports.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `bigserial` | нет | автоинкремент | Суррогатный ключ (bigserial) |
| `user_id` | `bigint` | нет |  | Автор → `users.id` |
| `title` | `varchar(200)` | нет |  | Заголовок (до 200 символов) |
| `status` | `varchar(20)` | нет | `'pending'` | Внутренний статус отправки: `pending` / `sent` / `failed` |
| `user_status` | `varchar(20)` | нет | `'received'` | Статус для пользователя («Мои репорты»): `received` / `investigating` / `fixed` |
| `external_provider` | `varchar(32)` | да |  | Провайдер трекера (сейчас `github`) |
| `external_issue_id` | `varchar(64)` | да |  | Id задачи в трекере |
| `external_issue_key` | `varchar(64)` | да |  | Ключ/номер задачи в трекере |
| `external_issue_url` | `text` | да |  | Ссылка на задачу |
| `payload` | `jsonb` | нет | `'{}'` | Данные репорта (JSON): описание, шаги, контекст страницы/браузера |
| `screenshot_path` | `text` | да |  | Ключ скриншота в файловом хранилище |
| `fail_reason` | `text` | да |  | Причина неудачной отправки |
| `created_at` | `timestamptz` | нет | `now()` | Время создания |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `user_id` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `ix_bug_reports_external_issue_id` | `external_issue_id` WHERE `(external_issue_id IS NOT NULL)` | Поиск по задаче трекера (обновление статуса по событиям трекера) |
| `ix_bug_reports_user_id_created` | `user_id, created_at DESC` | Репорты пользователя |

**Перечисления и статусы.**

`status`: `pending` — ждёт отправки (по умолчанию), `sent` — отправлен, `failed` — не отправлен. `user_status`: `received` — получен (по умолчанию), `investigating` — разбираемся, `fixed` — исправлено.

## 3.19. Домен platform — технические таблицы платформы

Технические таблицы без бизнес-смысла: очередь фоновых заданий, кэш тяжёлых чтений, счётчики лимитов частоты. Сюда же по смыслу отнесены технические таблицы других доменов — `pending_uploads` (домен files) и `log_forward_state` (домен security). Подробно — [архитектура](../07-architecture/architecture.md).

### background_jobs

**Назначение.** Очередь фоновых заданий в Postgres: воркер внутри процесса API берёт задания `FOR UPDATE SKIP LOCKED`. Подробно — [архитектура, §3](../07-architecture/architecture.md#3-фоновые-задания).

**Домен-владелец:** platform.

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `uuid` | нет | UUID v4 *(прил.)* | UUID задания (отдаётся клиенту для опроса) |
| `job_type` | `varchar(64)` | нет |  | Тип задания (см. перечень) |
| `status` | `varchar(20)` | нет |  | Статус: `queued` / `running` / `done` / `failed` / `cancelled` |
| `created_by` | `bigint` | да |  | Кто поставил → `users.id` |
| `request_id` | `varchar(64)` | да |  | Сквозной id запроса, породившего задание |
| `input_meta` | `jsonb` | нет | `{}` *(прил.)* | Постановка задачи (JSON) |
| `input_path` | `text` | да |  | Ключ входного файла в хранилище (`jobs/<id>_<имя>`) |
| `result` | `jsonb` | да |  | Результат (JSON) |
| `progress` | `jsonb` | да |  | Ход выполнения (JSON) — пишет задание, экран опрашивает |
| `error_message` | `text` | да |  | Причина отказа для человека |
| `error_class` | `varchar(128)` | да |  | Класс ошибки |
| `attempts` | `integer` | нет | `0` *(прил.)* | Число попыток |
| `max_attempts` | `integer` | нет | `3` *(прил.)* | Потолок попыток (по умолчанию 3) |
| `created_at` | `timestamptz` | нет | `now()` | Время постановки |
| `started_at` | `timestamptz` | да |  | Начало выполнения |
| `finished_at` | `timestamptz` | да |  | Окончание |
| `heartbeat_at` | `timestamptz` | да |  | Последний признак жизни (обновляется записью хода) |
| `next_attempt_at` | `timestamptz` | да |  | Не брать раньше этого момента (пауза перед повтором 30·2^(n−1) с) |

**Первичный ключ:** (`id`).

**Внешние ключи:**

| Колонки | Ссылается на | ON DELETE |
|---|---|---|
| `created_by` | `users(id)` | NO ACTION |

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_background_jobs_queued` | `next_attempt_at, created_at` WHERE `((status)::text = 'queued'::text)` | Поллинг очереди: `status='queued'` по `next_attempt_at, created_at` |

**Перечисления и статусы.**

`status`: `queued` — в очереди; `running` — выполняется; `done` — готово; `failed` — ошибка; `cancelled` — отменено.

`job_type`: `parse_excel` — разбор Excel-ведомости; `ai_analysis` — ИИ-разбор тендера/пары; `security_incident_email` — письмо об инциденте; `supplier_notification` — письмо о новом контрагенте; `notification_mail` — письмо по каталогу уведомлений; `chat_mail` — письмо о сообщении чата; `bug_report_dispatch` — отправка баг-репорта.

**Особенности.**

- Очистки завершённых заданий нет — см. открытые вопросы.

### heavy_read_cache

**Назначение.** Готовые тела тяжёлых ответов (сравнительная таблица, сводка бюджета, цены реестра) — общие для всех процессов. Хранятся байтами, уже сжатыми gzip; отдаются без повторной сериализации. Ключ включает ревизию/отпечаток данных — явной инвалидации нет, старые записи уходят по TTL.

**Домен-владелец:** platform.

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `key` (PK) | `text` | нет |  | Ключ: `<вид>:<хеш частей ключа>` (например, `registry-prices:…`) |
| `payload` | `bytea` | нет |  | Тело ответа (gzip-байты, `bytea`) |
| `expires_at` | `timestamptz` | нет |  | Срок жизни (`HEAVY_READ_CACHE_SECONDS`, 60 с) |

**Первичный ключ:** (`key`).

**Внешние ключи:** нет.

**Особенности.**

- **UNLOGGED** (задаётся SQL-миграцией): не пишется в WAL, после сбоя сервера очищается — допустимо для кэша.
- Очистка: периодическая задача `shared-store-purge` (раз в час) удаляет просроченные строки.
- Перед таблицей — кэш в памяти процесса на тот же срок.

### rate_limit_counters

**Назначение.** Счётчики фиксированного окна для лимитов частоты (вход, обновление сессии, MFA, формы по ссылке, лендинг, тяжёлые чтения). Общие для всех процессов: лимит в памяти процесса умножался бы на их число.

**Домен-владелец:** platform.

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `key` (PK) | `text` | нет |  | Ключ `<корзина>:<субъект>[:<ip>]`, например `login:<ip>`, `heavy-read:user:<id>`, `chat_invite_read:<токен>:<ip>` |
| `hits` | `integer` | нет |  | Число попаданий в текущем окне |
| `expires_at` | `timestamptz` | нет |  | Конец окна |

**Первичный ключ:** (`key`).

**Внешние ключи:** нет.

**Особенности.**

- **UNLOGGED**: потеря счётчиков при сбое лишь обнуляет окно.
- Одно атомарное `INSERT … ON CONFLICT DO UPDATE` на попадание. Для auth-эндпоинтов при недоступности счётчиков — отказ (fail-closed).
- Очистка: `shared-store-purge` (раз в час).

### pending_uploads

**Назначение.** Выданная presigned-ссылка на прямую загрузку в бакет, ещё не подтверждённая. Строка `attachments` создаётся только при подтверждении (объект в бакете проверен: размер, тип, антивирус), эта запись тогда удаляется.

**Домен-владелец:** files (технич.). **Бизнес-логика:** [files.md](../02-business-logic/files.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `uuid` | нет | UUID v4 *(прил.)* | UUID загрузки |
| `entity_type` | `varchar(20)` | нет |  | Тип будущего владельца (как у `attachments`) |
| `entity_id` | `bigint` | нет |  | Id будущего владельца |
| `doc_type` | `varchar(100)` | да |  | Слот документа |
| `file_name` | `varchar(500)` | нет |  | Имя файла |
| `file_path` | `varchar(1000)` | нет |  | Ключ объекта (префикс `pending-upload/`) |
| `storage_backend` | `varchar(10)` | нет |  | Хранилище (`s3`) |
| `content_type` | `varchar(100)` | нет |  | Заявленный MIME-тип |
| `file_size` | `bigint` | нет |  | Заявленный размер, байт |
| `uploaded_by` | `bigint` | да |  | Кто загружает (логическая ссылка, без FK) |
| `created_at` | `timestamptz` | нет | `now()` | Время выдачи ссылки |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**Индексы:**

| Индекс | Колонки / выражение | Назначение |
|---|---|---|
| `idx_pending_uploads_created_at` | `created_at` | Поиск просроченных для очистки |

**Особенности.**

- Очистка: периодическая задача `pending-uploads-cleanup` (раз в час) удаляет записи старше `UPLOAD_PENDING_TTL_HOURS` (24 ч) вместе с объектом в бакете.

### log_forward_state

**Назначение.** Только состояние отправки технических логов (сам приёмник — URL, токен, включатель — задаётся переменными окружения). Курсор по образцу SIEM: пачка уходит одним POST, курсор двигается после успеха.

**Домен-владелец:** security (технич.). **Бизнес-логика:** [security.md](../02-business-logic/security.md).

| Колонка | Тип | NULL | По умолчанию | Описание |
|---|---|---|---|---|
| `id` (PK) | `integer` | нет | `1` | Всегда 1 |
| `last_sent_seq` | `bigint` | нет | `0` | Курсор: номер последней отправленной строки лога (нумерация в процессе продолжается с него после рестарта) |
| `last_sent_at` | `timestamptz` | да |  | Время последней успешной отправки |
| `last_error` | `text` | да |  | Последняя ошибка |
| `dropped_count` | `bigint` | нет | `0` | Сколько строк выброшено переполненным буфером |
| `updated_at` | `timestamptz` | нет | `now()` | Время изменения |

**Первичный ключ:** (`id`).

**Внешние ключи:** нет.

**CHECK-ограничения:**

- `log_forward_state_singleton`: `(id = 1)` — Одна строка

**Особенности.**

- Отправка — периодическая задача `log-forward` (интервал `LOG_FORWARD_INTERVAL`, 10 с).

## 4. Служебные объекты БД вне перечня 82 таблиц

| Объект | Тип | Назначение |
|---|---|---|
| `atlas_schema_revisions` | Таблица инструмента миграций | Журнал применённых версий миграций (версия, описание, число применённых операций, время и длительность, ошибка, хеш). Ведётся инструментом миграций, приложением не читается; в бизнес-перечень не входит. При выборе другого инструмента миграций — его аналог |
| `chat_event_seq` | Последовательность (`bigint`) | Глобальный монотонный номер событий чата: `chat_messages.event_seq` (по умолчанию `nextval`) и `edited_seq`. `OWNED BY chat_messages.event_seq`. Создаётся ручной SQL-миграцией под фиксированным именем (приложение обращается к ней по имени) |
| `trg_prevent_audit_log_modification()` | Функция plpgsql | Исключение при UPDATE/DELETE в `security_audit_log` |
| `trg_prevent_qualification_history_modification()` | Функция plpgsql | Исключение при UPDATE/DELETE в `supplier_qualification_history` |
| `prevent_update_security_audit_log`, `prevent_delete_security_audit_log` | Триггеры BEFORE … FOR EACH ROW | Append-only журнала ИБ |
| `prevent_update_supplier_qualification_history`, `prevent_delete_supplier_qualification_history` | Триггеры BEFORE … FOR EACH ROW | Append-only истории квалификации |
| Канал `LISTEN/NOTIFY` | Механизм СУБД | Шина живых событий между процессами (таблиц нет) — [архитектура, §4](../07-architecture/architecture.md#4-живые-события-realtime) |

Представлений (VIEW), материализованных представлений, типов ENUM, партиционирования и расширений в схеме нет. Последовательности автоинкремента `serial`/`bigserial` — стандартные (`<таблица>_id_seq`).

Комментарии к объектам БД (`COMMENT ON`) заданы у таблиц `app_settings`, `invite_quote_drafts`, `mail_templates`, `supplier_work_experience`, `tender_creation_drafts`, `tender_protocols` и колонок `suppliers.category`, `suppliers.finance_status`, `suppliers.finance_limit_amount`, `suppliers.founded_at`, `tenders.winner_accreditation_status` — их смысл перенесён в описания выше.


## 5. Объёмы данных на демо-стенде

Снимок живой БД демо-стенда (октябрь 2026; данные обезличены). Общий размер БД — **≈32 МБ**. Используется для оценки нагрузки: основной объём — строки КП и позиции тендеров (растут как «позиций × участников × версий»), журнал ИБ и снимки ИИ-разборов.

| Таблица | Строк | Размер (с индексами) | Комментарий для оценки роста |
|---|---:|---:|---|
| `quote_items` | 8 619 | 1,3 МБ | ≈ позиций тендера × участников × версий КП; самая быстрорастущая бизнес-таблица |
| `tender_items` | 2 239 | 1,1 МБ | Позиции ФКП: от десятков до тысяч на тендер (в среднем ≈200 на демо-тендер) |
| `tender_sections` | 554 | 216 КБ | Разделы ФКП |
| `security_audit_log` | 9 592 | 4,2 МБ | Растёт с каждым входом/действием (без очистки); ≈94 % — выход по неактивности |
| `heavy_read_cache` | 19 | 8,1 МБ | Готовые тела ответов (сравнение до мегабайт на тендер); живут 60 с |
| `ai_analysis_results` | 7 | 2,7 МБ | ≈0,4 МБ на разбор (снимок таблицы целиком) |
| `tender_fkp_versions` | 11 | 192 КБ | Слепок структуры на каждую версию ФКП |
| `quote_condition_values` | 123 | 96 КБ | Условия × участники × версии |
| `tender_condition_fields` | 126 | 80 КБ | ≈11 условий на тендер |
| `activity_log` | 108 | 176 КБ | События тендеров |
| `app_notifications` | 98 | 160 КБ | По записи на получателя события |
| `chat_messages` | 71 | 272 КБ | Сообщения (с FTS-индексом) |
| `auth_sessions` | 60 | 152 КБ | Сессии устройств (чистятся через 7 дней после истечения) |
| `supplier_checks` | 50 | 80 КБ | 5 проверок на контрагента за запуск |
| `tender_round_recipients` | 46 | 80 КБ | Адресаты раундов |
| `attachments` | 41 | 96 КБ | Метаданные; файлы — в хранилище |
| `supplier_status_events` | 70 | 48 КБ | ≈7 событий на аккредитацию |
| `supplier_qualification_history` | 40 | 48 КБ | ≈4 решения на аккредитацию |
| `tender_invitations` | 29 | 72 КБ | ≈3 участника на тендер |
| `quote_versions` / `quotes` | 17 / 12 | 88 / 96 КБ | |
| `tenders` | 11 | 160 КБ | |
| `suppliers` | 10 | 192 КБ | Контрагенты (+30 контактов, 30 строк выручки, 26 записей опыта) |
| `users` | 17 | 48 КБ | |

Остальные таблицы на стенде содержат единицы строк или пусты (`bug_reports`, `security_incidents`, `security_incident_rules`, `rkn_packages`, `topup_requests`, `pending_uploads`, `invite_quote_drafts`, `tender_protocols`, `mail_templates`, `chat_message_revisions`, `position_comment_reads`, `login_challenges`, `password_reset_tokens`, `unit_aliases`).

**Ориентиры для промышленной эксплуатации** (выведены из структуры, не замерены): тендер на 1 000 позиций с 10 участниками и 3 версиями КП даёт ≈30 000 строк `quote_items`; при 200 тендерах в год — ≈6 млн строк в год, что требует индексов по `quote_version_id`/`tender_item_id` (есть) и кэша тяжёлых чтений сравнения (есть). Журнал ИБ на стенде за два месяца — ≈9,7 тыс. строк, из них ≈0,6 тыс. содержательных (остальное — выходы по неактивности); рост пропорционален числу пользователей и их действий. Целевые объёмы заказчика — см. открытые вопросы.


## 6. Сверка полноты

- Разделов уровня `###` (по одному на таблицу) в этом файле: **82**.
- Таблиц `table "…"` в декларативной схеме: **82**; ORM-моделей с `__tablename__`: **82**; таблиц в живой БД стенда (схема `public`, без `atlas_schema_revisions`): **82**.
- Наборы имён схемы и ORM совпадают; наборы колонок каждой таблицы в схеме, ORM и живой БД совпадают.
- Расхождения ORM-моделей со схемой (истиной считается схема БД, при воссоздании ориентироваться на неё):
  - у FK `ai_chat_messages.tender_id/user_id/thread_id`, `ai_chat_threads.tender_id/user_id`, `ai_insight_cache.tender_id`, `position_comment_reads.quote_item_id` в схеме задано `ON DELETE CASCADE`, в ORM-моделях поведение не объявлено;
  - `log_pattern_rules.created_by` в схеме имеет FK на `users(id)` с `ON DELETE SET NULL`, в ORM-модели FK не объявлен;
  - комментарий в ORM-модели `quote_items` упоминает вычисляемую колонку `total_price` — в схеме и живой БД её нет, итог строки вычисляется приложением.
- Таблиц, которые есть в ORM, но отсутствуют в схеме (или наоборот), нет.

