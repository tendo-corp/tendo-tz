# API: тендеры (оглавление, карточка, правка, приглашения)

Раздел API тендеров: создание тендера из разобранной структуры, карточка, правка полей и статуса (закрытие с победителем, отмена), предпросмотр правки сроков с каскадом на раунды, дополнение и правка состава ФКП с версиями и рассылкой, пересборка разделов по файлу ФКП, заметки сотрудников, лента событий, сводка использования пользовательских статусов, приглашения участников и персональные ссылки. Остальные ручки тендеров вынесены в отдельные файлы (оглавление ниже). Общие соглашения — [overview.md](overview.md).

Связанные разделы: бизнес-логика — [../02-business-logic/tenders.md](../02-business-logic/tenders.md) (сущности, статусы, права), [../02-business-logic/tenders-lifecycle.md](../02-business-logic/tenders-lifecycle.md) (мастер, поля, сроки, закрытие, правка ФКП), [../02-business-logic/tenders-invitations.md](../02-business-logic/tenders-invitations.md), [../02-business-logic/tenders-quotes.md](../02-business-logic/tenders-quotes.md), [../02-business-logic/tenders-comparison.md](../02-business-logic/tenders-comparison.md), [../02-business-logic/tenders-rounds.md](../02-business-logic/tenders-rounds.md), [../02-business-logic/tenders-documents.md](../02-business-logic/tenders-documents.md), [../02-business-logic/fkp-import.md](../02-business-logic/fkp-import.md), [../02-business-logic/insights.md](../02-business-logic/insights.md), [../02-business-logic/end-to-end-scenarios.md](../02-business-logic/end-to-end-scenarios.md); экраны — [../03-screens/tenders-registry.md](../03-screens/tenders-registry.md), [../03-screens/tender-create.md](../03-screens/tender-create.md), [../03-screens/tender-card.md](../03-screens/tender-card.md), [../03-screens/templates.md](../03-screens/templates.md), [../03-screens/home.md](../03-screens/home.md), [../03-screens/fill.md](../03-screens/fill.md).

Сложность раздела тендеров (77 операций): **XL**. Этого файла — **L**: создание с проверками (M), правка полей с каскадом сроков на раунды и закрытием (L), правка ФКП с версиями и рассылкой (L), пересборка разделов (L), приглашения с раундом №1 и письмами (M).

## Оглавление раздела тендеров

| Файл | Что внутри | Операций |
|---|---|---|
| этот файл | карточка, создание, правка, статус, состав ФКП, заметки, лента, `/statuses/usage`, приглашения | 14 |
| [endpoints-tenders-import.md](endpoints-tenders-import.md) | импорт ФКП (`/tenders/parse-excel*`), черновики мастера (`/creation-drafts*`); последовательность «импорт → черновик → тендер» | 13 |
| [endpoints-tenders-templates.md](endpoints-tenders-templates.md) | шаблоны тендеров (`/templates*`) | 6 |
| [endpoints-tenders-comparison.md](endpoints-tenders-comparison.md) | сравнительная таблица и выгрузка, протокол, версии КП, решения по корректировкам, доработка, комментарии к позициям | 14 |
| [endpoints-tenders-analysis.md](endpoints-tenders-analysis.md) | ИИ-анализ тендера (`/tenders/{id}/insights*`); последовательность разбора | 15 |
| [endpoints-tenders-invite.md](endpoints-tenders-invite.md) | форма КП по ссылке (`/invite/{token}*`); последовательность подачи КП файлом Excel | 15 |

Счёт — по первому тегу операции в `openapi.json`: `tenders` (64), `creation-drafts` (7), `templates` (6). Раунды (`rounds`) — [endpoints-rounds.md](endpoints-rounds.md); входящие КП (`quotes`) — [endpoints-quotes.md](endpoints-quotes.md); комментарии к ячейкам (создание) — [endpoints-comments.md](endpoints-comments.md); переписка — [endpoints-chat.md](endpoints-chat.md); реестр тендеров и отчёты — [endpoints-reports.md](endpoints-reports.md).

## Сводка всех операций раздела

| Метод | Путь | Кратко | Доступ | Файл |
|---|---|---|---|---|
| POST | `/tenders` | Создать тендер из разобранной структуры | `tenders.manage` + объект | этот |
| GET | `/tenders/{tender_id}` | Карточка тендера | вошедший + объект | этот |
| PATCH | `/tenders/{tender_id}` | Правка полей, статуса, ответственного | `tenders.manage` (+ пароль для закрытия/отмены) | этот |
| POST | `/tenders/{tender_id}/edit-preview` | Что сдвинется при правке сроков | `tenders.manage` | этот |
| PUT | `/tenders/{tender_id}/structure` | Дополнить ФКП разделами/позициями/условиями | сотрудник + объект | этот |
| POST | `/tenders/{tender_id}/structure/rebuild` | Пересобрать разделы по файлу ФКП | `tenders.manage` | этот |
| PATCH | `/tenders/{tender_id}/items` | Править объём/единицу/имя, снять/вернуть позиции | сотрудник + объект | этот |
| POST | `/tenders/{tender_id}/items` | Вставить позицию | сотрудник + объект | этот |
| PUT | `/tenders/{tender_id}/staff-notes` | Заметка сотрудников (автосохранение) | сотрудник + объект | этот |
| GET | `/tenders/{tender_id}/activity` | Лента событий тендера | вошедший + объект | этот |
| GET | `/statuses/usage` | Сколько тендеров и КП помечено пользовательскими статусами | сотрудник | этот |
| POST | `/tenders/{tender_id}/invitations` | Пригласить контрагентов | `tenders.manage` | этот |
| POST | `/tenders/{tender_id}/invitations/{invitation_id}/regenerate` | Перевыпустить персональную ссылку | `tenders.manage` + пароль | этот |
| POST | `/tenders/{tender_id}/invitations/{invitation_id}/revoke` | Отозвать ссылку (отключить участника) | `tenders.manage` + пароль | этот |
| POST | `/tenders/parse-excel/sheets` | Листы книги | сотрудник | import |
| POST | `/tenders/parse-excel` | Поставить разбор Excel | сотрудник | import |
| GET | `/tenders/parse-excel/jobs/{job_id}` | Статус/результат разбора | сотрудник | import |
| POST | `/tenders/parse-excel/jobs/{job_id}/cancel` | Отменить разбор | сотрудник | import |
| POST | `/tenders/parse-excel/remap` | Разбор по карте колонок | сотрудник | import |
| POST | `/tenders/parse-excel/enrich` | ИИ-подсказки по файлу | сотрудник | import |
| POST | `/creation-drafts` | Создать черновик мастера | сотрудник | import |
| GET | `/creation-drafts` | Список черновиков | сотрудник | import |
| GET | `/creation-drafts/{draft_id}` | Черновик | автор/админ | import |
| PUT | `/creation-drafts/{draft_id}` | Сохранить черновик | автор/админ | import |
| DELETE | `/creation-drafts/{draft_id}` | Удалить черновик | автор/админ | import |
| POST | `/creation-drafts/{draft_id}/remap` | Разбор сохранённого файла по карте колонок | автор/админ | import |
| POST | `/creation-drafts/{draft_id}/enrich` | ИИ-подсказки по сохранённому файлу | автор/админ | import |
| GET | `/templates` | Список шаблонов | сотрудник | templates |
| POST | `/templates` | Создать шаблон | сотрудник | templates |
| POST | `/templates/from-tender/{tender_id}` | Шаблон из тендера | сотрудник | templates |
| PUT | `/templates/{template_id}` | Изменить шаблон | сотрудник | templates |
| DELETE | `/templates/{template_id}` | Удалить шаблон | сотрудник | templates |
| PATCH | `/templates/{template_id}/column-map` | Запомнить карту колонок | сотрудник | templates |
| GET | `/tenders/{tender_id}/comparison` | Сравнительная таблица | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/comparison/export` | Выгрузка СТ в xlsx | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/protocol` | Черновик протокола | + «Цены» | comparison |
| PUT | `/tenders/{tender_id}/protocol` | Сохранить протокол | `tenders.manage` | comparison |
| GET | `/tenders/{tender_id}/protocol/export` | Протокол в docx | + «Цены» | comparison |
| GET | `/tenders/{tender_id}/quotes/{quote_id}/versions` | Версии КП | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/quotes/{quote_id}/versions/{version_number}` | Версия КП | сотрудник + объект | comparison |
| POST | `/tenders/{tender_id}/quote-items/{item_id}/review` | Решение по корректировке | сотрудник + объект | comparison |
| POST | `/tenders/{tender_id}/quotes/{quote_id}/review-batch` | Решения пачкой | сотрудник + объект | comparison |
| POST | `/tenders/{tender_id}/quotes/{quote_id}/request-revision` | Вернуть КП на доработку | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/comments` | Переписка по ячейкам СТ | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/position-comments` | Комментарии поставщиков к позициям | сотрудник + объект | comparison |
| GET | `/tenders/{tender_id}/comment-workload` | Счётчики «не отработано» | сотрудник + объект | comparison |
| POST | `/tenders/{tender_id}/position-comments/read` | Отметить просмотренным | сотрудник + объект | comparison |
| POST | `/tenders/{tender_id}/insights` | Сводка/сценарий на правилах | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights` | То же (GET) | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/run` | Статус разбора | «Цены» + объект | analysis |
| POST | `/tenders/{tender_id}/insights/run` | Запустить разбор | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/run/{result_id}` | Результат разбора | «Цены» + объект | analysis |
| POST | `/tenders/{tender_id}/insights/pair` | Разбор пары | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/pair` | Результат пары | «Цены» + объект | analysis |
| POST | `/tenders/{tender_id}/insights/letter` | Черновик письма | «Цены» + объект | analysis |
| POST | `/tenders/{tender_id}/insights/ask` | Вопрос по таблице | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/chats` | Мои разговоры | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/chat` | Реплики разговора | «Цены» + объект | analysis |
| DELETE | `/tenders/{tender_id}/insights/chat` | Удалить разговор | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/latest` | Последняя сводка | «Цены» + объект | analysis |
| GET | `/tenders/{tender_id}/insights/history` | История разборов | «Цены» + объект | analysis |
| POST | `/tenders/{tender_id}/insights/{result_id}/apply` | Переход в таблицу | «Цены» + объект | analysis |
| GET | `/invite/{token}` | Форма КП | по токену | invite |
| GET | `/invite/{token}/activity` | Лента контрагента | по токену | invite |
| GET | `/invite/{token}/draft` | Черновик формы | по токену | invite |
| PUT | `/invite/{token}/draft` | Сохранить черновик | по токену | invite |
| POST | `/invite/{token}/quote` | Подать КП | по токену | invite |
| POST | `/invite/{token}/ack-corrections` | Ознакомлен с изменениями ФКП | по токену | invite |
| GET | `/invite/{token}/documents/{attachment_id}` | Скачать документ | по токену | invite |
| GET | `/invite/{token}/documents/{attachment_id}/presign` | Ссылка на скачивание | по токену | invite |
| GET | `/invite/{token}/attachments` | Свои файлы к КП | по токену | invite |
| POST | `/invite/{token}/quote-import` | Разбор заполненного Excel КП | по токену | invite |
| POST | `/invite/{token}/files` | Приложить файл | по токену | invite |
| POST | `/invite/{token}/files/presign-upload` | Ссылка на прямую загрузку | по токену | invite |
| POST | `/invite/{token}/files/confirm` | Подтвердить загрузку | по токену | invite |
| DELETE | `/invite/{token}/files/uploads/{upload_id}` | Отменить загрузку | по токену | invite |
| DELETE | `/invite/{token}/files/{attachment_id}` | Удалить файл | по токену | invite |

## Доступ — общее

- **Доступ к тендеру** — через его объект (проект): администратор видит все; остальные — объекты по грантам на портфель/объект. Нет тендера — 404 «Тендер не найден»; объект не выдан — 403 «Нет доступа к тендерам этого проекта». Контрагент (роль `supplier`) объектов не имеет — ручки этого файла для него закрыты тем же 403 (форма КП — по ссылке-приглашению, [endpoints-tenders-invite.md](endpoints-tenders-invite.md)).
- **`tenders.manage`** — право матрицы (по умолчанию тендерный специалист; администратор — всегда); отказ — 403 «Недостаточно прав для этого действия — обратитесь к администратору системы» + `permission_denied` в журнале ИБ. Ручки состава ФКП, заметок и решений по КП требуют только гейта «сотрудник» и доступа к объекту — то есть технически доступны и СБ/юристу с доступом к объекту (см. [tenders.md §5](../02-business-logic/tenders.md#5-права-роль--действие) и открытые вопросы).
- **Повторный ввод пароля** (overview §2.5) — заголовок `X-Reauth-Password` с текущим паролем пользователя: нужен для закрытия и отмены тендера, перевыпуска и отзыва ссылки. Нет заголовка — 403 «Подтвердите действие паролем»; неверный — 403 «Пароль не подошёл — действие не выполнено».
- **Заморозка.** В статусах «Завершён»/«Отменён» правка состава ФКП запрещена: 400 «Тендер в статусе «<статус>» — состав заморожен. Правки возможны только после административного восстановления статуса.» Полный перечень запретов — [tenders.md §4.5](../02-business-logic/tenders.md#45-заморозка-после-завершения).
- Ответы, возвращающие карточку после записи (`POST /tenders`, `PATCH`, правка состава), отдают плановую сумму без маскировки (см. открытые вопросы); `GET /tenders/{id}` маскирует её без права «Цены».

## Объект `TenderDetail` (карточка)

| Поле | Тип | Смысл |
|---|---|---|
| `id`, `number` | integer, string | номер `T-ГГГГ-NNN` |
| `title` | string | название |
| `status` | string | `draft`, `open`, `closed`, `cancelled` ([tenders.md §4.1](../02-business-logic/tenders.md#41-системные-статусы)) |
| `custom_status_code` | string \| null | пользовательский статус |
| `needs_attention` | boolean | «Требует внимания» |
| `responsible` | string | ФИО ответственного (`""` — не назначен) |
| `start`, `end` | string | даты сбора КП `ГГГГ-ММ-ДД` (`""` — нет) |
| `project_id`, `project_name`, `portfolio_name`, `object_name` | | объект: «Портфель, Объект» |
| `staff_notes` | `{text, updated_by, updated_at}` | заметка сотрудников |
| `work_type`, `work_type_ids`, `work_type_names` | string, integer[], string[] | виды работ (первый — основной) |
| `budget_position_id`, `budget_position_name`, `budget_position_code` | | бюджетная позиция: полный путь «Раздел › Подраздел › Статья» и код |
| `budget_amount` | number \| null | плановая сумма (`null` без права «Цены») |
| `finance_masked` | boolean | плановая сумма скрыта от смотрящего |
| `winner_supplier_id`, `winner_supplier_name` | | победитель |
| `winner_accreditation_status`, `winner_accreditation_label` | string \| null | сводный статус аккредитации победителя на момент выбора («Победитель без аккредитации: …») |
| `sections[]` | array | `{id, name, code, parent_section_id, unit, quantity, items_before, items: [{id, name, unit, quantity, original_quantity, is_removed, is_corrected, item_number, budget_code, mark, equipment_code, manufacturer, attrs}]}` в порядке ФКП; `items_before` — место подраздела среди позиций родителя |
| `condition_fields[]` | array | `{id, field_name, field_type}` |
| `invitations[]` | array | `{id, supplier_id, supplier_name, status, invite_token, invited_at, notification_sent_at (ДД.ММ.ГГГГ ЧЧ:ММ, МСК), supplier_is_active, access_closed, access_reason (invite_revoked/org_disabled), joined_round_number, access_revoked_by, access_revoked_at, supplier_overall_status}` |

---

## Тендер

### POST /tenders

**Назначение.** Создать тендер в статусе «Черновик» из **уже разобранной** структуры (шаг 2 мастера). Ошибка разбора файла не создаёт тендер-сироту: без позиций, объекта, бюджетной позиции — 400. Ответственным становится создатель.

**Доступ.** `tenders.manage` + доступ к выбранному объекту.

**Тело запроса (JSON):**

| Поле | Тип | Обяз. | Правила и тексты ошибок |
|---|---|---|---|
| `project_id` | integer | да | объект; вместе с `title` — «Укажите проект (объект) и название тендера»; нет доступа — 403 «Нет доступа к тендерам этого проекта» |
| `title` | string | да | обрезка пробелов |
| `sections` | array | да | дерево `{name, code?, unit?, quantity?, items_before?, items: [{name, unit, quantity, item_number?, budget_code?, mark?, equipment_code?, manufacturer?, attrs_json?}], children?}` или плоский список; пустые позиции и узлы отбрасываются; пусто — «Нельзя создать тендер без позиций — загрузите файл и дождитесь разбора»; имя раздела > 500 — «Имя раздела длиннее 500 символов — сократите его: «…»»; позиции > 1000 — «Имя позиции длиннее 1000 символов — сократите его: «…»»; код раздела ≤ 100 символов без хвостовой точки |
| `budget_position_id` | integer | да | «Укажите бюджетную позицию»; «Бюджетная позиция не найдена»; «Нельзя привязать архивную бюджетную позицию»; «Период ГГГГ по этой позиции закрыт — новые тендеры нельзя привязывать» (год — по дате начала сбора, иначе текущий) |
| `budget_amount` | number \| string | нет | плановая сумма ≥ 0, пробелы и запятая допустимы, ≤ 9 999 999 999 999,99: «Некорректная плановая сумма», «Плановая сумма не может быть отрицательной», «Плановая сумма слишком большая — не больше 9 999 999 999 999,99 ₽» |
| `work_type_ids` (или `work_type_id`) | integer[] | да | ≥ 1, дубли схлопываются: «Выберите хотя бы один вид работ»; «Вид работ не найден или архивный» |
| `start_date`, `end_date` | string `ГГГГ-ММ-ДД` | да | «Укажите даты начала и окончания сбора КП»; «Дата окончания сбора КП должна быть позже даты начала» |
| `condition_fields` | `{field_name, field_type}[]` | нет | пустые названия отбрасываются; `field_type`: `text` (по умолчанию), `number`, `date`, `boolean` |
| `draft_id` | string (UUID) | нет | черновик мастера → `converted`, исходный файл → документ «Шаблон ФКП» (ошибки черновика не мешают созданию) |
| `template_id` | integer | нет | шаблон-источник (запоминается) |

Единицы позиций — только из справочника единиц (с синонимами): неизвестные — 400 «Нет в справочнике единиц: «…», «…» и ещё N. Выберите единицу из справочника или добавьте её туда.» (первые 5). Проверки выполняются до записи.

**Ответ `201`:** `{"item": TenderDetail}`.

**Ошибки.** 400 — правила выше; 403 — право/объект; 422 — не JSON.

**Пример.** `{"project_id": 24, "title": "Стяжки полов, корпус 1", "work_type_ids": [5], "budget_position_id": 31, "budget_amount": "1 000 000", "start_date": "2026-10-01", "end_date": "2026-10-15", "sections": [{"name": "Стяжки", "items": [{"name": "Стяжка цементно-песчаная 50 мм", "unit": "м²", "quantity": 600}]}], "condition_fields": [{"field_name": "Срок выполнения работ, мес.", "field_type": "number"}], "draft_id": "3b1e…"}` → `201 {"item": {"id": 111, "number": "T-2026-111", "status": "draft", "responsible": "Иванов Иван Иванович", …}}`

**Побочные эффекты.** Тендер, связки видов работ, разделы, позиции, условия; номер `T-<год UTC>-<id, 3+ знака>`; черновик мастера → `converted`, исходник перевешивается на тендер.

---

### GET /tenders/{tender_id}

**Назначение.** Карточка тендера: поля, состав ФКП, условия, участники.

**Доступ.** Вошедший + доступ к объекту (контрагенту — 403).

**Параметры пути:** `tender_id` — integer.

**Ответ `200`:** `{"item": TenderDetail}`.

**Ошибки.** 404 «Тендер не найден»; 403 «Нет доступа к тендерам этого проекта».

**Пример.** → `{"item": {"id": 100, "number": "T-2026-100", "title": "Витражи, дворовой фасад", "status": "open", "custom_status_code": null, "needs_attention": true, "responsible": "Иванов Иван Иванович", "start": "2026-08-26", "end": "2026-10-06", "object_name": "ЖК «Северный парк», МКД-1", "work_type_ids": [17], "budget_position_code": "MKD.09", "budget_amount": 269100000.0, "finance_masked": false, "winner_supplier_id": null, "sections": [{"id": 1005, "name": "Витраж Вт-1", "code": "1", "parent_section_id": 1004, "items": [{"id": 2504, "name": "Профильная система", "unit": "м²", "quantity": 14.85, "is_removed": false, "attrs": {"тип": "Тип1"}}]}], "condition_fields": [{"id": 128, "field_name": "Условия оплаты (авансирование)", "field_type": "text"}], "invitations": [{"id": 112, "supplier_id": 201, "supplier_name": "ООО «Альфа Фасад»", "status": "submitted", "invite_token": "<токен>", "invited_at": "26.08.2026 10:15", "access_closed": false, "joined_round_number": 1, "supplier_overall_status": "accredited"}]}}`

**Побочные эффекты.** Нет.

---

### PATCH /tenders/{tender_id}

**Назначение.** Правка полей тендера после создания: название, сроки, виды работ, объект, бюджетная позиция, плановая сумма, системный статус (в т. ч. закрытие с победителем и отмена), пользовательский статус, ответственный. Меняются только переданные поля; все проверки выполняются до писем каскада.

**Доступ.** `tenders.manage` + объект. Название, сроки и виды работ — только администратор, создатель или ответственный (у старых тендеров без обоих — любой с доступом). Перевод в `closed`/`cancelled` — повторный ввод пароля.

**Тело запроса (все поля необязательны):**

| Поле | Тип | Правила и тексты |
|---|---|---|
| `title` | string | «Название тендера не может быть пустым»; «Название длиннее 500 символов» |
| `start_date`, `end_date` | `ГГГГ-ММ-ДД` | пара проверяется как при создании; смена начала после приглашений — нужен `confirm_details_change`; смена окончания — каскад на раунды (ниже) |
| `work_type_ids` / `work_type_id` | integer[] | «Выберите хотя бы один вид работ»; после приглашений смена набора — нужен `confirm_details_change` |
| `confirm_details_change` | boolean | подтверждение: «После приглашения участников смена начала сбора требует подтверждения (confirm_details_change)» / «… смена видов работ требует подтверждения (confirm_details_change)» |
| `project_id` | integer | «Укажите объект (проект)»; после приглашений — `confirm_project_change` («После приглашения участников смена объекта требует подтверждения (confirm_project_change)»); доступ к новому объекту |
| `budget_position_id` | integer | `null` — «Бюджетная позиция обязательна»; проверки как при создании (год — по дате начала) |
| `budget_amount` | number \| string \| null | как при создании |
| `status` | string | `draft`, `open`, `closed`, `cancelled`: «Недопустимый статус тендера»; из `closed`/`cancelled` в нетерминальный — «Переход из «Завершён»/«Отменён» запрещён без административного восстановления с причиной» |
| `winner_supplier_id` | integer | при `status: closed` (обязателен, если победителя ещё нет): «Для статуса «Завершён» укажите победителя (winner_supplier_id)»; «Победитель должен быть участником этого тендера»; «Контрагент не найден»; без смены статуса на закрытом — «Победитель задаётся при переводе в «Завершён»» |
| `confirm_non_accredited`, `non_accredited_comment` | boolean, string | победитель без аккредитации: без подтверждения — 400 `{"code": "non_approved_confirmation_required", "message": "Контрагент не прошёл аккредитацию: <статус>. Подтвердите выбор победителя и укажите комментарий.", "status", "status_label"}`; без комментария — «Укажите комментарий к выбору победителя без аккредитации» |
| `status_comment` (или `comment`) | string | комментарий к смене статуса (в ленту) |
| `custom_status_code` | string \| null | код из справочника статусов тендера: «Неизвестный статус тендера»; `null`/`""` — снять |
| `responsible_user_id` | integer \| null | действующий сотрудник: «Ответственным можно назначить только действующего сотрудника»; `null` — снять |

**Каскад даты окончания на раунды** ([tenders-lifecycle.md §7.1](../02-business-logic/tenders-lifecycle.md#71-правка-сроков-после-создания-с-каскадом-на-раунды)): раунды, чей срок совпадал с прежней датой окончания, переносятся на новую дату 23:59 МСК; отправленным раундам при продлении — уведомление и письмо адресатам; сокращение срока отправленного неистёкшего раунда блокирует всю правку: 400 «Срок отправленного раунда №N сократить нельзя. Закройте раунд и создайте новый».

**Ответ `200`:** `{"item": TenderDetail}`.

**Ошибки.** 400 — правила выше; 400 «Тендер в статусе «Завершён» — название, сроки и виды работ не меняются»; 403 «Название, сроки и виды работ может менять создатель, ответственный или администратор»; 403 — пароль («Подтвердите действие паролем», «Пароль не подошёл — действие не выполнено»); 403/404 — общие.

**Пример.** Закрытие: заголовок `X-Reauth-Password: <пароль>`, тело `{"status": "closed", "winner_supplier_id": 207, "status_comment": "Решение комиссии"}` → `{"item": {"status": "closed", "winner_supplier_id": 207, "winner_supplier_name": "АО «Эта Окна»", …}}`

**Побочные эффекты.** Лента: `tender_details_changed` (поле, было, стало) для названия, дат и видов работ; `tender_status_changed`; `winner_non_accredited_confirmed` (+ журнал ИБ); `tender_custom_status_changed`; `tender_responsible_changed`; `round_deadline_extended` при каскаде. Закрытие: КП победителя → `accepted`, `closed_at`, идущие раунды закрываются, уведомления и письма ответственному, победителю и остальным участникам ([tenders-lifecycle.md §8](../02-business-logic/tenders-lifecycle.md#8-закрытие-тендера-и-выбор-победителя)). Отмена: `closed_at`, раунды закрываются, писем нет. Время смены статуса (`status_changed_at`).

---

### POST /tenders/{tender_id}/edit-preview

**Назначение.** Предпросмотр правки (ничего не пишет): какие раунды сдвинутся при смене даты окончания и сколько адресатов получат письмо, какие раунды «отвязаны» или заблокированы, нужно ли подтверждение менеджера.

**Доступ.** `tenders.manage` + объект; для правки названия/сроков/видов работ — те же ограничения, что у PATCH.

**Тело запроса:** те же поля, что у `PATCH /tenders/{tender_id}` (учитываются `start_date`, `end_date`, `work_type_ids`/`work_type_id`).

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `rounds[]` | array | `{round_number, status ("draft"/"sent"), from (ISO|null), to (ISO), notify (сколько адресатов получат письмо)}` |
| `detached` | integer[] | номера раундов со своим сроком (не сдвигаются) |
| `blocked` | integer[] | номера отправленных раундов, срок которых сократился бы (правка будет отклонена) |
| `notify_total` | integer | всего писем |
| `needs_confirm` | boolean | нужен `confirm_details_change` (меняется начало или виды работ, и есть приглашённые) |

**Ошибки.** 400 — даты, статус тендера; 403 — права на детали; общие.

**Пример.** `{"end_date": "2026-10-20"}` → `{"rounds": [{"round_number": 1, "status": "sent", "from": "2026-10-15T20:59:00+00:00", "to": "2026-10-20T20:59:00+00:00", "notify": 3}], "detached": [], "blocked": [], "notify_total": 3, "needs_confirm": false}`

**Побочные эффекты.** Нет.

---

### PUT /tenders/{tender_id}/structure

**Назначение.** Дополнить ФКП после старта: новые разделы с позициями и доп. условия добавляются в конец. Снимок версии ФКП создаётся при добавлении разделов/позиций; рассылка участникам — при `notify=true` (по умолчанию).

**Доступ.** Сотрудник + объект; тендер не в терминальном статусе.

**Тело запроса:**

| Поле | Тип | Обяз. | Правила |
|---|---|---|---|
| `sections[]` | array | нет | `{name, code?, parent_section_id?, items: [{name, unit, quantity, item_number?, budget_code?, mark?, equipment_code?, manufacturer?, …}]}`; разделы и позиции без названия пропускаются; неизвестная единица отменяет всё дополнение (400 как при создании) |
| `condition_fields[]` | array | нет | `{field_name, field_type}`; пустые пропускаются |
| `notify` | boolean | нет, `true` | разослать новую версию ФКП участникам |

**Ответ `200`:** `{"item": TenderDetail, "added_sections": integer, "added_items": integer, "notified": integer (сколько участников уведомлено), "fkp_version": {id, tender_id, version_number, author_id, change_summary, notified_at, created_at} | null, "notify_pending": boolean}`.

**Ошибки.** 400 «Тендер в статусе «…» — состав заморожен…»; 400 — единицы; общие.

**Пример.** `{"sections": [{"name": "Дополнительные работы", "items": [{"name": "Демонтаж старых витражей", "unit": "м²", "quantity": 40}]}], "notify": true}` → `{"item": {…}, "added_sections": 1, "added_items": 1, "notified": 6, "fkp_version": {"version_number": 2, "change_summary": "разделов +1; позиций +1", "notified_at": "2026-10-15T09:00:00+00:00"}, "notify_pending": false}`

**Побочные эффекты.** «Требует внимания» = true; версия ФКП (снимок разделов, позиций, условий); ревизия тендера растёт; лента `structure_updated` (видно поставщикам); `notify=true` — уведомление «Обновлён шаблон ФКП (vN)» и письмо «Новая версия ФКП» участникам в стадиях «отправлено/доставлено/заполняет/подано»; `notify=false` — версия не разослана, автору уведомление «Версия ФКП создана — рассылка отложена». Рассылка отложенных версий — ручкой раздела раундов ([endpoints-rounds.md](endpoints-rounds.md)).

---

### POST /tenders/{tender_id}/structure/rebuild

**Назначение.** Переложить позиции тендера в дерево свежего разбора его «Шаблона ФКП»: по умолчанию — только план (`dry_run=true`): новое оглавление и как сопоставлены позиции. Запись: новые разделы с номерами из файла, позиции переезжают (id, цены КП, комментарии сохраняются), старые разделы удаляются; строки-заголовки, которые прежний разбор взял за позиции, становятся подразделами, а сами позиции удаляются вместе со строками КП (обсуждения их ячеек переезжают на первую позицию подраздела). Состав и объёмы не меняются, рассылки нет.

**Доступ.** `tenders.manage` + объект; запись — только не в терминальном статусе.

**Параметры запроса:** `dry_run` — boolean, по умолчанию `true`.

**Тело запроса (`multipart/form-data`, необязательно):** `file` — файл ФКП (≤ 50 МБ), если у тендера нет сохранённого исходника (тендер перенесён из прежней системы); вложения тендера не меняются.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `tender_id`, `dry_run` | | |
| `items` | integer | позиций в тендере |
| `matched`, `by_code`, `by_neighbor`, `file_only` | integer | сколько позиций сопоставлено, из них по номеру / по соседству; строк только в файле |
| `sections` | integer | разделов в новом дереве |
| `outline` | array | новое оглавление |
| `absorbed[]` | array | позиции-заголовки, становящиеся подразделами: `{id, item_number, name, section, blocked (в текущих КП есть цены/отметки — запись невозможна), kept_removed (отметки только в прежних редакциях — остаётся снятой позицией), supplier_comments[]}` |
| `absorbed_quote_rows` | integer | сколько строк КП уйдёт |
| `formula_check` | object \| null | сверка итогов ФКП с деревом |

**Ошибки.** 400 «У тендера нет исходного файла ФКП — приложите файл ФКП, по которому пересобрать разделы»; 400 «В файле ФКП не нашлось ни одного раздела»; 400 «План пересборки потерял позиции тендера — пересборка не выполнена»; 400 (запись) «Строки-заголовки ФКП стали подразделами, но в текущих КП по ним есть цены или отметки специалиста: <номера>. Снимает их новая редакция КП подрядчика (новый раунд) — после неё повторите пересборку.»; 400 — заморозка; 413; общие.

**Пример.** `POST /tenders/100/structure/rebuild?dry_run=true` → `{"tender_id": 100, "dry_run": true, "items": 898, "matched": 898, "by_code": 870, "by_neighbor": 28, "file_only": 0, "sections": 202, "outline": […], "absorbed": [], "absorbed_quote_rows": 0, "formula_check": null}`

**Побочные эффекты.** Только при `dry_run=false`: «тихая» версия ФКП «Разделы пересобраны по файлу ФКП» (не рассылается и не делает КП устаревшими), ревизия тендера растёт, лента `structure_rebuilt`.

---

### PATCH /tenders/{tender_id}/items

**Назначение.** Корректировка существующих позиций пачкой: новый объём (исходный запоминается), снятие позиции (мягко), восстановление, новая единица, новое наименование.

**Доступ.** Сотрудник + объект; не в терминальном статусе.

**Тело запроса:**

| Поле | Тип | Правила |
|---|---|---|
| `items[]` | array | `{id, quantity?, unit?, name?, remove?, restore?}`; чужие/несуществующие id пропускаются; `restore: true` — вернуть снятую; `remove: true` — снять; `quantity` (`null`/`""` — очистить); `unit` — из справочника (неизвестная отменяет всю пачку); `name` — непустое |
| `notify` | boolean | по умолчанию `true` |

**Ответ `200`:** `{"item": TenderDetail, "qty_changed": integer, "removed": integer, "notified": integer, "fkp_version": object|null, "notify_pending": boolean}`.

**Ошибки.** 400 — заморозка, единицы; общие.

**Пример.** `{"items": [{"id": 2504, "quantity": 16.2}, {"id": 2505, "remove": true}], "notify": false}` → `{"item": {…}, "qty_changed": 1, "removed": 1, "notified": 0, "fkp_version": {"version_number": 3, "change_summary": "изменён объём: 1; снято позиций: 1"}, "notify_pending": true}`

**Побочные эффекты.** Позиции помечаются скорректированными; у всех участников сбрасывается «ознакомлен с изменениями»; «Требует внимания» = true; если менялись объём или снятие — версия ФКП, ревизия, лента `structure_updated`, рассылка (как у `PUT /structure`); автору — уведомление «Корректировки ФКП применены».

---

### POST /tenders/{tender_id}/items

**Назначение.** Вставить позицию после указанной (или в конец раздела); хвост раздела и подразделы, стоящие среди его позиций, сдвигаются.

**Доступ.** Сотрудник + объект; не в терминальном статусе.

**Тело запроса:**

| Поле | Тип | Обяз. | Правила |
|---|---|---|---|
| `name` | string | да | «Укажите наименование» |
| `after_item_id` | integer | одно из двух | позиция, после которой вставить: «Позиция after_item_id не найдена» |
| `section_id` | integer | одно из двух | раздел (вставка в конец): «Раздел не найден»; нет обоих — «Нужен section_id или after_item_id» |
| `unit` | string | нет | из справочника единиц |
| `quantity` | number \| "" | нет | |
| `notify` | boolean | нет, `true` | |

**Ответ `201`:** `{"item": TenderDetail, "inserted_id": integer, "fkp_version": object, "notified": integer, "notify_pending": boolean}`.

**Ошибки.** 400/404 — выше; 400 — заморозка, единица; общие.

**Пример.** `{"after_item_id": 2504, "name": "Уплотнитель EPDM", "unit": "м", "quantity": 120}` → `201 {"item": {…}, "inserted_id": 3400, "fkp_version": {"version_number": 4, "change_summary": "Добавлена позиция: Уплотнитель EPDM"}, "notified": 6, "notify_pending": false}`

**Побочные эффекты.** Позиция (скорректированная), сброс «ознакомлен», «Требует внимания», версия ФКП, ревизия, лента `structure_updated`, рассылка или уведомление «рассылка отложена».

---

### PUT /tenders/{tender_id}/staff-notes

**Назначение.** Рабочая заметка сотрудников по тендеру (автосохранение). Контрагентам не отдаётся.

**Доступ.** Сотрудник + объект.

**Тело запроса:** `{"text": string}` — до 20 000 символов (длиннее обрезается); пустая строка — очистить.

**Ответ `200`:** `{"text": string, "updated_by": string (ФИО), "updated_at": ISO|null}`.

**Ошибки.** Общие.

**Пример.** `{"text": "Ждём уточнения по крепежу от двух участников"}` → `{"text": "Ждём уточнения по крепежу от двух участников", "updated_by": "Иванов Иван Иванович", "updated_at": "2026-10-15T09:05:00Z"}`

**Побочные эффекты.** Автор и время правки (если текст изменился). В ленту не пишется.

---

### GET /tenders/{tender_id}/activity

**Назначение.** Лента событий тендера для сотрудника с фильтрами, курсором и настоящим `total` под теми же фильтрами. Каталог событий — [tenders-lifecycle.md §15](../02-business-logic/tenders-lifecycle.md#15-события-уведомления-аудит).

**Доступ.** Вошедший + объект.

**Параметры запроса:**

| Параметр | Тип | Смысл |
|---|---|---|
| `limit` | integer | 1–200, по умолчанию 50 |
| `before_id` | integer | курсор (id последнего показанного) |
| `kind` | string, повторяемый | группы: `quote`, `round`, `message`, `tender`, `participant` (неизвестные игнорируются) |
| `author_id` | integer | только действия этого сотрудника |
| `supplier_id` | integer | только события про этого контрагента |
| `q` | string | ≤ 100 символов: поиск по тексту события и автору |
| `since`, `until` | date | период, включительно, сутки МСК |

**Ответ `200`:** `{"items": [{id, action, entity_type, entity_id, details, actor, actor_kind (staff/supplier/system), subject_supplier_id, created_at}], "next_before_id": integer|null, "total": integer, "authors": [{id, name}], "suppliers": [{id, name}]}` — `authors`/`suppliers` — варианты фильтров. Новые сверху.

**Ошибки.** 422 — `limit`, `q` > 100, даты; общие.

**Пример.** → `{"items": [{"id": 407, "action": "tender_custom_status_changed", "entity_type": "tender", "entity_id": 100, "details": {"from_status": "budget_check", "to_status": null, "to_status_label": null}, "actor": "Админова Елена", "actor_kind": "staff", "subject_supplier_id": null, "created_at": "2026-10-02T16:48:34Z"}], "next_before_id": 407, "total": 46, "authors": [{"id": 2, "name": "Иванов Иван Иванович"}], "suppliers": [{"id": 201, "name": "ООО «Альфа Фасад»"}]}`

**Побочные эффекты.** Нет.

---

### GET /statuses/usage

**Назначение.** Сколько тендеров и КП помечено каждым пользовательским статусом — справочник статусов показывает это перед архивированием статуса. Живёт в разделе тендеров (теги `tenders`, `statuses`): справочник ниже тендеров по зависимостям и их таблиц не видит. Справочник статусов — [endpoints-catalog.md](endpoints-catalog.md).

**Доступ.** Сотрудник.

**Параметры.** Нет.

**Ответ `200`:** `{"tender": {"<код>": число}, "quote": {"<код>": число}}` — только коды, которыми что-то помечено.

**Ошибки.** 401/403.

**Пример.** → `{"tender": {"budget_check": 2}, "quote": {"clarification": 1}}`

**Побочные эффекты.** Нет.

---

## Приглашения

### POST /tenders/{tender_id}/invitations

**Назначение.** Пригласить контрагентов в тендер и выдать персональные ссылки `/fill/<токен>`. Правила — [tenders-invitations.md §3](../02-business-logic/tenders-invitations.md#3-создание-приглашений--правила-сервера).

**Доступ.** `tenders.manage` + объект.

**Тело запроса:**

| Поле | Тип | Обяз. | Правила |
|---|---|---|---|
| `supplier_ids` | integer[] | да | «Выберите хотя бы одного поставщика»; нечисловые и несуществующие пропускаются |
| `confirm_non_approved` | boolean | нет | подтверждение приглашения неодобренных (квалификация не «Одобрен»/«Одобрен с лимитом») |
| `non_approved_comment` | string | при подтверждении | «Укажите комментарий к приглашению неодобренных контрагентов» |

**Ответ `201`:** `{"items": [{supplier_id, supplier_name, invite_token, status, already_invited: false, qualification_status, joined_round_number}], "skipped_already_invited": [{supplier_id, supplier_name, invite_token, status, already_invited: true}]}` — `joined_round_number`: номер идущего отправленного раунда, в который добавлен новый участник (`null`, если такого нет).

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | тендер завершён/отменён | «Нельзя приглашать участников в закрытый или отменённый тендер» |
| 400 | пустой список | «Выберите хотя бы одного поставщика» |
| 409 | запрет «не приглашать до» не истёк | «Контрагент «<название>» отклонён, повторное приглашение заблокировано до ДД.ММ.ГГГГ» |
| 400 | неодобренные без подтверждения | `{"code": "non_approved_confirmation_required", "message": "Среди выбранных есть неодобренные контрагенты. Подтвердите приглашение явно и укажите комментарий.", "non_approved": [{supplier_id, supplier_name, status}]}` |
| 400 | все уже приглашены | «Все выбранные контрагенты уже приглашены в этот тендер» |
| 400 | ни одного существующего контрагента | «Не удалось создать приглашения» |

**Пример.** `{"supplier_ids": [201, 205]}` → `201 {"items": [{"supplier_id": 205, "supplier_name": "ООО «Эпсилон Строй»", "invite_token": "<токен>", "status": "invited", "already_invited": false, "qualification_status": "approved"}], "skipped_already_invited": [{"supplier_id": 201, "supplier_name": "ООО «Альфа Фасад»", "status": "submitted", "already_invited": true}]}`

**Побочные эффекты.** Приглашения (`invited`, случайный UUID v4, кто и когда); тендер `draft` → `open`; раунд №1 «В процессе» со всеми приглашёнными, если раундов нет; иначе новые — адресатами идущего отправленного раунда и черновика следующего; после сохранения — событие ленты `invitation_created` на каждого и письмо-приглашение по шаблону (результат доставки — в строке адресата раунда); при неодобренных — событие `invitation_non_approved_confirmed`.

---

### POST /tenders/{tender_id}/invitations/{invitation_id}/regenerate

**Назначение.** Перевыпуск персональной ссылки: в то же приглашение пишется новый токен, старая ссылка перестаёт работать; стадия сбрасывается в «отправлено». Также способ вернуть доступ отозванному участнику.

**Доступ.** `tenders.manage` + объект + повторный ввод пароля (`X-Reauth-Password`).

**Ответ `200`:** `{"item": {id, supplier_id, supplier_name, invite_token, status: "invited", invited_at}}`.

**Ошибки.** 404 «Приглашение не найдено» (нет или другого тендера); 403 — пароль; общие.

**Пример.** `POST /tenders/100/invitations/112/regenerate` (`X-Reauth-Password: <пароль>`) → `{"item": {"id": 112, "supplier_id": 201, "supplier_name": "ООО «Альфа Фасад»", "invite_token": "<новый токен>", "status": "invited", "invited_at": "2026-10-15T09:10:00+00:00"}}`

**Побочные эффекты.** Кто и когда пригласил обновляется; лента `participant_access_restored`; журнал ИБ `participant_access_restored` (IP, тендер, контрагент); письмо на e-mail организации «Обновлённая ссылка — тендер <номер>» с новой ссылкой и фразой «Старая ссылка больше не действует.»

---

### POST /tenders/{tender_id}/invitations/{invitation_id}/revoke

**Назначение.** Отозвать ссылку (отключить участника) без перевыпуска: статус `revoked`, токен заменяется «мёртвым» — старая ссылка не открывается, новой не выдаётся. Участник остаётся колонкой СТ с маркером «Доступ закрыт», исключается из расчётов и не может стать победителем.

**Доступ.** `tenders.manage` + объект + повторный ввод пароля.

**Ответ `200`:** `{"item": {id, supplier_id, supplier_name, invite_token (новый, недействующий), status: "revoked", invited_at}}`.

**Ошибки.** 404 «Приглашение не найдено»; 409 «Ссылка уже обнулена»; 403 — пароль; общие.

**Пример.** `POST /tenders/100/invitations/115/revoke` (`X-Reauth-Password: <пароль>`) → `{"item": {"id": 115, "supplier_id": 204, "status": "revoked", …}}`

**Побочные эффекты.** Контрагент удаляется из черновиков раундов (отправленные — история); лента `participant_revoked`; журнал ИБ `participant_access_revoked`; писем нет.
