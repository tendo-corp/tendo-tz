# API: контрагенты (suppliers)

Ручки раздела «Контрагенты» для сотрудников: реестр с серверными фильтрами и фасетами, ручное заведение и проверка ИНН, очереди СБ и юриста, карточка (реквизиты, документы, автопроверки по открытым реестрам, лента, участие в тендерах, история решений), независимые оси статусной модели (СБ / юрист / финансы → сводный статус, пул, дата аккредитации), выручка по годам, опыт работ, контакты, пользователи ЛК, корзина. Регистрация по ссылке-приглашению (анкета) и личный кабинет контрагента (`/me/*`) — в [endpoints-suppliers-registration.md](endpoints-suppliers-registration.md). Общие соглашения (аутентификация, формат ошибок, пагинация, SSE) — в [overview.md](overview.md).

Связанные разделы: бизнес-логика — [../02-business-logic/suppliers.md](../02-business-logic/suppliers.md), приглашение в тендеры — [../02-business-logic/tenders-invitations.md](../02-business-logic/tenders-invitations.md); экраны — [../03-screens/suppliers.md](../03-screens/suppliers.md), [../03-screens/supplier-card.md](../03-screens/supplier-card.md), [../03-screens/register.md](../03-screens/register.md), [../03-screens/cabinet.md](../03-screens/cabinet.md); файлы и прямая загрузка — [endpoints-files.md](endpoints-files.md); уведомления — [endpoints-notifications.md](endpoints-notifications.md).

Сложность блока: **L** — реестр с фильтрами по зашифрованным полям (M), карточка и статусная модель с пересчётом и историей (L), автопроверки по внешним реестрам с деградацией (M), контакты/ЛК/корзина (S).

## Сводка

Файл [endpoints-suppliers-registration.md](endpoints-suppliers-registration.md) отмечен как «рег.».

| Метод | Путь | Кратко | Доступ |
|---|---|---|---|
| GET | `/suppliers` | Реестр контрагентов с фильтрами и поиском | сотрудник |
| GET | `/suppliers/new-count` | Счётчик новых анкет для бейджа | сотрудник |
| GET | `/suppliers/status-facets` | Все значения статусных фильтров со счётчиками | сотрудник |
| GET | `/suppliers/inn-lookup` | Занят ли ИНН + сведения ЕГРЮЛ для автозаполнения | сотрудник |
| POST | `/suppliers` | Завести контрагента вручную | сотрудник (решение «одобрен» — админ) |
| GET | `/suppliers/queue/security` | Очередь проверки СБ | сотрудник, кроме юриста |
| GET | `/suppliers/queue/lawyer` | Очередь юридической проверки | сотрудник, кроме СБ |
| GET | `/suppliers/{supplier_id}` | Карточка контрагента | сотрудник (финансы — право «Цены») |
| PATCH | `/suppliers/{supplier_id}` | Правка реквизитов, видов работ, тегов, квалификации | сотрудник (решение — админ) |
| GET | `/suppliers/{supplier_id}/checks` | Последние результаты автопроверок | сотрудник |
| POST | `/suppliers/{supplier_id}/checks/run` | Запустить автопроверки по реестрам | сотрудник |
| GET | `/suppliers/{supplier_id}/activity` | Лента событий контрагента по тендерам и аккредитации | сотрудник |
| GET | `/suppliers/{supplier_id}/participation` | Участие в тендерах и сводка побед | сотрудник |
| GET | `/suppliers/{supplier_id}/qualification-history` | История решений по квалификации | сотрудник |
| PATCH | `/suppliers/{supplier_id}/sb-status` | Статус проверки СБ | роль `security` или админ |
| PATCH | `/suppliers/{supplier_id}/legal-status` | Статус юридической проверки | роль `lawyer` или админ |
| PATCH | `/suppliers/{supplier_id}/finance-status` | Статус финансового блока (+ лимит) | право «Цены» или админ |
| PATCH | `/suppliers/{supplier_id}/pool` | Ручной перевод между пулами | тендерный специалист или админ |
| PATCH | `/suppliers/{supplier_id}/accredited-at` | Дата аккредитации | тендерный специалист или админ |
| PATCH | `/suppliers/{supplier_id}/accreditation-date` | То же (синоним пути) | тендерный специалист или админ |
| POST | `/suppliers/{supplier_id}/questionnaire/submit` | Отметить анкету поданной (ручной контрагент) | право `suppliers.manage` |
| GET | `/suppliers/{supplier_id}/revenues` | Выручка по годам | право «Цены» |
| PUT | `/suppliers/{supplier_id}/revenues` | Сохранить выручку по годам | право «Цены» |
| GET | `/suppliers/{supplier_id}/work-experience` | Договоры опыта по видам работ | сотрудник |
| POST | `/suppliers/{supplier_id}/work-experience` | Добавить договор опыта | тендерный специалист или админ |
| PUT | `/suppliers/{supplier_id}/work-experience/{experience_id}` | Изменить договор опыта | тендерный специалист или админ |
| DELETE | `/suppliers/{supplier_id}/work-experience/{experience_id}` | Удалить договор опыта | тендерный специалист или админ |
| GET | `/suppliers/{supplier_id}/contacts` | Контакты контрагента | сотрудник; контрагент — только свои |
| POST | `/suppliers/{supplier_id}/contacts` | Добавить контакт | сотрудник |
| PATCH | `/suppliers/{supplier_id}/contacts/{contact_id}` | Изменить контакт | сотрудник |
| DELETE | `/suppliers/{supplier_id}/contacts/{contact_id}` | Удалить контакт | сотрудник |
| GET | `/suppliers/{supplier_id}/cabinet-users` | Кто входит в ЛК контрагента | `suppliers.manage` |
| POST | `/suppliers/{supplier_id}/cabinet-users` | Привязать существующего пользователя к ЛК | `suppliers.manage` |
| DELETE | `/suppliers/{supplier_id}/cabinet-users/{user_id}` | Отвязать пользователя от ЛК | `suppliers.manage` |
| POST | `/suppliers/{supplier_id}/cabinet-login` | Создать учётную запись ЛК | `suppliers.manage` |
| DELETE | `/suppliers/{supplier_id}` | В корзину (мягкое удаление) | `suppliers.manage` |
| GET | `/suppliers/trash/list` | Корзина | `suppliers.manage` |
| POST | `/suppliers/{supplier_id}/restore` | Восстановить из корзины | `suppliers.manage` |
| DELETE | `/suppliers/{supplier_id}/hard` | Удалить из корзины насовсем | `suppliers.manage` |
| POST | `/suppliers/invitations` | рег. — создать ссылку на анкету | сотрудник |
| GET | `/suppliers/invitations/list` | рег. — журнал ссылок | сотрудник |
| POST | `/suppliers/invitations/{invitation_id}/regenerate` | рег. — перевыпустить ссылку | сотрудник |
| POST | `/suppliers/invitations/{invitation_id}/revoke` | рег. — отозвать ссылку | сотрудник |
| GET | `/suppliers/invitations/{token}/check` | рег. — проверить ссылку | публично |
| GET | `/suppliers/register/work-types` | рег. — виды работ для анкеты | публично по токену |
| POST | `/suppliers/register` | рег. — отправить анкету | публично по токену |
| POST | `/suppliers/register/{supplier_id}/attachments` | рег. — файл анкеты (multipart) | публично по токену |
| POST | `/suppliers/register/{supplier_id}/attachments/presign-upload` | рег. — прямая загрузка файла анкеты | публично по токену |
| POST | `/suppliers/register/{supplier_id}/attachments/confirm` | рег. — подтвердить прямую загрузку | публично по токену |
| DELETE | `/suppliers/register/{supplier_id}/attachments/uploads/{upload_id}` | рег. — отменить прямую загрузку | публично по токену |
| POST | `/suppliers/register/{supplier_id}/complete` | рег. — финализировать анкету | публично по токену |
| GET | `/me/supplier` | рег. — профиль организации в ЛК | контрагент |
| PATCH | `/me/supplier` | рег. — правка контактов и настроек ЛК | контрагент |
| GET | `/me/tenders` | рег. — мои тендеры | контрагент |
| GET | `/me/quotes` | рег. — мои КП | контрагент |
| GET | `/me/activity` | рег. — моя лента событий | контрагент |

## Доступ — общее

- **Сотрудник** — гейт overview §3 (роли `admin`, `tender_specialist`, `security`, `lawyer`); отказ контрагенту и лиду — 403 «Доступно только сотрудникам компании».
- **`suppliers.manage`** — право матрицы; по умолчанию выдано только администратору (матрицу обходит). Отказ — 403 «Недостаточно прав для этого действия — обратитесь к администратору системы» + `permission_denied` в журнале ИБ.
- **Право «Цены»** (`finance.view` или персональный флаг `can_view_finance`, администратор — всегда) решает, отдаются ли выручка и лимит; в реестре финансы не отдаются никогда.
- Ролевые проверки внутри ручек (оси статусов, пул, дата аккредитации, опыт работ) описаны у ручек.
- Для любого `{supplier_id}`: контрагента нет или он в корзине — **404 «Контрагент не найден»** (кроме ручек корзины).

## Справочники статусов

**Квалификация** (`status`, единая ось анкеты и решения тендерного комитета):

| Код | Подпись | Группа |
|---|---|---|
| `new` | Новая анкета | рабочий конвейер |
| `questionnaire_pending` | Анкета не заполнена | рабочий конвейер |
| `review` | На согласовании | рабочий конвейер |
| `check` | Проверяется | рабочий конвейер |
| `revision` | На доработке | рабочий конвейер |
| `approved` | Одобрен | решение (только админ) |
| `approved_with_limit` | Одобрен с лимитом (лимит — на один договор) | решение (только админ) |
| `rejected` | Отклонён (обязательна причина; опция «не приглашать 12 месяцев») | решение (только админ) |

**Оси проверок СБ (`sb_status`) и юриста (`legal_status`)** — общая шкала: `not_required` «Не требуется», `pending` «Ожидает», `in_review` «На проверке», `passed` «Пройдена», `passed_with_remarks` «Пройдена с замечаниями», `failed` «Не пройдена».

**Финансовый блок (`finance_status`)**: `pending` «Ожидает», `approved` «Одобрено», `approved_with_limit` «Одобрено с лимитом», `rejected` «Отказ».

**Сводный статус (`overall_status`)** — вычисляется на каждое изменение оси, вручную не правится: `questionnaire_missing` «Анкета не заполнена», `draft` «Черновик анкеты», `in_review` «На проверке», `accredited` «Аккредитован», `rejected` «Отклонён». Правило (порядок важен):
1. любая ось `failed` / `rejected` → `rejected`;
2. все три оси в «хороших» значениях (`passed`, `passed_with_remarks`, `not_required`; у финансов `approved`, `approved_with_limit`) → `accredited`;
3. контрагент заведён вручную и анкета не подана → `questionnaire_missing`;
4. анкета не подана и все оси `pending` → `draft`;
5. иначе → `in_review`.

**Пул (`pool`)**: `main` «Основной пул», `negotiation` «Для переторгов», `none` «Вне пулов». Автоправило: если СБ или юрист поставили `passed_with_remarks` и пул не выставлялся вручную — контрагент переводится в `negotiation`.

**Автопроверки (`checks`)** — ключи `inn` (ЕГРЮЛ/ЕГРИП + контрольная сумма ИНН), `sro` (реестр СРО), `arbitrage` (картотека арбитражных дел), `bankruptcy` (ЕФРСБ), `blacklist` (реестр недобросовестных поставщиков); значения `ok`, `warn`, `fail`, `pend` (не проверялось), `unavailable` (источник не ответил, нужна ручная проверка по ссылке).

**Обязательные документы анкеты** (`doc_type`): `charter` «Устав», `inn_extract` «Выписка ЕГРЮЛ / свидетельство ИНН», `authority` «Документ, подтверждающий полномочия подписанта». Необязательные: `sro`, `license_mchs`, `license_minculture`, `iso`, `other`; принимаются и прежние ключи `rd_url`, `tz_file`, `smr_contract_file`, `surety_file`.

## Объект `SupplierItem`

Единая форма контрагента в реестре, карточке и ответах правки (ключи — как ждёт фронт, смешанный регистр сохранён):

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | id |
| `name` / `short` | string | полное / краткое наименование |
| `inn`, `kpp`, `ogrn` | string | реквизиты (`""`, если нет) |
| `status` | string | квалификация (см. справочник) |
| `category` | string | устаревшее поле, интерфейс не использует |
| `workTypes` / `workTypeIds` | string[] / integer[] | виды работ из справочника (специализация) |
| `submitted` | string | дата создания карточки `ДД.ММ.ГГГГ` (МСК) |
| `region`, `website`, `tax` | string | регион, сайт, система налогообложения |
| `ceo`, `phone`, `email` | string | ФИО руководителя, телефон (E.164), e-mail организации — **хранятся зашифрованными** |
| `undecryptable` | string[] | какие из `ceo`/`phone`/`email` не расшифровались (значение пустое, интерфейс просит ввести заново) |
| `requisiteIssues` | `{field, message}[]` | реквизиты, не проходящие проверку сейчас (старые записи) — предупреждение с кнопкой «Исправить» |
| `revenue` | number[] | устаревший массив выручки трёх лет «старый → новый»; `[]` без права «Цены» |
| `revenues` | `{id, year, amount}[]` | выручка по годам; `[]` без права «Цены» и в реестре |
| `finance_masked` | boolean | финансы скрыты от смотрящего |
| `finance_status`, `finance_status_comment`, `finance_status_changed_at` | string / string\|null / ISO\|null | ось финансов |
| `finance_limit_amount` | number \| null | лимит на один договор |
| `sb_status`, `sb_status_comment`, `sb_status_changed_at` | | ось СБ |
| `legal_status`, `legal_status_comment`, `legal_status_changed_at` | | ось юриста |
| `overall_status` | string | сводный статус |
| `pool`, `pool_comment`, `pool_changed_at`, `pool_is_manual` | | пул; `pool_is_manual` — выставлен вручную (автоправило больше не трогает) |
| `noInviteUntil` | string \| null | `ГГГГ-ММ-ДД` — до какой даты не приглашать (отказ с запретом на 12 месяцев) |
| `accredited_at` / `accreditation_revoked_at` | ISO \| null | период аккредитации: дата выдачи и дата снятия |
| `questionnaire_submitted_at` | ISO \| null | когда анкета подана на аккредитацию |
| `founded_at` | string \| null | дата создания организации `ГГГГ-ММ-ДД` |
| `staff`, `experience`, `sro`, `license_mchs`, `license_minculture`, `comment` | | численность, описание опыта, допуск СРО, лицензии, комментарий сотрудника |
| `userTags` | string[] | пользовательские теги |
| `manual` | boolean | заведён вручную |
| `checks` | object | `{inn, sro, arbitrage, bankruptcy, blacklist}` → результат (в реестре все `pend`) |
| `checkDetails` | object | детали последней проверки по ключу (только карточка) |
| `docs` | `{id, name, doc_type, date, has_file}[]` | документы анкеты; `has_file=false` — запись без файла (старые данные) |
| `pdnConsent` | `{given, at, documentRef}` | согласие на обработку ПДн основного контакта и редакция текста согласия |
| `registeredViaStaleLink` | boolean | анкета подана по ссылке, которую позже перевыпустили или отозвали |
| `questionnaireAuthor` | `{full_name, email, job_title}` \| null | кто заполнил анкету (основной контакт) |

Только в `GET /suppliers/{id}` добавляются: `inviteDocs` (файлы, загруженные контрагентом через формы КП: `{id, name, doc_type, label, date, has_file}`), `statusEvents` (лента статусных событий, см. ниже), `workExperience` (договоры опыта, см. `GET …/work-experience`).

Событие `statusEvents[]`: `{id, kind, old_status, new_status, comment, limit_amount, user_id, user_name, created_at}`; `kind` — `sb_status`, `legal_status`, `finance_status`, `pool`, `accreditation_granted`, `accreditation_revoked`, `accreditation_date`, `questionnaire_submitted`, `checks_completed`. Новые сверху.

---

## Реестр

### GET /suppliers

**Назначение.** Реестр контрагентов (не удалённых) с серверными фильтрами, поиском и пагинацией. Финансы в реестре не отдаются никогда.

**Доступ.** Сотрудник.

**Параметры запроса:**

| Параметр | Тип | Смысл / ограничения |
|---|---|---|
| `status` | string | квалификация; несколько через запятую |
| `work_type_id` | integer | вид работ (по связке контрагент ↔ вид работ) |
| `sb_status`, `legal_status`, `finance_status`, `overall_status`, `pool` | string | статусные фильтры, каждый — несколько значений через запятую; пустое значение в БД считается значением по умолчанию (`pending`, `draft`, `none`) |
| `region` | string | регион, подстрока без учёта регистра; несколько через запятую |
| `q` | string | поиск по наименованию, краткому названию, ИНН, ОГРН, региону, ФИО руководителя и по опыту работ (заказчик, адрес объекта, вид объекта, состав работ). E-mail в поиске не участвует |
| `revenue_year` | integer | год выручки для фильтра (по умолчанию — прошлый, «опорный») |
| `revenue_min` / `revenue_max` | number | выручка за год не меньше / не больше; контрагент без суммы за этот год в срез не попадает |
| `operating_period` | string | срок работы (по `founded_at`, полных лет): `lt_1`, `1_to_3`, `3_to_5`, `5_to_10`, `gte_10`, `empty` (дата не указана); через запятую |
| `limit` / `offset` | integer | overview §8 |

Сортировка — по `id`. Поиск по ФИО руководителя выполняется после расшифровки (поле зашифровано), поэтому `total` считается **после** всех фильтров, затем берётся страница.

**Ответ `200`:** конверт `{items: SupplierItem[], total, limit, offset}`; в элементах `revenues: []`, `finance_masked: true`, `checks` — все `pend`, `checkDetails: {}`.

**Ошибки.** 401/403 — общие; 422 — нечисловые `work_type_id`/`revenue_*`, `limit` вне 1–200.

**Пример.** `GET /suppliers?overall_status=accredited&pool=main&q=фасад&limit=1` →

```json
{"items": [{"id": 201, "name": "ООО «Альфа Фасад»", "short": "Альфа Фасад", "inn": "7700000000",
  "status": "approved", "workTypes": ["Устройство фасадов"], "workTypeIds": [13], "region": "Москва",
  "email": "user@example.com", "overall_status": "accredited", "sb_status": "passed", "legal_status": "passed",
  "finance_status": "approved", "pool": "main", "revenues": [], "finance_masked": true,
  "checks": {"inn": "pend", "sro": "pend", "arbitrage": "pend", "bankruptcy": "pend", "blacklist": "pend"},
  "docs": [{"id": 195, "name": "Устав.pdf", "doc_type": "charter", "date": "12.11.2025", "has_file": true}],
  "pdnConsent": {"given": true, "at": "2025-11-12T13:05:00+00:00", "documentRef": "pdn-consent-v1.0"},
  "registeredViaStaleLink": false, "questionnaire_submitted_at": "2025-11-12T13:05:00+00:00"}],
 "total": 10, "limit": 1, "offset": 0}
```

**Побочные эффекты.** Нет.

---

### GET /suppliers/new-count

**Назначение.** Счётчик новых контрагентов (`status = new`, не удалённых) для бейджа в боковом меню.

**Доступ.** Сотрудник.

**Параметры.** Нет.

**Ответ `200`:** `{"count": integer}`.

**Ошибки.** 401/403.

**Пример.** → `{"count": 2}`

**Побочные эффекты.** Нет.

---

### GET /suppliers/status-facets

**Назначение.** Значения статусных фильтров реестра со счётчиками. Значения берутся из справочника **целиком** (фильтр «Отклонён» виден, даже когда отклонённых 0); счётчики — по живому реестру.

**Доступ.** Сотрудник.

**Параметры.** Нет.

**Ответ `200`:** `{"facets": {<фасет>: [{value, label, count}]}}`, фасеты: `overall_status`, `sb_status`, `legal_status`, `finance_status`, `pool` (порядок значений — как в справочниках выше; значение из БД, которого нет в справочнике, добавляется в конец с `label = value`), `region` (различные непустые регионы живого реестра по алфавиту, `label = value`).

**Ошибки.** 401/403.

**Пример.** → `{"facets": {"overall_status": [{"value": "questionnaire_missing", "label": "Анкета не заполнена", "count": 0}, {"value": "accredited", "label": "Аккредитован", "count": 10}], "pool": [{"value": "main", "label": "Основной пул", "count": 7}], "region": [{"value": "Москва", "label": "Москва", "count": 6}]}}` (сокращено).

**Побочные эффекты.** Нет.

---

### GET /suppliers/inn-lookup

**Назначение.** Для формы ручного заведения: занят ли ИНН в реестре и что о нём знает ЕГРЮЛ (кнопка «Заполнить по ИНН»). Ничего не записывает. Форма проверяет дубль при уходе из поля (`registry=false`, быстро), реестр — только по кнопке.

**Доступ.** Сотрудник.

**Параметры запроса:**

| Параметр | Тип | Обяз. | Смысл |
|---|---|---|---|
| `inn` | string | да | ИНН (нормализуется: 10 или 12 цифр, контрольная сумма) |
| `registry` | boolean | нет, `true` | спрашивать ЕГРЮЛ (медленно, таймаут 8 с, соединение 4 с) |

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `inn` | string | нормализованный ИНН |
| `duplicate` | `{id, name, deleted}` \| null | карточка, уже занявшая ИНН (`deleted=true` — она в корзине) |
| `registry` | object \| null | `null` при `registry=false`; иначе `{found, summary, liquidated, name, short_name, ogrn, kpp, address, source_url}`; при недоступности ЕГРЮЛ — `found: false` и пояснение в `summary` |

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | ИНН не прошёл проверку | `{"error": "Проверьте заполнение формы", "fieldErrors": {"inn": "Проверьте ИНН: контрольная цифра не сходится, скорее всего опечатка"}}` |
| 422 | нет `inn` | |

**Пример.** `GET /suppliers/inn-lookup?inn=7700000000&registry=false` → `{"inn": "7700000000", "duplicate": {"id": 201, "name": "ООО «Альфа Фасад»", "deleted": false}, "registry": null}`.

**Побочные эффекты.** Исходящий запрос к ЕГРЮЛ (при `registry=true`).

---

### POST /suppliers

**Назначение.** Ручное заведение контрагента сотрудником («Добавить вручную»), минуя анкету. Реквизиты проверяются по тем же правилам, что в анкете; все ошибки формы возвращаются разом.

**Доступ.** Сотрудник; статус-решение (`approved`, `approved_with_limit`, `rejected`) — только администратор.

**Тело запроса (JSON-объект):**

| Поле | Тип | Обяз. | Ограничения |
|---|---|---|---|
| `name` | string | да | «Укажите название компании» |
| `short` (или `short_name`) | string | нет | по умолчанию = `name` |
| `inn` | string | да | ИНН 10/12 цифр с контрольной суммой; уникален в реестре (включая корзину) |
| `kpp` | string | нет | 9 знаков (5–6 позиции — цифры или A–Z) |
| `ogrn` | string | нет | 13 (ОГРН) / 15 (ОГРНИП) цифр с контрольной цифрой; согласован с ИНН |
| `legal_address` | string | нет | адрес: город, улица, дом |
| `contact` | `{full_name, phone, email}` | нет | если начат — ФИО обязательно и нужен телефон **или** e-mail; ошибки по ключам `contact.<поле>` |
| `status` | string | нет | квалификация, по умолчанию `questionnaire_pending` |
| `workTypeIds` (или `workTypes`) | (integer\|string)[] | нет | id или точные названия из справочника видов работ |
| `qualification_comment` | string | нет | комментарий решения (по умолчанию «Заведён вручную»), если статус — решение |

Оси СБ и юриста создаются в `pending` (их решают владельцы), финансы — `pending`; если администратор сразу ставит статус-решение — СБ и юрист `not_required`. Сводный статус считается сразу (обычно `questionnaire_missing`). Создаются пустые строки выручки за три последних завершённых года; переданный контакт становится основным (согласия на ПДн у него нет).

**Ответ `201`:** `{"item": SupplierItem}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | ошибки полей | `{"error": "Проверьте заполнение формы", "fieldErrors": {"name": "Укажите название компании", "inn": "…", "contact.phone": "Укажите телефон или email контакта"}}` |
| 400 | неизвестный статус | «Некорректный статус квалификации» |
| 400 | вид работ не найден | «Вид работ не найден в справочнике: <значение>» |
| 403 | статус-решение не администратором | «Решение по квалификации выставляют администратор и руководитель тендерного отдела» |
| 409 | ИНН занят | `{"error": "Контрагент с таким ИНН уже есть: <название>[ (в корзине)]", "fieldErrors": {"inn": "…"}, "duplicate": {"id", "name", "deleted"}}`; при гонке — «Контрагент с таким ИНН уже есть в реестре» |

**Пример.** `{"name": "ООО «Тест Строй»", "inn": "7700000000", "contact": {"full_name": "Иванов Иван", "email": "user@example.com"}, "workTypeIds": [13]}` → `201 {"item": {"id": 215, "name": "ООО «Тест Строй»", "status": "questionnaire_pending", "overall_status": "questionnaire_missing", "manual": true, …}}`.

**Побочные эффекты.** Журнал ИБ `supplier_registered`; запись центра уведомлений всем сотрудникам «Новый контрагент добавлен вручную» (`kind: tender`); фоновое задание `supplier_notification` — письмо держателям права `suppliers.manage` «Новый контрагент: <название>»; при статусе-решении — строка истории квалификации.

---

### GET /suppliers/queue/security

**Назначение.** Очередь проверки СБ: контрагенты с `sb_status = pending`, не удалённые, по дате подачи анкеты (без даты — в конце), затем по id.

**Доступ.** Сотрудник; юристу — 403.

**Параметры запроса:** `limit`, `offset` (overview §8).

**Ответ `200`:** конверт; элемент `{id, legal_name, questionnaire_submitted_at (ISO|null), waiting_days (integer|null — полных суток с подачи)}`.

**Ошибки.** 403 «Очередь СБ доступна службе безопасности» (роль `lawyer`).

**Пример.** → `{"items": [{"id": 214, "legal_name": "ООО «Тест Строй»", "questionnaire_submitted_at": "2026-10-10T09:00:00+00:00", "waiting_days": 5}], "total": 1, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### GET /suppliers/queue/lawyer

**Назначение.** Очередь юридической проверки: `legal_status = pending`; формат и сортировка — как у очереди СБ.

**Доступ.** Сотрудник; роли `security` — 403.

**Параметры запроса:** `limit`, `offset`.

**Ответ `200`:** как у `GET /suppliers/queue/security`.

**Ошибки.** 403 «Очередь юриста доступна юристу» (роль `security`).

**Пример.** → `{"items": [], "total": 0, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

## Карточка

### GET /suppliers/{supplier_id}

**Назначение.** Карточка контрагента: `SupplierItem` с последними автопроверками, документами анкеты и форм КП, лентой статусных событий и опытом работ.

**Доступ.** Сотрудник. Выручка (`revenues`, `revenue`) — только с правом «Цены», иначе `finance_masked: true`, `revenues: []`, `revenue: []` (лимит `finance_limit_amount` и финансовый статус отдаются всем сотрудникам).

**Параметры пути:** `supplier_id` — integer.

**Ответ `200`:** `{"item": SupplierItem + inviteDocs + statusEvents + workExperience}`; `checks`/`checkDetails` — последние результаты по каждому ключу (как в `GET …/checks`).

**Ошибки.** 404 «Контрагент не найден».

**Пример.** → `{"item": {"id": 201, "name": "ООО «Альфа Фасад»", "finance_masked": false, "revenues": [{"id": 237, "year": 2025, "amount": 1612800.0}], "checks": {"inn": "ok", "sro": "warn", "arbitrage": "ok", "bankruptcy": "ok", "blacklist": "ok"}, "statusEvents": [{"id": 514, "kind": "accreditation_granted", "old_status": null, "new_status": "accredited", "comment": "Аккредитация выдана 18.11.2025", "user_name": "Админова Елена", "created_at": "2025-11-18T14:30:00+00:00"}], "inviteDocs": [], "workExperience": [], …}}`

**Побочные эффекты.** При праве «Цены»: досоздаются недостающие строки выручки (три последних завершённых года и все пропуски от самого старого внесённого года), журнал ИБ `finance_viewed` (`entity_type=supplier`).

---

### PATCH /suppliers/{supplier_id}

**Назначение.** Правка карточки: реквизиты и контакты организации, виды работ, теги, решение по квалификации, дата аккредитации. Форма присылает карточку целиком; проверяются только **изменившиеся** реквизиты (старые записи с некорректным ИНН не блокируют правку названия).

**Доступ.** Сотрудник; статус-решение — администратор; `revenue` — право «Цены»; `accredited_at` — тендерный специалист или администратор.

**Тело запроса (все поля необязательны):**

| Поле | Тип | Правило |
|---|---|---|
| `name`, `short`, `region`, `tax`, `staff`, `experience`, `sro`, `license_mchs`, `license_minculture`, `comment` | string / integer | записываются как есть |
| `inn` | string | обязателен, если передан и меняется; контрольная сумма; уникальность (409 как у `POST /suppliers`) |
| `kpp`, `ogrn` | string | проверка формата; пара ИНН–ОГРН проверяется, если меняется одно из них |
| `ceo` | string | ФИО (минимум фамилия и имя; буквы, пробел, дефис, точка) |
| `phone`, `email`, `website`, `founded_at` | string | нормализаторы overview §7; `founded_at` — `ДД.ММ.ГГГГ`/ISO, не в будущем, год ≥ 1900. Пустое значение поверх нерасшифрованного поля не записывается |
| `workTypeIds` / `workTypes` | array | полностью переписывают виды работ; id или названия из справочника |
| `userTags` | string[] | теги |
| `status` | string | квалификация (справочник); решение — только админ |
| `qualification_comment` | string | обязателен при переходе в `rejected` |
| `finance_limit_amount` | number | обязателен для `approved_with_limit`, ≥ 0 |
| `no_invite_12m` | boolean | при `rejected` — «не приглашать» на 365 дней |
| `revenue` | number[] | устаревший массив выручки (право «Цены») |
| `accredited_at` | string \| null | см. `PATCH …/accredited-at` |
| `overall_status`, `sb_status`, `legal_status`, `finance_status`, `pool` | — | **запрещены** (400) |

Поле `docs` не принимается: документы появляются только реальной загрузкой файла.

**Ответ `200`:** `{"item": SupplierItem}` (с документами, видами работ, согласием ПДн).

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | передан `overall_status` | «Общий статус вычисляется по частным проверкам и вручную не редактируется» |
| 400 | передана ось/пул | «Поле sb_status меняется через PATCH /suppliers/{id}/sb-status — с комментарием и историей» (аналогично `legal_status`, `finance_status`, `pool`) |
| 400 | ошибки реквизитов | `{"error": "Проверьте заполнение формы", "fieldErrors": {...}}` |
| 400 | квалификация | «Некорректный статус квалификации»; «Укажите причину отказа — она уходит в историю решений»; «Укажите сумму лимита на один договор»; «Некорректная сумма лимита»; «Лимит не может быть отрицательным» |
| 400 | вид работ | «Вид работ не найден в справочнике: <значение>» |
| 403 | решение не админом | «Решение по квалификации выставляют администратор и руководитель тендерного отдела» |
| 403 | `revenue` без права | «Нет права редактировать финансовый блок» |
| 409 | ИНН занят | как у `POST /suppliers` |

**Пример.** `{"name": "ООО «Альфа Фасад»", "userTags": ["витражи"], "status": "approved_with_limit", "finance_limit_amount": 50000000}` → `{"item": {"id": 201, "status": "approved_with_limit", "finance_limit_amount": 50000000.0, …}}`.

**Побочные эффекты.** Журнал ИБ: `supplier_requisites_changed` (смена ИНН/КПП/ОГРН, `{поле: {from, to}}`), `supplier_work_types_changed` (`added`, `removed`), `supplier_qualification_decided` (prev/new/limit); строка append-only истории квалификации при изменении статуса/лимита/запрета или с комментарием; событие `accreditation_date` при смене даты.

---

### GET /suppliers/{supplier_id}/checks

**Назначение.** Последний результат каждой автопроверки по реестрам.

**Доступ.** Сотрудник.

**Ответ `200`:** `{"checks": {ключ: результат}, "details": {ключ: {result, details, checked_at}}}`. `details` — `summary` (текст для человека), `source_url` (ссылка для ручной проверки) и данные источника (у `inn`: `inn`, `checksum`, `name`, `short_name`, `ogrn`, `kpp`, `address`, `raw_status`; у `sro`: `manual_required`). Ключ без проверок — `pend` и нет записи в `details`.

**Ошибки.** 404 «Контрагент не найден».

**Пример.** → `{"checks": {"inn": "ok", "sro": "warn", "arbitrage": "ok", "bankruptcy": "ok", "blacklist": "ok"}, "details": {"inn": {"result": "ok", "details": {"summary": "Найдено в ЕГРЮЛ/ЕГРИП: Альфа Фасад", "source_url": "https://egrul.nalog.ru/…", "checksum": true}, "checked_at": "2025-11-13T06:12:00+00:00"}}}`

**Побочные эффекты.** Нет.

---

### POST /suppliers/{supplier_id}/checks/run

**Назначение.** Запустить пять автопроверок параллельно и сохранить результаты. Источники нестабильны: на каждый запрос — таймаут 8 с (соединение 4 с); при сбое, CAPTCHA или блокировке сайта результат `unavailable` с пояснением и ссылкой для ручной проверки, ручка не падает. Особые правила проверки ИНН: не сошлась контрольная сумма — `fail` без похода в реестр; запись реестра без наименования — `warn` (не подтверждение); признак прекращения деятельности — `fail`.

**Доступ.** Сотрудник.

**Тело запроса.** Нет.

**Ответ `200`:** как у `GET …/checks`.

**Ошибки.** 400 «У контрагента не указан ИНН»; 404.

**Пример.** `POST /suppliers/201/checks/run` → `{"checks": {"inn": "ok", "sro": "warn", "arbitrage": "unavailable", "bankruptcy": "unavailable", "blacklist": "ok"}, "details": {…}}`

**Побочные эффекты.** По строке результатов на каждую проверку (история хранится); журнал ИБ `supplier_checks_run` (результаты по ключам). Время ответа — до ~10 с.

---

### GET /suppliers/{supplier_id}/activity

**Назначение.** Агрегированная лента контрагента у сотрудника: события по всем тендерам, куда он приглашён, плюс статусные события и история квалификации.

**Доступ.** Сотрудник.

**Параметры запроса:**

| Параметр | Тип | Смысл |
|---|---|---|
| `limit` | integer | 1–200, по умолчанию 50 |
| `before_id` | integer | курсор: id последнего показанного события |
| `scope` | `supplier` \| `tender` | `supplier` (по умолчанию) — события, касающиеся этого контрагента; `tender` — всё в его тендерах, включая чужие подачи |
| `tender_id` | integer | только этот тендер |
| `kind` | string, повторяемый | группы: `quote` (подача, доработка, смена статуса КП, решение по корректировке), `round` (раунды, версии ФКП), `message` (чат), `tender` (структура, документы, статус, ответственный, реквизиты), `participant` (приглашения, отзыв, восстановление доступа); неизвестные группы игнорируются |
| `q` | string | ≤ 100 символов; поиск по тексту события и автору |
| `since` / `until` | date `ГГГГ-ММ-ДД` | период, границы включительно, сутки по МСК |

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `items[]` | object | `{id, action, entity_type, entity_id, details, actor, actor_kind (staff/supplier/system), subject_supplier_id, created_at, tender_id, tender_number, tender_title, supplier_visible}`; `supplier_visible` — увидит ли событие кабинет контрагента; `details` — полные (без обрезки) |
| `next_before_id` | integer \| null | курсор следующей страницы (`null`, если страница неполная) |
| `statusEvents` | array | статусные события (как в карточке) |
| `qualificationHistory` | array | см. `GET …/qualification-history` |
| `tenders` | `{id, number, title}[]` | тендеры контрагента для фильтра |

Сортировка — новые сверху.

**Ошибки.** 404; 422 — `scope` не из списка, `limit` вне 1–200, `q` > 100.

**Пример.** → `{"items": [{"id": 392, "action": "correction_decided", "actor": "Админова Елена", "actor_kind": "staff", "details": {"kind": "volume", "from_quantity": 542.8, "to_quantity": 618.792, "review_status": "accepted"}, "tender_id": 100, "tender_number": "T-2026-100", "tender_title": "Витражи, дворовой фасад", "supplier_visible": true, "created_at": "2026-09-28T08:03:42Z"}], "next_before_id": 392, "statusEvents": [], "qualificationHistory": [], "tenders": [{"id": 100, "number": "T-2026-100", "title": "Витражи, дворовой фасад"}]}`

**Побочные эффекты.** Нет.

---

### GET /suppliers/{supplier_id}/participation

**Назначение.** Блок «Участие в тендерах»: строка на каждое приглашение и сводка.

**Доступ.** Сотрудник.

**Ответ `200`:** `{"items": [...], "summary": {"tenders", "wins", "closed"}}`. Элемент: `tender_id`, `tender_number`, `title`, `tender_status`, `invited_at`, `closed_at`, `role` (`winner` / `participant`), `outcome` (`won` — победитель; `lost` — закрыт, победил другой; `no_bid` — закрыт, КП не подано; `in_progress`; `declined` — отказался; `cancelled` — тендер отменён), `amount` (сумма заявки или `null`). `closed` в сводке — число строк с `won`/`lost`/`no_bid` (база для доли побед).

**Ошибки.** 404.

**Пример.** → `{"items": [{"tender_id": 109, "tender_number": "T-2026-109", "title": "Фасад, МКД-3", "tender_status": "closed", "invited_at": "2026-08-31T06:30:00Z", "closed_at": "2026-09-22T12:00:00Z", "role": "participant", "outcome": "lost", "amount": 216808450.63}], "summary": {"tenders": 4, "wins": 0, "closed": 1}}`

**Побочные эффекты.** Нет.

---

### GET /suppliers/{supplier_id}/qualification-history

**Назначение.** История решений по квалификации и изменений осей (append-only, задним числом не правится — запрет на уровне БД).

**Доступ.** Сотрудник.

**Ответ `200`:** `{"items": [{id, prev_status, new_status, limit_amount, comment, no_invite_until, decided_by, decided_by_name, decided_at}]}`, новые сверху. Смены осей записываются с префиксом оси: `sb_status:pending` → `sb_status:passed`.

**Ошибки.** 404.

**Пример.** → `{"items": [{"id": 378, "prev_status": "review", "new_status": "approved", "limit_amount": null, "comment": "Решение тендерного комитета", "no_invite_until": null, "decided_by": 1, "decided_by_name": "Админова Елена", "decided_at": "2025-11-18T14:30:00+00:00"}]}`

**Побочные эффекты.** Нет.

---

## Статусная модель

### PATCH /suppliers/{supplier_id}/sb-status

**Назначение.** Решение службы безопасности по своей оси. Оси независимы: переход одной не проверяет и не блокирует другие.

**Доступ.** Роль `security` или администратор.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения |
|---|---|---|---|
| `status` | string | да | шкала СБ/юриста |
| `comment` | string | да | непустой после обрезки |

**Ответ `200`:** `{"item": SupplierItem}` (без документов и видов работ — клиент перечитывает карточку).

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | значение не из шкалы | «Некорректный статус проверки» |
| 400 | пустой комментарий | «Комментарий обязателен» |
| 403 | не та роль | «Проверка СБ: нет права менять этот статус» |
| 404 | | «Контрагент не найден» |

**Пример.** `{"status": "passed_with_remarks", "comment": "Есть судебные споры, некритично"}` → `{"item": {"id": 205, "sb_status": "passed_with_remarks", "overall_status": "in_review", "pool": "negotiation", …}}`

**Побочные эффекты.** Запись оси (значение, комментарий, автор, время); статусное событие `sb_status` и строка истории квалификации; пересчёт `overall_status`; при переходе в/из `accredited` — `accredited_at` = сейчас и событие `accreditation_granted` (повторное одобрение открывает новый период) или `accreditation_revoked_at` и событие `accreditation_revoked`; автоперевод в пул `negotiation` (событие `pool` с комментарием «Автоматически: проверка пройдена с замечаниями»). SSE `type: supplier` (`supplier_id`) всем сотрудникам. После коммита — уведомление «Проверка СБ закрыта» (`<название>: <статус>. <комментарий>`) по каталогу (носители роли-адресата + админы, письмо по галочке), если значение терминальное (`passed`, `passed_with_remarks`, `failed`, `not_required`); если закрыты все три оси — однократно событие `checks_completed` и уведомление «Контрагент проверен: СБ, юрист и финансы».

---

### PATCH /suppliers/{supplier_id}/legal-status

**Назначение.** Решение юриста по своей оси; правила, ответ и побочные эффекты — как у `sb-status`.

**Доступ.** Роль `lawyer` или администратор.

**Тело запроса:** `status` (шкала СБ/юриста, обяз.), `comment` (обяз.).

**Ответ `200`:** `{"item": SupplierItem}`.

**Ошибки.** 400 «Некорректный статус проверки» / «Комментарий обязателен»; 403 «Юридическая проверка: нет права менять этот статус»; 404.

**Пример.** `{"status": "passed", "comment": "Учредительные документы проверены"}` → `{"item": {"legal_status": "passed", …}}`

**Побочные эффекты.** Как у `sb-status`; уведомление «Юридическая проверка закрыта».

---

### PATCH /suppliers/{supplier_id}/finance-status

**Назначение.** Третья ось — финансовый блок; у «одобрено с лимитом» обязательна сумма лимита на один договор.

**Доступ.** Право «Цены» или администратор.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения |
|---|---|---|---|
| `status` | string | да | `pending`, `approved`, `approved_with_limit`, `rejected` |
| `comment` | string | да | непустой |
| `limit_amount` (или `finance_limit_amount`) | number | при `approved_with_limit` | > 0; для остальных статусов лимит сбрасывается |

**Ответ `200`:** `{"item": SupplierItem}` с финансами.

**Ошибки.** 400 «Некорректный статус проверки», «Комментарий обязателен», «Укажите сумму лимита на один договор», «Некорректная сумма лимита», «Лимит не может быть отрицательным или нулевым»; 403 «Финансовый статус меняет финансовый блок и администратор»; 404.

**Пример.** `{"status": "approved_with_limit", "comment": "Выручка ниже порога", "limit_amount": 30000000}` → `{"item": {"finance_status": "approved_with_limit", "finance_limit_amount": 30000000.0, …}}`

**Побочные эффекты.** Как у `sb-status` (без отдельного уведомления о закрытии оси; «проверен всеми» — при закрытии последней оси); лимит пишется в историю квалификации.

---

### PATCH /suppliers/{supplier_id}/pool

**Назначение.** Ручной перевод между пулами с обязательным комментарием. Общий статус и проверки не меняются; после ручного решения автоправило «с замечаниями → для переторгов» пул больше не трогает.

**Доступ.** Тендерный специалист или администратор.

**Тело запроса:** `pool` — `main` / `negotiation` / `none` (обяз.); `comment` — string (обяз.).

**Ответ `200`:** `{"item": SupplierItem}`.

**Ошибки.** 400 «Некорректный пул», «Комментарий обязателен»; 403 «Пул меняют тендерный специалист и администратор»; 404.

**Пример.** `{"pool": "main", "comment": "Подтвердили качество на объекте"}` → `{"item": {"pool": "main", "pool_is_manual": true, …}}`

**Побочные эффекты.** `pool_is_manual = true`, автор и время; статусное событие `pool`.

---

### PATCH /suppliers/{supplier_id}/accredited-at

**Назначение.** Ручное выставление или исправление даты аккредитации. Сводный статус от даты не меняется.

**Доступ.** Тендерный специалист или администратор.

**Тело запроса:**

| Поле | Тип | Обяз. | Ограничения |
|---|---|---|---|
| `accredited_at` (или `date`) | string \| null | да | `ДД.ММ.ГГГГ` или `ГГГГ-ММ-ДД[Thh:mm:ss[Z]]`; не в будущем; `null`/пусто — очистить (нельзя у аккредитованного) |

**Ответ `200`:** `{"item": SupplierItem}` (с документами и видами работ; финансы — по праву «Цены»).

**Ошибки.** 400 «Неверный формат даты. Используйте ДД.ММ.ГГГГ или ГГГГ-ММ-ДД», «Некорректная дата: <значение>», «Дата аккредитации не может быть в будущем», «Нельзя удалить дату аккредитации у аккредитованного контрагента»; 403 «Нет права редактировать карточку контрагента»; 404.

**Пример.** `{"accredited_at": "18.11.2025"}` → `{"item": {"accredited_at": "2025-11-18T00:00:00+00:00", …}}`

**Побочные эффекты.** Если дата изменилась — статусное событие `accreditation_date` (старое и новое значение `ДД.ММ.ГГГГ`); SSE `type: supplier`.

---

### PATCH /suppliers/{supplier_id}/accreditation-date

**Назначение.** Синоним пути `PATCH /suppliers/{supplier_id}/accredited-at` (тот же обработчик, поддерживается для старых клиентов).

**Доступ.** Тендерный специалист или администратор.

**Тело запроса:** `accredited_at` или `date` — как у основного пути.

**Ответ `200`:** `{"item": SupplierItem}`.

**Ошибки.** Как у `accredited-at`.

**Пример.** `{"date": "2025-11-18"}` → `{"item": {"accredited_at": "2025-11-18T00:00:00+00:00", …}}`

**Побочные эффекты.** Как у `accredited-at`.

---

### POST /suppliers/{supplier_id}/questionnaire/submit

**Назначение.** Отметить анкету поданной у контрагента, заведённого вручную (у анкеты по ссылке это делает `POST /suppliers/register/{id}/complete`). Обязательные документы должны быть загружены реальными файлами (сотрудник грузит их через `POST /files/upload` с `entity_type=supplier`). Повторный вызов ничего не меняет и не шлёт второго уведомления.

**Доступ.** Право `suppliers.manage`.

**Тело запроса.** Нет.

**Ответ `200`:** `{"item": SupplierItem}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | нет файла в обязательном слоте | `{"error": "Не хватает обязательных документов", "missing_required_docs": [{"key": "charter", "label": "Устав"}]}` |
| 403 | нет права | общий текст |
| 404 | | «Контрагент не найден» |

**Пример.** `POST /suppliers/215/questionnaire/submit` → `{"item": {"id": 215, "status": "review", "overall_status": "in_review", "questionnaire_submitted_at": "2026-10-15T08:00:00+00:00", …}}`

**Побочные эффекты.** `questionnaire_submitted_at`; оси, которые ещё не трогали, — `pending`; статус `new` → `review`; пересчёт сводного статуса; статусное событие `questionnaire_submitted`; три уведомления по каталогу: «Новая анкета: требуется проверка СБ», «Новая анкета: требуется юридическая проверка», «Новая анкета на аккредитацию» (тело «<название> (ИНН …) — анкета отправлена», ссылка на карточку).

---

## Финансы

### GET /suppliers/{supplier_id}/revenues

**Назначение.** Выручка по годам. Перед чтением досоздаются пустые строки за три последних завершённых года (и пропуски от самого старого внесённого года). Пустая строка текущего (незавершённого) года не показывается.

**Доступ.** Право «Цены».

**Ответ `200`:** `{"items": [{"id", "year", "amount (number|null)"}]}`, годы по убыванию.

**Ошибки.** 403 «Нет права видеть финансовый блок»; 404.

**Пример.** → `{"items": [{"id": 237, "year": 2025, "amount": 1612800.0}, {"id": 236, "year": 2024, "amount": 1426300.0}, {"id": 235, "year": 2023, "amount": null}]}`

**Побочные эффекты.** Возможное создание пустых строк выручки.

---

### PUT /suppliers/{supplier_id}/revenues

**Назначение.** Сохранить суммы выручки по годам (upsert по году).

**Доступ.** Право «Цены».

**Тело запроса:** `{"items": [{"year": integer, "amount": number|null|""}]}` или сам массив. Год — целое, без повторов; пустая сумма — `null`.

**Ответ `200`:** `{"items": [...]}` — как у GET.

**Ошибки.** 400 «Ожидается список {year, amount}», «Некорректный год выручки», «Год <N> указан дважды»; 403 «Нет права редактировать финансовый блок»; 404.

**Пример.** `{"items": [{"year": 2025, "amount": 1612800}]}` → `{"items": [{"id": 237, "year": 2025, "amount": 1612800.0}, …]}`

**Побочные эффекты.** Нет (журнал ИБ не пишется — см. открытые вопросы).

---

## Опыт работ

Строка опыта (`WorkExperienceItem`): `id`, `supplier_id`, `work_type_id`, `work_type_name`, `customer_name`, `object_type`, `object_address`, `work_scope`, `role` (`генподрядчик` / `подрядчик`), `start_date` / `end_date` (`ММ.ГГГГ`, конец может быть `null`), `amount_thousand_rub` (тыс. ₽), `is_completed` (конец указан и не позже текущего месяца), `created_at`, `updated_at`.

Правила полей (общие для анкеты и карточки; ошибки — `{"error": "Проверьте заполнение данных о договоре", "fieldErrors": {...}}`, 400):

| Поле | Обяз. | Правило и текст ошибки |
|---|---|---|
| `work_type_id` | да | «Выберите вид работ»; есть в справочнике («Вид работ не найден в справочнике»); привязан к контрагенту («Вид работ не привязан к контрагенту») |
| `customer_name` | да | ≤ 500 символов; «Укажите наименование организации — заказчика» |
| `object_type` | да | ≤ 255; «Укажите вид объекта строительства» |
| `object_address` | да | ≤ 500; «Укажите адрес объекта» |
| `work_scope` | да | «Укажите наименование работ/услуг/поставок по договору» |
| `role` | да | `генподрядчик`/`подрядчик` (принимаются и `general_contractor`/`subcontractor`); «Роль должна быть «генподрядчик» или «подрядчик»» |
| `start_date` | да | `ММ.ГГГГ`, месяц 01–12, год 1900–2100; «Укажите дату начала в формате ММ.ГГГГ» |
| `end_date` | нет | тот же формат, не раньше начала («Дата окончания не может быть раньше даты начала») |
| `amount_thousand_rub` | да | > 0; «Укажите стоимость по договору», «Стоимость должна быть больше нуля» |

### GET /suppliers/{supplier_id}/work-experience

**Назначение.** Договоры опыта контрагента, сгруппированные по виду работ (сортировка: вид работ, id).

**Доступ.** Сотрудник.

**Ответ `200`:** `{"items": WorkExperienceItem[]}`.

**Ошибки.** 404.

**Пример.** → `{"items": [{"id": 3, "supplier_id": 201, "work_type_id": 13, "work_type_name": "Устройство фасадов", "customer_name": "ООО «Заказчик 5»", "object_type": "Жилой дом 17 эт.", "object_address": "г. Энск, ул. Кирова, 5", "work_scope": "НВФ с керамогранитом, 9 400 м²", "role": "подрядчик", "start_date": "06.2025", "end_date": null, "amount_thousand_rub": 228000.0, "is_completed": false}]}`

**Побочные эффекты.** Нет.

---

### POST /suppliers/{supplier_id}/work-experience

**Назначение.** Добавить договор в опыт контрагента.

**Доступ.** Роль `tender_specialist` или `admin`.

**Тело запроса:** поля строки опыта (таблица выше).

**Ответ `200`:** `{"item": WorkExperienceItem}`.

**Ошибки.** 400 — ошибки полей; 403 «Правка карточки контрагента недоступна для вашей роли»; 404.

**Пример.** `{"work_type_id": 13, "customer_name": "ООО «Заказчик»", "object_type": "Школа", "object_address": "г. Энск, ул. Ленина, 1", "work_scope": "Фасад", "role": "подрядчик", "start_date": "01.2024", "end_date": "12.2024", "amount_thousand_rub": 15000}` → `{"item": {"id": 31, "is_completed": true, …}}`

**Побочные эффекты.** Нет.

---

### PUT /suppliers/{supplier_id}/work-experience/{experience_id}

**Назначение.** Заменить все поля строки опыта (полная валидация).

**Доступ.** `tender_specialist` или `admin`.

**Параметры пути:** `supplier_id`, `experience_id` — integer.

**Тело запроса:** как у POST.

**Ответ `200`:** `{"item": WorkExperienceItem}`.

**Ошибки.** 400; 403 «Правка карточки контрагента недоступна для вашей роли»; 404 «Договор опыта не найден» (нет строки или она другого контрагента).

**Пример.** → `{"item": {"id": 31, "amount_thousand_rub": 16500.0, …}}`

**Побочные эффекты.** Нет.

---

### DELETE /suppliers/{supplier_id}/work-experience/{experience_id}

**Назначение.** Удалить строку опыта.

**Доступ.** `tender_specialist` или `admin`.

**Ответ `200`:** `{"ok": true}`.

**Ошибки.** 403; 404 «Договор опыта не найден».

**Пример.** `DELETE /suppliers/201/work-experience/31` → `{"ok": true}`

**Побочные эффекты.** Нет.

---

## Контакты

Контакт — справочное лицо компании; связи с учётной записью ЛК у контакта нет (доступ в ЛК — вкладка «Пользователи»). ФИО, телефон и e-mail хранятся зашифрованными. Письма по тендерам уходят на e-mail **организации**, а не контакта.

Объект `ContactItem`: `id`, `supplier_id`, `full_name`, `job_title`, `phone`, `email`, `is_primary`, `pdn_consent_given`, `pdn_consent_at`, `pdn_consent_document_ref`, `created_at`, `undecryptable` (какие из `full_name`/`phone`/`email` не расшифровались), `receives_mailings` (e-mail контакта совпадает с e-mail организации), `duplicate_email_suppliers` (`[{id, name}]` — другие контрагенты с тем же e-mail контакта; только сотруднику).

Инвариант: у контрагента с контактами ровно один основной (`is_primary`). Первый контакт становится основным автоматически; при удалении основного основным становится первый оставшийся; снять флаг с единственного основного нельзя (флаг возвращается).

### GET /suppliers/{supplier_id}/contacts

**Назначение.** Контакты контрагента (по id).

**Доступ.** Вошедший: сотрудник — любой контрагент; роль `supplier` — только своя организация (без `duplicate_email_suppliers`).

**Ответ `200`:** `{"items": ContactItem[]}`.

**Ошибки.** 403 «Нет доступа к карточке контрагента» (чужой контрагент); 403 «Доступно только сотрудникам компании» (лид); 404.

**Пример.** → `{"items": [{"id": 144, "supplier_id": 201, "full_name": "Орлова Мария Сергеевна", "job_title": "Руководитель тендерного отдела", "phone": "+79000000000", "email": "user@example.com", "is_primary": true, "pdn_consent_given": true, "pdn_consent_at": "2025-11-12T13:05:00+00:00", "pdn_consent_document_ref": "pdn-consent-v1.0", "undecryptable": [], "receives_mailings": false, "duplicate_email_suppliers": []}]}`

**Побочные эффекты.** Нет.

---

### POST /suppliers/{supplier_id}/contacts

**Назначение.** Добавить контакт.

**Доступ.** Сотрудник.

**Тело запроса:**

| Поле | Тип | Обяз. | Правило |
|---|---|---|---|
| `full_name` | string | да | «Укажите ФИО контакта»; шифротекст не принимается |
| `job_title` | string | нет | |
| `phone` | string | нет | нормализатор телефона |
| `email` | string | нет | нормализатор e-mail |
| `is_primary` | boolean | нет | сделать основным (снимает флаг с остальных) |
| `pdn_consent_given` | boolean | нет | согласие на ПДн; время ставится сервером |
| `pdn_consent_document_ref` | string | нет | редакция документа согласия |

**Ответ `201`:** `{"item": ContactItem}`.

**Ошибки.** 400 `{"error": "Проверьте заполнение формы", "fieldErrors": {...}}`; 404.

**Пример.** `{"full_name": "Петров Пётр", "phone": "+7 900 000-00-00", "email": "user@example.com"}` → `201 {"item": {"id": 160, "is_primary": false, …}}`

**Побочные эффекты.** Нет.

---

### PATCH /suppliers/{supplier_id}/contacts/{contact_id}

**Назначение.** Изменить контакт; меняются только переданные поля. Пустое значение поверх нерасшифрованного поля не записывается.

**Доступ.** Сотрудник.

**Тело запроса:** поля как у POST (все необязательны); `pdn_consent_given=false` сбрасывает время согласия, `true` ставит его, если не было.

**Ответ `200`:** `{"item": ContactItem}`.

**Ошибки.** 400 — поля; 404 «Контакт не найден» (нет или чужой контрагент); 404 «Контрагент не найден».

**Пример.** `{"is_primary": true}` → `{"item": {"id": 160, "is_primary": true, …}}`

**Побочные эффекты.** Нет.

---

### DELETE /suppliers/{supplier_id}/contacts/{contact_id}

**Назначение.** Удалить контакт.

**Доступ.** Сотрудник.

**Ответ `204`** без тела.

**Ошибки.** 404 «Контакт не найден».

**Пример.** `DELETE /suppliers/201/contacts/160` → `204`

**Побочные эффекты.** Если удалён основной — основным становится первый оставшийся.

---

## Пользователи ЛК

Входов в ЛК у контрагента может быть сколько угодно; один пользователь принадлежит не более чем одному контрагенту (уникальность связки по пользователю). Связка «пользователь → контрагент» при отвязке гасится, а не удаляется.

### GET /suppliers/{supplier_id}/cabinet-users

**Назначение.** Вкладка «Пользователи» карточки: кто входит в ЛК.

**Доступ.** `suppliers.manage`.

**Ответ `200`:** `{"items": [{user_id, full_name, email, is_active, mfa_enabled, linked_at}]}` — действующие связки, по id пользователя.

**Ошибки.** 403; 404.

**Пример.** → `{"items": [{"user_id": 3, "full_name": "Орлова Мария Сергеевна", "email": "user@example.com", "is_active": true, "mfa_enabled": false, "linked_at": "2025-11-10T09:40:00+00:00"}]}`

**Побочные эффекты.** Нет.

---

### POST /suppliers/{supplier_id}/cabinet-users

**Назначение.** Привязать **существующего** пользователя к ЛК; его роль приводится к `supplier`.

**Доступ.** `suppliers.manage`.

**Тело запроса:** `{"user_id": integer}` (обяз.).

**Ответ `201`:** `{"items": [...]}` — обновлённый список (как GET).

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | нет/нечисловой `user_id` | «Укажите пользователя» |
| 400 | пользователь привязан к другому контрагенту | «Пользователь уже привязан к контрагенту «<название>» — сначала отвяжите его там» |
| 400 | роль не заведена (ошибка сидов) | «Роль supplier не заведена — сиды ролей не накачены» |
| 404 | | «Пользователь не найден» / «Контрагент не найден» |

**Пример.** `{"user_id": 42}` → `201 {"items": [{"user_id": 3, …}, {"user_id": 42, …}]}`

**Побочные эффекты.** Связка создаётся или реактивируется; журнал ИБ `role_changed` (`new_role: supplier`, `supplier_link`, `supplier_id`), если что-то изменилось.

---

### DELETE /suppliers/{supplier_id}/cabinet-users/{user_id}

**Назначение.** Отвязать пользователя от ЛК. Учётная запись не удаляется, роль не меняется.

**Доступ.** `suppliers.manage`.

**Ответ `204`.**

**Ошибки.** 404 «Пользователь не привязан к этому контрагенту».

**Пример.** `DELETE /suppliers/201/cabinet-users/42` → `204`

**Побочные эффекты.** Связка деактивирована; журнал ИБ `role_changed` (`supplier_link: deactivated`).

---

### POST /suppliers/{supplier_id}/cabinet-login

**Назначение.** Создать новую учётную запись ЛК (роль `supplier`) и привязать её к контрагенту.

**Доступ.** `suppliers.manage`.

**Тело запроса:**

| Поле | Тип | Обяз. | Правило |
|---|---|---|---|
| `email` | string | нет | по умолчанию — e-mail организации; нормализатор e-mail; не занят другой учётной записью |
| `password` | string | нет | политика паролей (overview §7); если не передан — генерируется (14 символов) |

**Ответ `200`:** `{"item": {"user_id", "email", "password"}}` — `password` содержит сгенерированный пароль (показывается один раз) или `null`, если пароль задан в запросе.

**Ошибки.** 400 `fieldErrors` (`email`, `password`); 400 «Email уже занят другой учётной записью»; 404.

**Пример.** `{"email": "user@example.com"}` → `{"item": {"user_id": 57, "email": "user@example.com", "password": "<сгенерирован>"}}`

**Побочные эффекты.** Пользователь (ФИО — руководитель или название организации, без права «Цены») и связка. Письмо не отправляется (см. открытые вопросы).

---

## Корзина

### DELETE /suppliers/{supplier_id}

**Назначение.** Мягкое удаление: контрагент пропадает из реестров и выборок, данные сохраняются.

**Доступ.** `suppliers.manage`.

**Ответ `204`.**

**Ошибки.** 404 «Контрагент не найден» (нет или уже в корзине).

**Пример.** `DELETE /suppliers/215` → `204`

**Побочные эффекты.** `is_deleted = true`, `deleted_at`. ИНН остаётся занятым (повторное заведение — 409 с указанием «в корзине»).

---

### GET /suppliers/trash/list

**Назначение.** Контрагенты в корзине, недавно удалённые сверху.

**Доступ.** `suppliers.manage`.

**Параметры запроса:** `limit`, `offset`.

**Ответ `200`:** конверт `{items: SupplierItem[], total, limit, offset}` (без документов и видов работ).

**Ошибки.** 403.

**Пример.** → `{"items": [{"id": 215, "name": "ООО «Тест Строй»", "inn": "7700000000", …}], "total": 1, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### POST /suppliers/{supplier_id}/restore

**Назначение.** Вернуть контрагента из корзины.

**Доступ.** `suppliers.manage`.

**Ответ `200`:** `{"item": SupplierItem}`.

**Ошибки.** 404 «Контрагент не найден в корзине».

**Пример.** `POST /suppliers/215/restore` → `{"item": {"id": 215, …}}`

**Побочные эффекты.** `is_deleted = false`, `deleted_at = null`.

---

### DELETE /suppliers/{supplier_id}/hard

**Назначение.** Удалить контрагента из корзины насовсем: виды работ, контакты, документы анкеты и сама запись (каскадом — выручка, опыт, статусные события, история квалификации).

**Доступ.** `suppliers.manage`.

**Ответ `204`.**

**Ошибки.** 404 «Сначала переместите в корзину» (нет или не в корзине). Если на контрагента ссылаются приглашения в тендеры, проверки, переписка, связки ЛК или уведомления, удаление в текущей реализации не проходит (ошибка сервера) — требуемое поведение (запрет с 409 или каскад) — см. открытые вопросы.

**Пример.** `DELETE /suppliers/215/hard` → `204`

**Побочные эффекты.** Безвозвратное удаление строк; файлы вложений в хранилище не удаляются (см. открытые вопросы).
