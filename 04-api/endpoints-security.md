# API: информационная безопасность (security)

Ручки раздела «Журнал безопасности»: журнал событий ИБ с фильтрами и тремя форматами выгрузки, каталог событий, передача журнала во внешний SIEM, статус и самопроверка полевого шифрования, инциденты (реестр, карточка, контекст срабатывания, таймлайн, смена статуса/критичности/ответственного), правила автоинцидентов, правила-паттерны технических логов с оповещением в технический канал и версионируемые пакеты сведений для уведомления Роскомнадзора по 152-ФЗ. Общие соглашения (аутентификация, ошибки, конверт `{items, total, limit, offset}`) — в [overview.md](overview.md).

Связанные разделы: бизнес-логика — [../02-business-logic/security.md](../02-business-logic/security.md) (полный каталог событий журнала, правила, инциденты); экран — [../03-screens/security-log.md](../03-screens/security-log.md).

## Сводка

| Метод | Путь | Кратко | Доступ |
|---|---|---|---|
| GET | `/security/events` | Журнал событий ИБ с фильтрами | право `security.manage` |
| GET | `/security/events/catalog` | Каталог событий: коды, подписи, категории, критичность | `security.manage` |
| GET | `/security/events/export` | Выгрузка среза журнала в CSV | `security.manage` |
| GET | `/security/events/export.jsonl` | Выгрузка среза в NDJSON (формат SIEM) | `security.manage` |
| GET | `/security/events/export.txt` | Выгрузка среза текстом (строка на событие) | `security.manage` |
| GET | `/security/siem` | Настройки передачи журнала в SIEM | `security.manage` |
| PUT | `/security/siem` | Изменить настройки SIEM | `security.manage` |
| POST | `/security/siem/drain` | Отправить следующую пачку сейчас («Проверить соединение») | `security.manage` |
| GET | `/security/crypto/status` | Статус провайдера шифрования и реестр шифруемых полей | `security.manage` |
| POST | `/security/crypto/self-test` | Самопроверка шифрования с записью в журнал | `security.manage` |
| GET | `/security/incidents` | Реестр инцидентов (очередь) | `security.manage` |
| GET | `/security/incidents/{incident_id}` | Карточка инцидента | `security.manage` |
| PATCH | `/security/incidents/{incident_id}` | Статус, критичность, ответственный, затронутые ресурсы | `security.manage` |
| GET | `/security/incidents/{incident_id}/context` | Правило и события, поднявшие инцидент | `security.manage` |
| GET | `/security/incidents/{incident_id}/timeline` | Таймлайн расследования | `security.manage` |
| POST | `/security/incidents/{incident_id}/timeline` | Комментарий в таймлайн | `security.manage` |
| GET | `/security/incident-rules` | Правила автоинцидентов | `security.manage` |
| POST | `/security/incident-rules` | Создать правило | `security.manage` |
| PATCH | `/security/incident-rules/{rule_id}` | Изменить правило | `security.manage` |
| DELETE | `/security/incident-rules/{rule_id}` | Удалить правило | `security.manage` |
| POST | `/security/incident-rules/{rule_id}/test` | Прогон правила по последним событиям (без создания инцидента) | `security.manage` |
| GET | `/security/log-patterns` | Паттерны технических логов | `security.manage` |
| POST | `/security/log-patterns` | Создать паттерн | `security.manage` |
| PATCH | `/security/log-patterns/{rule_id}` | Изменить паттерн | `security.manage` |
| DELETE | `/security/log-patterns/{rule_id}` | Удалить паттерн | `security.manage` |
| POST | `/security/log-patterns/{rule_id}/test` | Прогон паттерна по присланным строкам | `security.manage` |
| POST | `/security/incidents/{incident_id}/rkn-package` | Зафиксировать новую версию пакета для РКН | `security.manage` |
| GET | `/security/incidents/{incident_id}/rkn-packages` | Версии пакетов по инциденту | `security.manage` |
| GET | `/security/rkn-packages/{package_id}` | Скачать пакет (JSON-файл) | `security.manage` |
| GET | `/security/rkn-draft` | Устаревший алиас: последняя версия пакета по инциденту | `security.manage` |

**Доступ — общее для раздела.** Все ручки закрыты правом `security.manage` (по умолчанию — роль «служба безопасности» и администратор). Отказ — `403` «Недостаточно прав для этого действия — обратитесь к администратору системы» + событие `permission_denied` в журнале; нет сессии — `401` «Не авторизован».

**Время.** Все отметки времени в ответах этого раздела (кроме JSONL/SIEM и пакетов РКН) — строки по Москве в формате `ДД.ММ.ГГГГ ЧЧ:ММ:СС`.

Сложность блока: **L** (журнал с выгрузками — M; инциденты и правила — M; SIEM и паттерны логов — M; РКН-пакеты — S).

### Справочники раздела

| Справочник | Значения |
|---|---|
| Критичность (`severity`) | `low` низкая, `medium` средняя, `high` высокая, `critical` критическая |
| Статус инцидента | `open` → `investigating` → `closed` (переход в любой статус разрешён, в т. ч. повторное открытие) |
| Исход события (`outcome`) | `success` \| `failure` — производная от кода события: `failure` для `login_failed`, `account_locked`, `permission_denied`, `session_idle_timeout`, `session_reuse_detected`, `incident_created`, `chat_rate_limited`, `chat_thread_cap_reached`, `file_upload_rejected`; остальные — `success` |
| Категории событий | «Аутентификация и доступ», «Пользователи и роли», «Отчёты и чувствительные просмотры», «Шифрование и ИБ-операции», «Контрагенты и тендеры», «ИИ и автоматизация», «Справочники и бюджет», «Портфели и проекты», «Шаблоны», «Биллинг», «Наблюдаемость» |
| Поля условий правил | `action`, `entity_type`, `entity_id`, `ip_address`, `user_id`; операции `eq`, `ne`, `in` |
| Уровни логов | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` (уровень паттерна — минимальный: `WARNING` = WARNING и выше) |
| Виды записей таймлайна | `status_changed`, `severity_changed`, `assigned`, `resources_changed`, `comment`, `rkn_package`, `email` |

### Объекты

**`Event`** (событие журнала):

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | id события |
| `user_id` | integer \| null | кто совершил |
| `incident_ids` | integer[] | инциденты, поднятые этим событием |
| `action` | string | код события (каталог — `GET /security/events/catalog`) |
| `outcome` | string | `success` \| `failure` |
| `actor` | string | ФИО; «user #N», если пользователь удалён; «—», если нет пользователя |
| `entity_type` / `entity_id` | string \| null / integer \| null | ресурс |
| `ip_address` | string \| null | IP клиента |
| `details` | object \| null | детали события (JSON) |
| `created_at` | string | `ДД.ММ.ГГГГ ЧЧ:ММ:СС` МСК |

**`Incident`**:

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | номер инцидента |
| `title` | string | заголовок (из правила) |
| `severity` | string | критичность |
| `status` | string | `open` \| `investigating` \| `closed` |
| `description` | string \| null | «Создано правилом «…» (#id) по событию #id.» |
| `related_event_ids` | integer[] | события, на которых сработало правило (дополняются, пока инцидент не закрыт) |
| `source_rule_id` | integer \| null | правило-источник; `null` у заведённых вручную исторически |
| `assignee_id` / `assignee_name` | integer \| null / string \| null | ответственный (имя — только в реестре) |
| `rule_title` | string \| null | название правила (только в реестре) |
| `affected_resources` | object[] | затронутые ресурсы (произвольные объекты, задаёт оператор) |
| `created_at` / `closed_at` | string \| null | время |

**`IncidentRule`**: `id`, `action` (код события-триггера), `severity`, `title`, `enabled`, `conditions` (`[{field, op, value}]`), `window_minutes` (integer \| null), `threshold` (integer \| null), `recipients` (string[] — внешние адреса), `created_at`.

**`LogPattern`**: `id`, `name`, `level`, `match`, `is_regex`, `window_minutes`, `threshold`, `dedupe_key` (если не задан — показывается фактический `pattern:<id>`), `enabled`, `channel` (всегда `telegram` — единственный канал технических оповещений), `created_at`.

**`RknPackageInfo`**: `id`, `incident_id`, `version`, `generated_by` (id пользователя), `generated_at` (ISO 8601 UTC).

### Фильтры журнала (общие для `GET /security/events` и трёх выгрузок)

Один набор фильтров — выгрузка обязана уносить ровно тот срез, что на экране.

| Параметр | Тип | Смысл |
|---|---|---|
| `action` | string | код события или несколько через запятую (отметка категории = все её события) |
| `outcome` | string | `success` \| `failure`; иное значение игнорируется |
| `date_from` / `date_to` | date `ГГГГ-ММ-ДД` | московские сутки включительно |
| `ip` | string | префикс IP (поиск по подсети) |
| `user_id` | integer | точный пользователь |
| `user` | string | подстрока ФИО или e-mail пользователя, без учёта регистра |
| `entity_type` | string | тип ресурса |
| `q` | string | подстрока в деталях события (без учёта регистра) |
| `incident_id` | integer | только события, связанные с инцидентом |
| `sort` | string | `created_at`, `action`, `actor`, `outcome`, `entity_type`, `ip`; префикс `-` — по убыванию. По умолчанию и при неизвестном поле — новые сверху. Вторичный ключ — id |

---

### GET /security/events

**Назначение.** Страница журнала событий ИБ.

**Доступ.** `security.manage`.

**Параметры запроса.** Фильтры журнала (см. выше) и:

| Параметр | Тип | По умолчанию | Ограничения |
|---|---|---|---|
| `limit` | integer | 100 | ≤ 500 |
| `offset` | integer | 0 | ≥ 0 |

**Ответ `200`:** `{items: Event[], total, limit, offset}`.

**Ошибки.** 401, 403, 422 (неверная дата, `limit` > 500).

**Пример.**

```http
GET /api/security/events?outcome=failure&date_from=2026-10-01&limit=2
```

```json
{"items": [{"id": 10317, "user_id": 77, "incident_ids": [], "action": "permission_denied", "outcome": "failure",
  "actor": "Петров Пётр", "entity_type": null, "entity_id": null, "ip_address": "203.0.113.10",
  "details": {"required": "reports.view"}, "created_at": "15.10.2026 10:00:27"}],
 "total": 41, "limit": 2, "offset": 0}
```

**Побочные эффекты.** Нет.

---

### GET /security/events/catalog

**Назначение.** Каталог событий журнала для фильтров экрана и конструктора правил.

**Доступ.** `security.manage`. **Параметры.** Нет.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `actions` | string[] | отсортированные коды событий (совместимость со старым клиентом) |
| `events` | `[{code, label, category, severity}]` | полный каталог с русскими подписями |
| `categories` | string[] | категории в порядке показа |
| `severities` | string[] | `low`, `medium`, `high`, `critical` |

Если заявки с лендинга выключены (`LEADS_ENABLED=false`), событие `lead_submitted` из каталога скрывается. В демо-конфигурации — 77 событий; полный перечень — в бизнес-логике.

**Ошибки.** 401, 403.

**Пример (фрагмент).**

```json
{"actions": ["account_locked", "login_failed", "login_success"],
 "events": [{"code": "login_failed", "label": "Неуспешный вход", "category": "Аутентификация и доступ", "severity": "medium"}],
 "categories": ["Аутентификация и доступ", "Пользователи и роли"], "severities": ["low", "medium", "high", "critical"]}
```

**Побочные эффекты.** Нет.

---

### GET /security/events/export

**Назначение.** Выгрузить срез журнала в CSV для Excel.

**Доступ.** `security.manage`.

**Параметры запроса.** Фильтры журнала. Пагинации нет: выгружается не больше 500 событий среза (первые по выбранной сортировке).

**Ответ `200`.** `text/csv; charset=utf-8`, `Content-Disposition: attachment; filename=security_events.csv`. Файл: BOM UTF-8, разделитель `;`; первая строка — гриф «ГРИФ: КОНФИДЕНЦИАЛЬНО (ПДн). Выгрузка содержит персональные данные / события безопасности. Обработка только уполномоченными лицами оператора.», затем пустая строка, затем заголовки «Дата/время; Событие; Пользователь; Тип ресурса; ID ресурса; IP; Детали» и строки событий (время МСК `ДД.ММ.ГГГГ ЧЧ:ММ:СС`, код события, ФИО или id).

**Ошибки.** 401, 403, 422.

**Пример.** `GET /api/security/events/export?action=login_failed,account_locked` → файл `security_events.csv`.

**Побочные эффекты.** Событие журнала `report_export` `{"report": "security_events", "stamp": "pdn_confidential"}`.

---

### GET /security/events/export.jsonl

**Назначение.** Машинная выгрузка среза журнала — объект на строку, тот же формат, что уходит в SIEM.

**Доступ.** `security.manage`.

**Параметры запроса.** Фильтры журнала; потолок — 500 событий.

**Ответ `200`.** `application/x-ndjson; charset=utf-8`, файл `security_events.jsonl`. Первая строка — служебная: `{"_meta": {"stamp": "<гриф ПДн>", "exported_at": "<ISO UTC>", "exported_by_user_id": N, "filters": {…переданные фильтры…}, "count": N}}`. Далее по строке на событие:

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | |
| `created_at` | string | ISO 8601 |
| `action`, `outcome` | string | |
| `category`, `severity` | string | из каталога |
| `user_id`, `entity_type`, `entity_id` | | |
| `ip_address` | string \| null | |
| `correlation_id` | string \| null | id запроса (`X-Request-Id`), в котором записано событие |
| `details` | object \| null | |

**Ошибки.** 401, 403, 422.

**Пример строки.**

```json
{"id": 10317, "created_at": "2026-10-15T07:00:27.120000+00:00", "action": "permission_denied", "outcome": "failure",
 "category": "Аутентификация и доступ", "severity": "medium", "user_id": 77, "entity_type": null, "entity_id": null,
 "ip_address": "203.0.113.10", "correlation_id": "9f2c…", "details": {"required": "reports.view"}}
```

**Побочные эффекты.** `report_export` `{"report": "security_events.jsonl", "stamp": "pdn_confidential", "rows": N}`.

---

### GET /security/events/export.txt

**Назначение.** Человекочитаемая выгрузка среза — приложение к акту, письму, тикету.

**Доступ.** `security.manage`.

**Параметры запроса.** Фильтры журнала; потолок — 500 событий.

**Ответ `200`.** `text/plain; charset=utf-8`, файл `security_events.txt`. Шапка: гриф ПДн; «Журнал событий безопасности. Выгружено: ДД.ММ.ГГГГ ЧЧ:ММ:СС МСК, событий: N.»; пустая строка. Далее строка на событие: `время | код | исход | пользователь | IP | тип#id | {детали JSON}` (отсутствующее — «—»).

**Ошибки.** 401, 403, 422.

**Пример строки.** `15.10.2026 10:00:27 | permission_denied | failure | Петров Пётр | 203.0.113.10 | — | {"required": "reports.view"}`

**Побочные эффекты.** `report_export` `{"report": "security_events.txt", "stamp": "pdn_confidential", "rows": N}`.

---

### GET /security/siem

**Назначение.** Текущие настройки и состояние передачи журнала во внешний SIEM. Секрет авторизации не возвращается.

**Доступ.** `security.manage`. **Параметры.** Нет.

**Ответ `200`:** `{"item": {...}}`:

| Поле | Тип | Смысл |
|---|---|---|
| `enabled` | boolean | передача включена (по умолчанию `false`) |
| `url` | string \| null | адрес HTTP-коллектора |
| `has_auth_header` | boolean | задано значение заголовка `Authorization` |
| `batch_size` | integer | размер пачки (по умолчанию 200) |
| `last_sent_event_id` | integer | курсор: id последнего отправленного события |
| `last_sent_at` | string \| null | ISO 8601 последней успешной отправки |
| `last_error` | string \| null | последняя ошибка (до 500 символов) |

**Ошибки.** 401, 403.

**Пример.** `{"item": {"enabled": false, "url": null, "has_auth_header": false, "batch_size": 200, "last_sent_event_id": 0, "last_sent_at": null, "last_error": null}}`

**Побочные эффекты.** При первом обращении создаётся строка настроек по умолчанию.

---

### PUT /security/siem

**Назначение.** Изменить настройки передачи; меняются только присланные поля.

**Доступ.** `security.manage`.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения / смысл |
|---|---|---|---|
| `enabled` | boolean | нет | включить передачу; требует заданного `url` (в запросе или ранее) |
| `url` | string | нет | адрес коллектора; обрезается по краям |
| `auth_header` | string | нет | полное значение заголовка `Authorization` (например, `Bearer …`); хранится зашифрованным |
| `batch_size` | integer | нет | 1–1000 |
| `last_sent_event_id` | integer | нет | ≥ 0; перемотка курсора («отправить заново начиная с события N+1») |

**Ответ `200`:** `{"item": {...}}` — как в `GET`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | `enabled=true`, а URL не задан | «Укажите URL коллектора перед включением» |
| 422 | `batch_size` вне 1–1000, `last_sent_event_id` < 0 | стандартная ошибка валидации |
| 401 / 403 | см. общее | |

**Пример.** `{"enabled": true, "url": "https://siem.example.com/ingest", "auth_header": "Bearer …", "batch_size": 500}`

**Побочные эффекты.** Фоновая периодическая задача (каждые `SIEM_DRAIN_INTERVAL` секунд, по умолчанию 15) начинает/прекращает отправку. В журнал ИБ изменение не пишется (см. открытые вопросы).

---

### POST /security/siem/drain

**Назначение.** Отправить следующую пачку событий прямо сейчас — кнопка «Проверить соединение». Тот же путь, что у фоновой задачи.

**Доступ.** `security.manage`. **Параметры / тело.** Нет.

**Механика.** Берутся события с `id > last_sent_event_id` по возрастанию, не больше `batch_size` (1–1000); отправляются одним `POST` на `url` с `Content-Type: application/x-ndjson` (строки — как в JSONL-выгрузке, без `_meta`) и заголовком `Authorization`, если задан; тайм-аут 15 с. Курсор сдвигается только после успешного ответа (2xx) — недоступный коллектор ничего не теряет.

**Ответ `200`** (одно из):
- `{"skipped": "выключено"}` — передача выключена или нет URL;
- `{"sent": 0}` — новых событий нет;
- `{"sent": N, "last_sent_event_id": M}` — успех;
- `{"sent": 0, "error": "<класс>: <текст>"}` — ошибка отправки (сохраняется в `last_error`).

**Ошибки.** 401, 403.

**Пример.** → `{"sent": 200, "last_sent_event_id": 10318}`

**Побочные эффекты.** Сетевой запрос к коллектору; обновление курсора, `last_sent_at`, `last_error`.

---

### GET /security/crypto/status

**Назначение.** Статус провайдера полевого шифрования и реестр шифруемых/намеренно открытых полей — для проверяющих.

**Доступ.** `security.manage`. **Параметры.** Нет.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `provider_id` | string | идентификатор провайдера (в текущей реализации — `fernet`) |
| `algorithm` | string | описание алгоритма |
| `is_certified_skzi` | boolean | сертифицированное СКЗИ (сейчас `false`) |
| `cipher_format` | string | формат шифртекста `v1:<provider>:<payload>` |
| `self_test_ok` | boolean | результат самопроверки «зашифровать → расшифровать» |
| `self_test` | object | `{ok, provider_id, algorithm, is_certified_skzi, roundtrip}` |
| `encrypted_fields` | `[{table, column, reason}]` | поля, шифруемые в БД (ФИО руководителя, телефон и e-mail контрагента; ФИО, телефон, e-mail контакта; секрет MFA; пароль SMTP; токен ссылки саморегистрации; секрет SIEM хранится так же) |
| `plaintext_by_design` | `[{table, column, reason}]` | намеренно открытые поля (ИНН, ОГРН — поиск и автопроверки) |

**Ошибки.** 401, 403.

**Пример (фрагмент).**

```json
{"provider_id": "fernet", "algorithm": "Fernet (AES-128-CBC + HMAC)", "is_certified_skzi": false,
 "cipher_format": "v1:<provider>:<payload>", "self_test_ok": true,
 "self_test": {"ok": true, "provider_id": "fernet", "is_certified_skzi": false, "roundtrip": true},
 "encrypted_fields": [{"table": "users", "column": "mfa_secret", "reason": "секрет второго фактора"}],
 "plaintext_by_design": [{"table": "suppliers", "column": "inn", "reason": "поиск и автопроверки"}]}
```

**Побочные эффекты.** Нет (самопроверка выполняется в памяти, без записи в журнал).

---

### POST /security/crypto/self-test

**Назначение.** Выполнить самопроверку шифрования и зафиксировать факт проверки в журнале.

**Доступ.** `security.manage`. **Параметры / тело.** Нет.

**Ответ `200`:** объект самопроверки `{ok, provider_id, algorithm, is_certified_skzi, roundtrip}`.

**Ошибки.** 401, 403.

**Пример.** → `{"ok": true, "provider_id": "fernet", "algorithm": "Fernet (AES-128-CBC + HMAC)", "is_certified_skzi": false, "roundtrip": true}`

**Побочные эффекты.** Событие журнала `crypto_selftest` `{"ok": …, "provider": …}`.

---

### GET /security/incidents

**Назначение.** Реестр инцидентов — очередь: незакрытые выше закрытых, внутри — по критичности (critical → low), затем новые сверху.

**Доступ.** `security.manage`.

**Параметры запроса:**

| Параметр | Тип | Смысл |
|---|---|---|
| `q` | string | подстрока в заголовке или описании; число (можно с `№`/`#`) — ещё и поиск по номеру |
| `severity` | string | одна или несколько через запятую |
| `status` | string | один или несколько через запятую |
| `assignee_id` | integer | ответственный |
| `unassigned` | boolean | только без ответственного (если не задан `assignee_id`) |
| `mine` | boolean | только назначенные на текущего пользователя (перекрывает `assignee_id`) |
| `limit` / `offset` | integer | 1–200 / ≥ 0 |

**Ответ `200`:** `{items: Incident[], total, limit, offset, open_count}`, где `open_count` — число незакрытых инцидентов во всей таблице (для бейджа вкладки), независимо от фильтров.

**Ошибки.** 401, 403, 422.

**Пример.**

```json
{"items": [{"id": 12, "title": "Подбор пароля", "severity": "high", "status": "open",
  "description": "Создано правилом «Подбор пароля» (#3) по событию #10251.", "related_event_ids": [10251, 10252],
  "source_rule_id": 3, "assignee_id": null, "assignee_name": null, "rule_title": "Подбор пароля",
  "affected_resources": [], "created_at": "14.10.2026 22:41:05", "closed_at": null}],
 "total": 1, "limit": 200, "offset": 0, "open_count": 1}
```

**Побочные эффекты.** Нет.

---

### GET /security/incidents/{incident_id}

**Назначение.** Карточка инцидента.

**Доступ.** `security.manage`.

**Параметры пути.** `incident_id` — integer.

**Ответ `200`:** `{"item": Incident}` (без `assignee_name` и `rule_title` — они `null`).

**Ошибки.** 404 «Инцидент не найден»; 401 / 403.

**Пример.** `GET /api/security/incidents/12` → `{"item": {"id": 12, "title": "Подбор пароля", …}}`

**Побочные эффекты.** Нет.

---

### PATCH /security/incidents/{incident_id}

**Назначение.** Перевести статус, изменить критичность, назначить/снять ответственного, изменить состав затронутых ресурсов. Каждое изменение пишется в таймлайн.

**Доступ.** `security.manage`.

**Параметры пути.** `incident_id` — integer.

**Тело запроса** (только присланные поля):

| Поле | Тип | Ограничения / смысл |
|---|---|---|
| `status` | string | `open` \| `investigating` \| `closed`; при `closed` ставится `closed_at`, при выходе из `closed` — сбрасывается |
| `severity` | string | `low` \| `medium` \| `high` \| `critical` |
| `assignee_id` | integer \| null | существующий пользователь; `null`/0 — снять |
| `affected_resources` | object[] | заменяет список целиком |

**Ответ `200`:** `{"item": Incident}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 404 | инцидента нет | «Инцидент не найден» |
| 404 | ответственный не найден | «Пользователь не найден» |
| 400 | неверный статус | «status: open\|investigating\|closed» |
| 400 | неверная критичность | «severity: low\|medium\|high\|critical» |
| 401 / 403 | см. общее | |

**Пример.** `{"status": "investigating", "assignee_id": 77}`

**Побочные эффекты.**
- Записи таймлайна: `status_changed` («Статус: open → investigating»), `severity_changed`, `assigned` («Ответственный: <ФИО>» / «Ответственный снят»; пишется при каждом присланном `assignee_id`), `resources_changed`.
- Событие журнала с кодом `incident_created` и деталями `{"changed": [поля], "status": "<новый>"}` (правила по нему не применяются). См. открытые вопросы о коде события.

---

### GET /security/incidents/{incident_id}/context

**Назначение.** Чем поднят инцидент: правило-источник и связанные события журнала (для раскрытой карточки).

**Доступ.** `security.manage`.

**Параметры пути.** `incident_id`.

**Ответ `200`:** `{"rule": IncidentRule | null, "events": Event[]}`; события — новые сверху. `rule: null` — законное состояние (инцидент без правила или правило удалено).

**Ошибки.** 404 «Инцидент не найден»; 401 / 403.

**Пример.** → `{"rule": {"id": 3, "action": "login_failed", "severity": "high", "title": "Подбор пароля", "enabled": true, "conditions": [], "window_minutes": 10, "threshold": 5, "recipients": [], "created_at": "01.10.2026 09:00:00"}, "events": [ … ]}`

**Побочные эффекты.** Нет.

---

### GET /security/incidents/{incident_id}/timeline

**Назначение.** Лента расследования по инциденту, по времени (старые сверху).

**Доступ.** `security.manage`.

**Параметры.** `incident_id` (путь); `limit` (1–200), `offset`.

**Ответ `200`:** конверт `{items, total, limit, offset}`, элемент: `{id, kind, author (ФИО или null для системных записей), text, payload (object|null — например, для `email`: {delivered, failed}), created_at}`.

Существование инцидента не проверяется: для несуществующего id — пустая страница.

**Ошибки.** 401, 403, 422.

**Пример.** → `{"items": [{"id": 1, "kind": "status_changed", "author": "Петров Пётр", "text": "Статус: open → investigating", "payload": null, "created_at": "15.10.2026 09:10:00"}], "total": 1, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### POST /security/incidents/{incident_id}/timeline

**Назначение.** Добавить комментарий оператора в таймлайн.

**Доступ.** `security.manage`.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения |
|---|---|---|---|
| `text` | string | да | 1–4000 символов; обрезается по краям |

**Ответ `201`:** `{"ok": true}`.

**Ошибки.** 404 «Инцидент не найден»; 422 — длина; 401 / 403.

**Пример.** `{"text": "Источник — перебор с одного IP, адрес заблокирован на межсетевом экране."}` → `{"ok": true}`

**Побочные эффекты.** Запись таймлайна `comment` с автором. В журнал ИБ не пишется.

---

### GET /security/incident-rules

**Назначение.** Список правил автоинцидентов, новые сверху.

**Доступ.** `security.manage`. **Параметры.** `limit` (1–200), `offset`.

**Ответ `200`:** `{items: IncidentRule[], total, limit, offset}`.

**Ошибки.** 401, 403, 422.

**Пример.** → `{"items": [], "total": 0, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### POST /security/incident-rules

**Назначение.** Создать правило: «при событии `action`, удовлетворяющем условиям, — N раз за M минут — завести инцидент и уведомить».

**Доступ.** `security.manage`.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения / смысл |
|---|---|---|---|
| `action` | string | да | 2–64 символа; код из каталога событий |
| `severity` | string | нет (`medium`) | критичность создаваемого инцидента |
| `title` | string | да | 3–500 символов; заголовок инцидента |
| `enabled` | boolean | нет (`true`) | |
| `conditions` | `[{field, op, value}]` | нет | `field` из списка полей условий; `op` — `eq` (по умолчанию), `ne`, `in` (для `in` `value` — массив) |
| `window_minutes` | integer \| null | нет | 1–1440 |
| `threshold` | integer \| null | нет | 1–1000; порог ≤ 1 или пустое окно — срабатывание на каждое событие |
| `recipients` | string[] | нет | внешние адреса для писем (каждый обязан содержать `@`); пустые отбрасываются |

**Ответ `201`:** `{"item": IncidentRule}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | неизвестный код события | «Неизвестный тип события: <код>» |
| 400 | неверная критичность | «severity: low\|medium\|high\|critical» |
| 400 | неподдерживаемое поле условия | «Условие по полю <поле> не поддерживается» |
| 400 | неверная операция | «Операция <op>: допустимы eq\|ne\|in» |
| 400 | `in` без списка | «Операция in ждёт список значений» |
| 400 | адрес без `@` | «Не похоже на адрес: <адрес>» |
| 422 | длины и диапазоны | стандартная ошибка валидации |
| 401 / 403 | см. общее | |

**Пример.**

```json
{"action": "login_failed", "severity": "high", "title": "Подбор пароля", "window_minutes": 10, "threshold": 5,
 "conditions": [{"field": "entity_type", "op": "eq", "value": "user"}], "recipients": ["user@example.com"]}
```

**Побочные эффекты.** С момента создания каждое новое событие журнала с этим кодом проверяется правилом (синхронно при записи события):
- если условия совпали и порог в окне достигнут, а открытого (не `closed`) инцидента этого правила нет — создаётся инцидент (`title`, `severity` из правила), событие журнала `incident_created` (`auto: true`), уведомления: запись в центре уведомлений и письмо (по личной настройке уведомлений) пользователям с ролью «служба безопасности» и администраторам через общий каталог уведомлений (тип «Инциденты информационной безопасности», ссылка на журнал безопасности; письмо: тема «Инцидент ИБ (<критичность>): <заголовок>»); на адреса `recipients` — письмо фоновой задачей с повторами (результат — запись `email` в таймлайне);
- если открытый инцидент правила уже есть — событие добавляется в его `related_event_ids`.

Изменение правил в журнал ИБ не пишется.

---

### PATCH /security/incident-rules/{rule_id}

**Назначение.** Изменить правило (кроме кода события-триггера).

**Доступ.** `security.manage`.

**Параметры пути.** `rule_id` — integer.

**Тело запроса** (все поля необязательны; `null` = не менять):

| Поле | Тип | Ограничения |
|---|---|---|
| `severity` | string | как при создании |
| `title` | string | после обрезки ≥ 3 символов |
| `enabled` | boolean | |
| `conditions` | array | как при создании, заменяет целиком |
| `window_minutes` | integer | 0–1440; 0 — снять окно |
| `threshold` | integer | 0–1000; 0 — снять порог |
| `recipients` | string[] | заменяет целиком |

**Ответ `200`:** `{"item": IncidentRule}`.

**Ошибки.** 404 «Правило не найдено»; 400 «title слишком короткий», ошибки severity/условий/адресов как при создании; 422; 401 / 403.

**Пример.** `{"enabled": false}`

**Побочные эффекты.** Нет (действует на будущие события).

---

### DELETE /security/incident-rules/{rule_id}

**Назначение.** Удалить правило.

**Доступ.** `security.manage`.

**Параметры пути.** `rule_id`.

**Ответ `204`** без тела.

**Ошибки.** 404 «Правило не найдено»; 401 / 403.

**Пример.** `DELETE /api/security/incident-rules/3` → `204`.

**Побочные эффекты.** Уже созданные инциденты остаются; их `context.rule` становится `null` (см. открытые вопросы о ссылочной целостности).

---

### POST /security/incident-rules/{rule_id}/test

**Назначение.** Прогнать правило по последним событиям журнала, ничего не создавая, — оператор видит, что поймает правило, до включения.

**Доступ.** `security.manage`.

**Параметры пути.** `rule_id`. **Тело.** Нет.

**Механика.** Берутся до 500 последних событий с кодом правила (в пределах окна, если оно задано), к ним применяются условия.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `matched` | integer | сколько событий совпало |
| `threshold` | integer | порог (0, если не задан) |
| `window_minutes` | integer \| null | окно |
| `would_fire` | boolean | при пороге > 1 и окне — `matched ≥ threshold`, иначе — есть хотя бы одно совпадение |
| `recipients` | string[] | внешние адреса правила |
| `sample` | object[] | до 20 совпавших событий в формате JSONL-выгрузки |
| `scanned_limit` | integer | 500 |

**Ошибки.** 404 «Правило не найдено»; 401 / 403.

**Пример.** → `{"matched": 7, "threshold": 5, "window_minutes": 10, "would_fire": true, "recipients": [], "sample": [ … ], "scanned_limit": 500}`

**Побочные эффекты.** Нет.

---

### GET /security/log-patterns

**Назначение.** Правила-паттерны технических логов: подстрока или регулярное выражение по строке лога приложения → оповещение в технический канал (сейчас — Telegram-бот).

**Доступ.** `security.manage`. **Параметры.** `limit` (1–200), `offset`. Порядок — по id.

**Ответ `200`:** `{items: LogPattern[], total, limit, offset}`.

**Ошибки.** 401, 403, 422.

**Пример.**

```json
{"items": [{"id": 1, "name": "Ошибки и трейсбэки", "level": "ERROR", "match": "ERROR|Traceback", "is_regex": true,
  "window_minutes": 5, "threshold": 1, "dedupe_key": "pattern:1", "enabled": true, "channel": "telegram",
  "created_at": "18.09.2026 14:03:57"}], "total": 4, "limit": 1, "offset": 0}
```

**Побочные эффекты.** Нет.

---

### POST /security/log-patterns

**Назначение.** Создать паттерн.

**Доступ.** `security.manage`.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения / смысл |
|---|---|---|---|
| `name` | string | да | 1–200 |
| `match` | string | да | 1–2000; подстрока или регулярное выражение |
| `level` | string | нет (`ERROR`) | уровень из справочника (без учёта регистра, хранится в верхнем) |
| `is_regex` | boolean | нет (`false`) | регулярное выражение обязано компилироваться |
| `window_minutes` | integer | нет (5) | 1–1440 |
| `threshold` | integer | нет (1) | 1–1000; 1 — на каждое совпадение |
| `dedupe_key` | string \| null | нет | ≤ 200; ключ антиспама оповещений; пусто — `pattern:<id>` |
| `enabled` | boolean | нет (`true`) | |

Всего правил — не больше 50 (оценка идёт на каждую строку лога).

**Ответ `201`:** `{"item": LogPattern}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | неизвестный уровень | «Неизвестный уровень: <уровень>» |
| 400 | регулярка не компилируется | «Некорректное регулярное выражение: <ошибка>» |
| 400 | уже 50 правил | «Больше 50 правил не поддерживается — отключите ненужные» |
| 422 | длины и диапазоны | |
| 401 / 403 | см. общее | |

**Пример.** `{"name": "Таймауты БД", "match": "statement timeout", "level": "WARNING", "window_minutes": 10, "threshold": 3}`

**Побочные эффекты.** Событие журнала `log_pattern_rule_changed` `{"change": "created"}`. Обработчик логов подхватывает новый набор правил при очередном обновлении снимка (периодическая задача, интервал `LOG_PATTERNS_REFRESH_INTERVAL`), работает только при `LOG_PATTERNS_ENABLED=true`. При срабатывании («N совпадений за M минут», подсчёт в памяти процесса) оповещение уходит в технический канал с антиспамом по `dedupe_key` (окно молчания `ALERTS_DEDUPE_SECONDS`).

---

### PATCH /security/log-patterns/{rule_id}

**Назначение.** Изменить паттерн (только присланные поля).

**Доступ.** `security.manage`.

**Параметры пути.** `rule_id`.

**Тело запроса.** Те же поля, что при создании, все необязательны, с теми же ограничениями. Проверка уровня и регулярки — по итоговым значениям (присланные + сохранённые).

**Ответ `200`:** `{"item": LogPattern}`.

**Ошибки.** 404 «Правило не найдено»; 400 — уровень/регулярка; 422; 401 / 403.

**Пример.** `{"enabled": false}`

**Побочные эффекты.** `log_pattern_rule_changed` `{"change": "updated"}`.

---

### DELETE /security/log-patterns/{rule_id}

**Назначение.** Удалить паттерн.

**Доступ.** `security.manage`.

**Ответ `204`** без тела.

**Ошибки.** 404 «Правило не найдено»; 401 / 403.

**Пример.** `DELETE /api/security/log-patterns/4` → `204`.

**Побочные эффекты.** `log_pattern_rule_changed` `{"change": "deleted"}`.

---

### POST /security/log-patterns/{rule_id}/test

**Назначение.** Предпросмотр «сработало бы»: правило прогоняется по строкам, которые прислал оператор (технические логи в БД не хранятся). Ничего не отправляется.

**Доступ.** `security.manage`.

**Параметры.** `rule_id` (путь); `lines` — строки лога, **query-параметр, повторяемый** (`?lines=…&lines=…`). Тела нет. Уровень строки при предпросмотре не учитывается — только совпадение текста.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `matched` | integer | совпавших строк |
| `scanned` | integer | всего строк |
| `threshold`, `window_minutes` | integer | параметры правила |
| `would_fire` | boolean | `matched ≥ threshold` |
| `too_broad` | boolean | совпали все строки (предупреждение о слишком широком правиле) |
| `dedupe_key` | string | фактический ключ антиспама |
| `sample` | string[] | до 20 совпавших строк |

**Ошибки.** 404 «Правило не найдено»; 400 «Некорректное регулярное выражение: …»; 401 / 403.

**Пример.** `POST /api/security/log-patterns/1/test?lines=ERROR%20db%20down&lines=INFO%20ok` → `{"matched": 1, "scanned": 2, "threshold": 1, "window_minutes": 5, "would_fire": true, "too_broad": false, "dedupe_key": "pattern:1", "sample": ["ERROR db down"]}`

**Побочные эффекты.** Нет.

---

### POST /security/incidents/{incident_id}/rkn-package

**Назначение.** Зафиксировать очередную версию пакета сведений для уведомления Роскомнадзора об инциденте с ПДн (152-ФЗ). Система только готовит комплект — подача ручная. Версии неизменяемы, каждая следующая — `version + 1`.

**Доступ.** `security.manage`.

**Параметры пути.** `incident_id`. **Тело.** Нет.

**Содержимое пакета** (снимок на момент формирования):

| Поле | Смысл |
|---|---|
| `document_type` | `pdn_incident_notification_draft` |
| `law_basis` | «ФЗ-152 «О персональных данных»» |
| `generated_at`, `generated_by_user_id` | когда и кем |
| `disclaimer` | «Черновик для оператора ПДн. Не является автоматической отправкой в Роскомнадзор. Сроки и состав уведомления уточняются по актуальной редакции 152-ФЗ.» |
| `data_categories` | категории данных (ПДн представителей поставщиков; события журнала) |
| `incident` | карточка инцидента (`Incident`) |
| `related_security_events` | связанные события (`Event[]`, старые сверху) |
| `operator_checklist` | 4 пункта-напоминания оператору |

**Ответ `201`:** `{"item": RknPackageInfo}`.

**Ошибки.** 404 «Инцидент не найден»; 401 / 403.

**Пример.** → `{"item": {"id": 5, "incident_id": 12, "version": 2, "generated_by": 77, "generated_at": "2026-10-15T07:20:00+00:00"}}`

**Побочные эффекты.** Запись таймлайна `rkn_package` («Сформирован пакет для РКН, версия N»); событие журнала `rkn_draft_exported` `{"incident_id": …, "version": …}`.

---

### GET /security/incidents/{incident_id}/rkn-packages

**Назначение.** Список версий пакетов по инциденту, новые сверху.

**Доступ.** `security.manage`.

**Параметры.** `incident_id` (путь); `limit` (1–200), `offset`.

**Ответ `200`:** `{items: RknPackageInfo[], total, limit, offset}`. Для несуществующего инцидента — пустой список.

**Ошибки.** 401, 403, 422.

**Пример.** → `{"items": [{"id": 5, "incident_id": 12, "version": 2, "generated_by": 77, "generated_at": "2026-10-15T07:20:00+00:00"}], "total": 2, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### GET /security/rkn-packages/{package_id}

**Назначение.** Скачать зафиксированную версию пакета.

**Доступ.** `security.manage`.

**Параметры пути.** `package_id`.

**Ответ `200`.** `application/json; charset=utf-8`, `Content-Disposition: attachment; filename=rkn_incident_<incident_id>_v<version>.json`; тело — содержимое пакета (см. выше), JSON с отступами, кириллица без экранирования.

**Ошибки.** 404 «Пакет не найден»; 401 / 403.

**Пример.** `GET /api/security/rkn-packages/5` → файл `rkn_incident_12_v2.json`.

**Побочные эффекты.** Нет (событие журнала пишется при формировании, а не при скачивании).

---

### GET /security/rkn-draft

**Назначение.** Устаревший алиас: отдаёт последнюю зафиксированную версию пакета по инциденту. Пересборки «на лету» нет. В новой системе можно не реализовывать, если клиент переведён на `rkn-packages` (см. открытые вопросы).

**Доступ.** `security.manage`.

**Параметры запроса:**

| Параметр | Тип | Обяз. | Смысл |
|---|---|---|---|
| `incident_id` | integer | да | инцидент |

**Ответ `200`.** Файл как у `GET /security/rkn-packages/{package_id}` для максимальной версии.

**Ошибки.** 404 «По инциденту нет ни одной версии пакета — сформируйте её (POST .../rkn-package)»; 422 — нет `incident_id`; 401 / 403.

**Пример.** `GET /api/security/rkn-draft?incident_id=12` → `rkn_incident_12_v2.json`.

**Побочные эффекты.** Нет.
