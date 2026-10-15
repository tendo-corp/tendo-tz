# API: сравнение КП, протокол, решения по КП и комментарии к позициям

Ручки рабочего пространства тендера после приёма КП: сравнительная таблица (СТ) и её выгрузка в Excel, протокол комиссии (черновик реквизитов и docx), история версий КП участника, решения по корректировкам (своя позиция, изменение объёма — по одной и пачкой), запрос доработки КП, переписка сотрудников по ячейкам СТ и история комментариев поставщика к позициям с отметками «отработано». Общие соглашения — [overview.md](overview.md); оглавление раздела — [endpoints-tenders.md](endpoints-tenders.md). Входящие КП и карточка КП — [endpoints-quotes.md](endpoints-quotes.md); раунды — [endpoints-rounds.md](endpoints-rounds.md); создание комментариев к ячейкам — [endpoints-comments.md](endpoints-comments.md).

Связанные разделы: бизнес-логика — [../02-business-logic/tenders-comparison.md](../02-business-logic/tenders-comparison.md) (состав СТ, метрики, итоги, экспорт, маски), [../02-business-logic/tenders-quotes.md](../02-business-logic/tenders-quotes.md) (корректировки, версии, доработка), [../02-business-logic/tenders-documents.md](../02-business-logic/tenders-documents.md#3-протокол-тендера-docx) (протокол), [../02-business-logic/tenders-rounds.md](../02-business-logic/tenders-rounds.md); экраны — [../03-screens/tender-card.md](../03-screens/tender-card.md), [../03-screens/tender-card-comparison.md](../03-screens/tender-card-comparison.md).

Сложность блока: **XL** — СТ с аналитикой, кэшем и масками (XL, вместе с экраном), выгрузка СТ «по форме» с формулами (L), протокол docx (M), корректировки объёма с переоткрытием строк у всех участников (L), комментарии (S).

## Сводка

| Метод | Путь | Кратко | Доступ |
|---|---|---|---|
| GET | `/tenders/{tender_id}/comparison` | Сравнительная таблица с аналитикой | сотрудник + объект; цены — «Цены» |
| GET | `/tenders/{tender_id}/comparison/export` | Выгрузка СТ в xlsx | то же |
| GET | `/tenders/{tender_id}/protocol` | Черновик протокола + данные системы | сотрудник + объект + «Цены» |
| PUT | `/tenders/{tender_id}/protocol` | Сохранить реквизиты протокола | `tenders.manage` + объект |
| GET | `/tenders/{tender_id}/protocol/export` | Протокол комиссии в docx | сотрудник + объект + «Цены» |
| GET | `/tenders/{tender_id}/quotes/{quote_id}/versions` | История версий КП с диффом | сотрудник + объект |
| GET | `/tenders/{tender_id}/quotes/{quote_id}/versions/{version_number}` | Одна версия КП целиком | сотрудник + объект |
| POST | `/tenders/{tender_id}/quote-items/{item_id}/review` | Решение по одной корректировке | сотрудник + объект |
| POST | `/tenders/{tender_id}/quotes/{quote_id}/review-batch` | Решение по нескольким корректировкам объёма КП | сотрудник + объект |
| POST | `/tenders/{tender_id}/quotes/{quote_id}/request-revision` | Вернуть КП на доработку | сотрудник + объект |
| GET | `/tenders/{tender_id}/comments` | Вся переписка сотрудников по ячейкам СТ | сотрудник + объект |
| GET | `/tenders/{tender_id}/position-comments` | История комментариев поставщиков к позициям | сотрудник + объект |
| GET | `/tenders/{tender_id}/comment-workload` | Счётчики «не отработано» по ячейкам | сотрудник + объект |
| POST | `/tenders/{tender_id}/position-comments/read` | Отметить записи истории просмотренными | сотрудник + объект |

**Доступ — общее.** Гейт «сотрудник» (overview §3), затем доступ к объекту тендера: нет тендера — 404 «Тендер не найден»; объект не выдан — 403 «Нет доступа к тендерам этого проекта». Для ручек КП (`versions`, `request-revision`) проверка идёт через КП: роль `supplier` — 403 «Доступно только сотрудникам компании»; КП нет — 404 «КП не найдено»; приглашения/тендера КП нет — 404 «Приглашение не найдено» / «Тендер не найден»; объект недоступен — 403 «Нет доступа к проекту этого тендера»; КП другого тендера — 404 «КП не найдено». Тяжёлые чтения (СТ, выгрузка, протокол и его выгрузка) — лимит `heavy-read` на пользователя → 429.

---

## Сравнительная таблица

### GET /tenders/{tender_id}/comparison

**Назначение.** Данные экрана «Сравнение»: позиции (с путём по разделам), разделы, доп. условия, участники с их ценами по позициям (текущая или выбранная версия КП), итоги, заполненность, версии, признаки доступа, и посчитанная на сервере аналитика (медиана, MIN/MAX, разброс, аномалии, ключевые позиции, потенциал экономии, места и лидер, срезы «Итого / Материалы / Работы», именованные фильтры). Формулы — [tenders-comparison.md](../02-business-logic/tenders-comparison.md).

**Доступ.** Сотрудник + объект; лимит `heavy-read`. Без права «Цены» все денежные поля приходят пустыми (`null`), строки, участники и заполненность видны. При включённой маске контактов (`CONTACTS_MASK_ENABLED`) без права `suppliers.contacts.view` ИНН/e-mail/телефон участников маскируются (`contacts_masked: true`).

**Параметры запроса:** `versions` — string, необязателен: выбранные версии КП `"<supplier_id>:<номер версии>,…"` (мусорные пары пропускаются); по умолчанию — текущие версии.

**Заголовки.** Ответ с `ETag` и `Cache-Control: private, no-cache`; запрос с `If-None-Match` того же значения → **304** без тела. Сервер кэширует тело на `HEAVY_READ_CACHE_SECONDS` (60 с) по ключу: тендер, ревизия тендера, версии формул и денежных правил, отпечаток КП тендера, выбранные версии, пороги аналитики, право «Цены», право на контакты.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `items[]` | array | позиции: `id`, `section_id`, `section_name`, `section_path[]` (`{id, name, code, parent_section_id}` от корня), `name`, `unit`, `quantity`, `original_quantity`, `is_removed`, `is_corrected`, `order_index`, `item_number`, `budget_code`, `mark`, `equipment_code`, `manufacturer`, `attrs` |
| `sections[]` | array | `id`, `name`, `code`, `parent_section_id`, `order_index`, `unit`, `quantity`, `items_before`, `item_ids` |
| `condition_fields[]` | array | `id`, `field_name`, `field_type` |
| `suppliers[]` | array | участник (колонка): `supplier_id`, `supplier_name`, `inn`, `contact`, `email`, `phone`, `tax_system`, `invitation_id`, `invitation_status`, `quote_id`, `quote_status`, `quote_version_number`, `revision_comment`, `submitted`, `items` (`{tender_item_id: {quote_item_id, materials, works, total (суммы по строке), materials_unit, works_unit, unit_total (за единицу), quantity (эффективный объём), declined, comment, quantity_override, review_status, review_comment, awaiting_supplier_volume, refill_required, negotiation_potential, awaiting_reply}}`), `new_items[]` (свои позиции), `condition_values` (`{id условия: текст}`), `totals` (`{materials, works, total}`), `fill_progress`, `fill_priced`, `fill_declined`, `fill_total`, `fill_zero`, `versions[]` (`{version_number, is_current, submitted_at, has_items, prices}` — `prices`: цены за единицу прошлых версий для «динамики»), `fkp_version_number`, `fkp_outdated` (подано по устаревшей версии ФКП), `awaiting_round_number`, `access_closed`, `access_reason` (`invite_revoked` / `org_disabled`), `access_revoked_by`, `access_revoked_at` |
| `active_round_number` | integer \| null | текущий раунд |
| `fkp_version_current` | integer \| null | последняя версия ФКП |
| `fkp_outdated_count` | integer | сколько КП подано по устаревшей версии |
| `tender_revision` | integer | ревизия тендера |
| `winner_supplier_id` | integer \| null | победитель |
| `contacts_masked` | boolean | контакты замаскированы |
| `analytics` | object | `formula_version`, `price_slice`, `tender_vat_basis`, `settings` (пороги: `spread_notice`, `spread_high`, `anomaly_k`, `key_works_threshold`), `works` (метрики по позициям), `sections` (подытоги), `contractors` (сводка по участникам: итог, полнота, место, Δ к лидеру), `tender` (`work_count`, `contractor_count`, `complete_kp_count`, `incomplete_kp_count`, `totals_min`, `totals_max`, `totals_median`, `leader_id`, `key_work_ids`…), `named_filters` (`{имя: {work_ids}}`), `filtered`, `pair_deltas`, `weight_sum_median_costs`, `median_of_complete_totals`, `has_price_split`, `slices` (срезы «Итого / Материалы / Работы») — числа аналитики строками-десятичными |

Отозванные участники и участники приостановленных организаций остаются колонками с маркером «Доступ закрыт», но исключаются из расчётов.

**Ошибки.** 403/404 — общие; 429.

**Пример (сокращено).** → `{"items": [{"id": 2504, "section_id": 1005, "name": "Профильная система", "unit": "м²", "quantity": 14.85, "is_removed": false}], "suppliers": [{"supplier_id": 201, "supplier_name": "ООО «Альфа Фасад»", "inn": "7700000000", "email": "user@example.com", "quote_status": "submitted", "quote_version_number": 2, "totals": {"materials": 140040684.73, "works": 111905311.94, "total": 251945996.67}, "items": {"2504": {"materials_unit": 15520.0, "works_unit": 16975.0, "total": 482550.75, "quantity": 14.85, "declined": false}}, "fill_progress": 100}], "active_round_number": 2, "tender_revision": 18, "winner_supplier_id": null, "contacts_masked": false, "analytics": {"settings": {"spread_notice": "15.00", "spread_high": "40.00", "anomaly_k": "3.00", "key_works_threshold": "0.80"}, "tender": {"work_count": 898, "contractor_count": 7, "complete_kp_count": 2, "leader_id": "207"}}}`

**Побочные эффекты.** Нет (запись в серверный кэш).

---

### GET /tenders/{tender_id}/comparison/export

**Назначение.** Выгрузка СТ в `.xlsx`. По умолчанию — **полный состав ФКП**: все участники в порядке приглашения и все позиции на одном листе «по форме» заказчика, шапка с контактами участников (маска — по правилу контактов), формулы с кэшем значений; победитель и подсветка мин/макс не выделяются; при праве «Цены» под таблицей — выручка участников за три последних года. Экранный фильтр влияет только на легенду в шапке, а не на состав строк. Подробно — [tenders-comparison.md §12](../02-business-logic/tenders-comparison.md#12-экспорт-ст-в-excel).

**Доступ.** Сотрудник + объект; лимит `heavy-read`. Без права «Цены» денежные ячейки пусты.

**Параметры запроса:**

| Параметр | Тип | По умолчанию | Смысл |
|---|---|---|---|
| `full` | boolean | `true` | `false` — **устарело**: срез экрана (фильтр участников/позиций режет строки); ответ с `Deprecation: true`, `Sunset`, `Warning` |
| `versions` | string | — | версии КП `"<supplier_id>:<версия>,…"` |
| `supplier_ids`, `item_ids`, `favorite_ids` | string | — | id через запятую (`supplier_ids`/`item_ids` — только при `full=false`; избранные — порядок/пометка) |
| `preset`, `filters`, `metrics` | string | — | пресет, фильтры и показатели экрана — для легенды шапки |
| `include_quotes` | boolean | `false` | приложить листы КП участников (бланки ФКП с их ценами) |
| `columns` | string | все | форма СТ: ключи столбцов через запятую — `number`, `budget_code`, `mark`, `equipment_code`, `manufacturer`, `attrs`, `unit`, `qty`, `ref`, `volume`, `mat_rate`, `works_rate`, `rate`, `mat`, `works`, `cost`, `note` (наименование есть всегда) |

**Ответ `200`:** тело `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `Content-Disposition: attachment; filename="<id тендера>_sravnenie_<ГГГГ-ММ-ДД_ЧЧММ>.xlsx"` (время московское). Книга собирается вне цикла событий.

**Ошибки.** 403/404 — общие; 429.

**Пример.** `GET /tenders/100/comparison/export?include_quotes=true&columns=number,unit,qty,mat_rate,works_rate,cost` → `200` файл `100_sravnenie_2026-10-15_1130.xlsx`

**Побочные эффекты.** Журнал ИБ `report_export` (`report: comparison.xlsx`, номер тендера, `full`, число участников и позиций, скрытые строки, победитель, пресет, `include_quotes`, `contacts_masked`).

---

## Протокол комиссии

Документ собирается из **сохранённого черновика** (реквизиты, которых нет в системе) и **данных системы** (бюджет, суммы, победитель с полным ИНН, приглашённые и подавшие). Умолчания в хранилище не пишутся: пустое поле подставляется при сборке (место и председатель — из настроек `PROTOCOL_PLACE`, `PROTOCOL_CHAIRMAN`; тема — «Обсуждение результатов тендера: <название>»). Структура документа — [tenders-documents.md §3](../02-business-logic/tenders-documents.md#3-протокол-тендера-docx).

Объект `ProtocolDraft` (лишние поля запрещены — 422):

| Поле | Тип | Ограничения | Смысл |
|---|---|---|---|
| `meeting_number` | string | ≤ 50 | номер заседания |
| `meeting_date` | date \| null | `ГГГГ-ММ-ДД` | дата |
| `place` | string | ≤ 300 | место (пусто — из настроек) |
| `topic` | string | ≤ 4000 | тема (пусто — по названию тендера) |
| `chairman`, `chairman_signature` | string | ≤ 200 | председатель и подпись в шапке |
| `voters` | string[] | ≤ 30 строк по 200 | члены комиссии |
| `secretary` | string | ≤ 200 | секретарь |
| `quorum` | string | ≤ 100 | кворум |
| `budget_lines` | `{label ≤ 100, amount ≥ 0, ≤ 15 цифр, 2 знака}[]` | ≤ 20 | статьи бюджета «код — сумма» |
| `attachments` | string[] \| null | ≤ 20 по 300 | приложения; `null` — по умолчанию («Сравнительная таблица №1…» и КП победителя), `[]` — нет |
| `presenter` | string | ≤ 200 | докладчик по сгенерированным пунктам |
| `invited_text`, `received_text`, `comparison_text` | string \| null | ≤ 4000 | переопределения сгенерированных пунктов; `null` — текст из данных системы |
| `questions` | `{text ≤ 4000, speaker ≤ 200}[]` | ≤ 30 | свои пункты вопросов |
| `advance`, `work_term` | string | ≤ 300 | аванс и срок работ решения №1 (пусто — прочерк) |
| `decision_text` | string \| null | ≤ 4000 | переопределение решения №1 |
| `decisions` | `{text ≤ 4000, executor ≤ 200}[]` \| null | ≤ 20 | прочие решения; `null` — «Подготовить пакет документации…», `[]` — нет |

Весь документ ≤ 64 КБ JSON — иначе 422 «Протокол больше 64 КБ».

### GET /tenders/{tender_id}/protocol

**Назначение.** Данные окна «Протокол»: сохранённый черновик, сгенерированные пункты и факты системы.

**Доступ.** Сотрудник + объект + «Цены»; лимит `heavy-read`.

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `saved` | boolean | есть ли сохранённый черновик |
| `updated_at` | ISO \| null | |
| `draft` | `ProtocolDraft` | сохранённое, слитое с предзаполнением (тема, приложения, решения по умолчанию) |
| `generated` | object | `invited_text`, `received_text`, `comparison_text`, `decision_text` — тексты из данных системы |
| `facts` | object | `budget_total` (плановая сумма), `contest_amount` («Конкурс» — сумма победителя), `delta` («Дельта» к бюджету), `contract_amount`, `contract_vat`, `vat_rate`, `winner` (`{name, inn (полный)}` \| null), `invited[]`, `received[]`, `invited_count`, `received_count` |

**Ошибки.** 403 «Протокол содержит суммы договора: нужно право видеть цены»; общие; 429.

**Пример.** → `{"saved": false, "updated_at": null, "draft": {"meeting_number": "", "topic": "Обсуждение результатов тендера: Витражи, дворовой фасад", "attachments": ["Сравнительная таблица №1 (прил. № 1)"], "decisions": [{"text": "Подготовить пакет документации…", "executor": ""}], …}, "generated": {"received_text": "Получены коммерческие предложения от 7 (семи) компаний…"}, "facts": {"budget_total": 269100000.0, "contest_amount": null, "vat_rate": 22, "winner": null, "invited_count": 7, "received_count": 7}}`

**Побочные эффекты.** Нет.

---

### PUT /tenders/{tender_id}/protocol

**Назначение.** Сохранить реквизиты протокола одним JSON за тендером (перезапись). Права «Цены» не требует: ответ не содержит сумм.

**Доступ.** Право `tenders.manage` + объект.

**Тело запроса:** `ProtocolDraft`.

**Ответ `200`:** `{"saved": true, "updated_at": ISO, "draft": ProtocolDraft (как сохранён)}`.

**Ошибки.** 422 — схема (лишнее поле, длины, > 64 КБ); 403/404 — общие.

**Пример.** `{"meeting_number": "12", "meeting_date": "2026-10-20", "voters": ["Иванов И. И.", "Петрова А. А."], "secretary": "Сидорова Е. В.", "advance": "30%", "work_term": "4 месяца"}` → `{"saved": true, "updated_at": "2026-10-15T09:00:00+00:00", "draft": {…}}`

**Побочные эффекты.** Строка протокола (автор, время правки).

---

### GET /tenders/{tender_id}/protocol/export

**Назначение.** Протокол заседания конкурсной комиссии в `.docx` по форме эталона: сохранённый черновик + данные системы + умолчания линии.

**Доступ.** Сотрудник + объект + «Цены»; лимит `heavy-read`.

**Ответ `200`:** `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `Content-Disposition: attachment; filename="<id тендера>_protokol_<ГГГГ-ММ-ДД_ЧЧММ>.docx"` (МСК).

**Ошибки.** 403 «Протокол содержит суммы договора: нужно право видеть цены»; общие; 429.

**Пример.** `GET /tenders/100/protocol/export` → `200` файл `100_protokol_2026-10-15_1200.docx`

**Побочные эффекты.** Журнал ИБ `report_export` (`report: protocol.docx`, номер тендера, победитель, был ли сохранён черновик, `winner_inn_full: true`).

---

## Версии КП и решения по корректировкам

### GET /tenders/{tender_id}/quotes/{quote_id}/versions

**Назначение.** История версий КП участника: кто подал, статус КП в момент версии, что изменилось к предыдущей, файлы подачи.

**Доступ.** Сотрудник + объект (проверка через КП).

**Ответ `200`:** `{"items": [...]}`, новые сверху:

| Поле | Тип | Смысл |
|---|---|---|
| `version_number`, `is_current`, `submitted_at` | | |
| `author`, `author_id` | string / integer \| null | кто подал (у старых подач — автор КП или название контрагента) |
| `supplier_name` | string | |
| `status` | string | статус КП для этой версии (`superseded` у прошлых) |
| `diff` | object | `{is_first, counts: {added, changed, removed}, total, items: [{kind ("added"/"changed"/"removed"), name, fields: [{field, label, from, to}]}]}` — поля: `materials_price` «материалы», `works_price` «работы», `quantity_override` «объём», `comment` «комментарий» и др. |
| `files` | array | файлы, пришедшие с подачей |

**Ошибки.** См. «Доступ — общее».

**Пример.** → `{"items": [{"version_number": 2, "is_current": true, "submitted_at": "2026-09-25T06:55:00Z", "author": "Орлова Мария Сергеевна", "author_id": 3, "supplier_name": "ООО «Альфа Фасад»", "status": "submitted", "diff": {"is_first": false, "counts": {"added": 0, "changed": 206, "removed": 0}, "total": 206, "items": [{"kind": "changed", "name": "Профильная система", "fields": [{"field": "materials_price", "label": "материалы", "from": 16000.0, "to": 15520.0}]}]}, "files": []}]}`

**Побочные эффекты.** Нет.

---

### GET /tenders/{tender_id}/quotes/{quote_id}/versions/{version_number}

**Назначение.** Одна версия КП целиком: строки по позициям, свои позиции, ответы на условия, итоги.

**Доступ.** Сотрудник + объект.

**Параметры пути:** `tender_id`, `quote_id`, `version_number` — integer.

**Ответ `200`:** `{"item": {version_number, is_current, submitted_at, created_by, items: {tender_item_id: {quote_item_id, materials, works, total, materials_unit, works_unit, unit_total, quantity, declined, comment, quantity_override, review_status, review_comment, awaiting_supplier_volume, refill_required, negotiation_potential, awaiting_reply}}, new_items[], condition_values: {id: текст}, totals: {materials, works, total}}}`. В БД цены хранятся за единицу; `materials`/`works`/`total` — по строке (цена × объём).

**Ошибки.** 404 «Версия не найдена»; см. «Доступ — общее».

**Пример.** → `{"item": {"version_number": 1, "is_current": false, "submitted_at": "2026-09-05T08:20:00Z", "items": {"2504": {"materials_unit": 16000.0, "works_unit": 17500.0, "quantity": 14.85, "total": 497475.0, "declined": false}}, "new_items": [], "condition_values": {"128": "0.45"}, "totals": {"materials": 142278238.93, "works": 114432934.94, "total": 256711173.87}}}`

**Побочные эффекты.** Нет.

---

### POST /tenders/{tender_id}/quote-items/{item_id}/review

**Назначение.** Решение специалиста по одной корректировке строки КП: **своя позиция** поставщика (принять/отклонить) или **изменение объёма** (принять — объём позиции тендера меняется для всех; отклонить — строка автора переоткрывается на дозаполнение). Правила — [tenders-quotes.md §6](../02-business-logic/tenders-quotes.md#6-корректировки-отклонения-поставщика-от-формы).

**Доступ.** Сотрудник + объект тендера.

**Параметры пути:** `tender_id`; `item_id` — id строки КП.

**Тело запроса:**

| Поле | Тип | Обяз. | Правила |
|---|---|---|---|
| `review_status` | string | да | `accepted` или `rejected` |
| `comment` (или `review_comment`) | string | при `rejected` | причина (уходит поставщику) |

**Ответ `200`:**

- своя позиция: `{"item": {id, review_status, review_comment}, "refill_item_ids": [], "quote_status": string}`;
- объём: `{"item": {id, review_status, review_comment, tender_item_id, quantity (новый объём позиции), volume_applied, refill_item_ids, refill_supplier_ids, results[] …}, "refill_item_ids": integer[], "quote_status": string}`.

**Ошибки.**

| Код | Когда | `detail` |
|---|---|---|
| 400 | неверный статус | «review_status должен быть accepted или rejected» |
| 400 | отклонение без причины | «Укажите причину отклонения» |
| 404 | строки нет / это не корректировка | «Предложенная позиция не найдена» |
| 404 | позиция объёма не из тендера | «Корректировка объёма не найдена» |
| 404 | КП нет | «КП не найдено» |
| 403 | строка из КП другого тендера | «Нет доступа к позиции этого КП» |

**Пример.** `POST /tenders/100/quote-items/31960/review` `{"review_status": "accepted"}` → `{"item": {"id": 31960, "review_status": "accepted", "tender_item_id": 3071, "quantity": 618.792, "volume_applied": true}, "refill_item_ids": [3071], "quote_status": "submitted"}`

**Побочные эффекты.** Ревизия тендера растёт. Объём принят: объём позиции тендера меняется (исходный запоминается, позиция помечается скорректированной), у **остальных** подавших КП (с ценой по этой позиции) затронутая строка переоткрывается на уточнение цены — им уведомление и письмо о пересчёте (одно на пакет; при нескольких решениях — «Решения по корректировкам») и своё событие ленты без данных автора корректировки. Объём отклонён: строка автора переоткрывается на дозаполнение. Поставщику-автору — уведомление «Корректировка принята»/«Корректировка отклонена» и письмо; заметка в переписку по тендеру; событие ленты `correction_decided` (видно поставщику — без чужих адресатов).

---

### POST /tenders/{tender_id}/quotes/{quote_id}/review-batch

**Назначение.** «Принять/отклонить все корректировки объёма у поставщика сразу»: одно решение — одна транзакция, одно письмо поставщику, одна запись ленты. Без `item_ids` берутся все **нерассмотренные** корректировки объёма текущей версии КП (повторный клик не переигрывает принятые).

**Доступ.** Сотрудник + объект.

**Тело запроса:**

| Поле | Тип | Обяз. | Правила |
|---|---|---|---|
| `review_status` | string | да | `accepted` / `rejected` |
| `comment` (или `review_comment`) | string | при `rejected` | причина |
| `item_ids` | integer[] | нет | id строк КП; по умолчанию все нерассмотренные |

**Ответ `200`:** `{"items": [{id, review_status, review_comment, tender_item_id, quantity, volume_applied}], "decided": integer, "refill_item_ids": integer[], "refill_supplier_ids": integer[], "quote_status": string}`.

**Ошибки.** 400 «review_status должен быть accepted или rejected», «Укажите причину отклонения»; 404 «КП не найдено», «Текущая версия КП не найдена», «Нерассмотренных корректировок объёма нет», «Корректировка объёма не найдена»; 403 «Нет доступа к этому КП» (КП другого тендера).

**Пример.** `{"review_status": "accepted"}` → `{"items": [{"id": 31955, "review_status": "accepted", "tender_item_id": 3066, "quantity": 681.72, "volume_applied": true}], "decided": 4, "refill_item_ids": [3066, 3071], "refill_supplier_ids": [202, 203], "quote_status": "submitted"}`

**Побочные эффекты.** Как у одиночного решения, но одним событием ленты `correction_decided` (`review_status: batch`, список решений) и одним письмом автору.

---

### POST /tenders/{tender_id}/quotes/{quote_id}/request-revision

**Назначение.** Вернуть поданное КП на доработку целиком с комментарием: статус КП `revision_requested`, форма у поставщика открывается, повторная подача создаёт новую версию.

**Доступ.** Сотрудник + объект.

**Тело запроса:** `{"comment": string}` — обязателен (непустой).

**Ответ `200`:** `{"item": {"id", "status": "revision_requested", "revision_comment"}}`.

**Ошибки.** 400 «Опишите, что нужно доработать»; см. «Доступ — общее». Статус тендера не проверяется (см. открытые вопросы).

**Пример.** `{"comment": "Уточните цены по разделу 3 — не учтён крепёж"}` → `{"item": {"id": 112, "status": "revision_requested", "revision_comment": "Уточните цены по разделу 3 — не учтён крепёж"}}`

**Побочные эффекты.** Автор и время запроса в КП; приглашение `submitted` → `filling`; событие ленты `revision_requested` (видно поставщику); уведомление в кабинете «Требуется повторное заполнение КП» (`kind: revision`, ссылка в кабинет); письмо на e-mail организации «Запрошена доработка КП — <номер>» с комментарием и ссылкой на форму (синхронно).

---

## Комментарии к позициям

### GET /tenders/{tender_id}/comments

**Назначение.** Вся переписка сотрудников по ячейкам СТ (позиция × участник) одной выборкой — чтобы маркер комментария рисовался у каждой ячейки до первого клика. Список плоский, включая ответы; тред клиент собирает по паре `supplier_id × entity_id`. Ответы контрагента, пришедшие из переписки по тендеру, лежат тут же без автора-пользователя.

**Доступ.** Сотрудник + объект.

**Параметры запроса:** `limit`, `offset` (overview §8).

**Ответ `200`:** конверт; элемент: `id`, `entity_id` (позиция), `supplier_id`, `parent_comment_id`, `author_name` (ФИО; у ответа контрагента — название организации; иначе «Удалённый пользователь»), `comment_text`, `created_at` (`ДД.ММ.ГГГГ ЧЧ:ММ`, МСК), `read` (прочитано мной: отметка или моя запись), `from_supplier` (запись контрагента).

**Ошибки.** Общие.

**Пример.** → `{"items": [{"id": 9268, "entity_id": 2504, "supplier_id": 203, "parent_comment_id": null, "author_name": "Иванов Иван Иванович", "comment_text": "Профиль дороже медианы на 40 % — уточнить систему", "created_at": "19.09.2026 15:30", "read": true, "from_supplier": false}], "total": 1, "limit": 200, "offset": 0}`

**Побочные эффекты.** Нет.

---

### GET /tenders/{tender_id}/position-comments

**Назначение.** История комментариев **поставщиков** к позициям по всем версиям КП (только чтение): по паре «контрагент × позиция» — записи от новой к первой; пустые тексты не становятся записями, подряд идущие одинаковые тексты схлопываются; свои позиции поставщика в историю не входят. Отличается от переписки сотрудников (`/comments`).

**Доступ.** Сотрудник + объект.

**Параметры запроса:** `limit`, `offset` — страница по парам (история одной пары не рвётся).

**Ответ `200`:** конверт; элемент: `{supplier_id, tender_item_id, entries: [{quote_item_id, text, version ("v1"…), author, at (ДД.ММ.ГГГГ ЧЧ:ММ), unread}]}` — `unread`: запись ещё не отмечена мной просмотренной.

**Ошибки.** Общие.

**Пример.** → `{"items": [{"supplier_id": 206, "tender_item_id": 2566, "entries": [{"quote_item_id": 17084, "text": "Объём по файлу КП: 9.5175 (в тендере 4.7588)", "version": "v1", "author": "Соловьёва Евгения Романовна", "at": "23.09.2026 10:05", "unread": true}]}], "total": 56, "limit": 1, "offset": 0}`

**Побочные эффекты.** Нет.

---

### GET /tenders/{tender_id}/comment-workload

**Назначение.** Что у **меня** не отработано по ячейкам тендера — один предикат для знака в ячейке и чипа в шапке колонки участника. «Не отработано» — непросмотренные мной записи истории комментария поставщика плюс непрочитанные ответы контрагента из переписки. Без пагинации: нужны все ячейки сразу.

**Доступ.** Сотрудник + объект.

**Ответ `200`:** `{"cells": [{supplier_id, tender_item_id, total (записей истории), unresolved (не отработано)}], "by_supplier": {"<supplier_id>": число ячеек с неотработанным}}`.

**Ошибки.** Общие.

**Пример.** → `{"cells": [{"supplier_id": 206, "tender_item_id": 2566, "total": 1, "unresolved": 1}], "by_supplier": {"206": 1, "205": 5}}`

**Побочные эффекты.** Нет.

---

### POST /tenders/{tender_id}/position-comments/read

**Назначение.** Персонально отметить записи истории комментария поставщика просмотренными (клиент передаёт записи, показанные в панели). Новая отличающаяся запись снова станет «не отработано».

**Доступ.** Сотрудник + объект.

**Тело запроса:** `{"quote_item_ids": integer[]}`; пустой список — ничего не делается. Чужие (не этого тендера) id молча отбрасываются.

**Ответ `200`:** `{"marked": integer}`.

**Ошибки.** 400 «Недопустимый перечень записей» (нечисловые id); общие.

**Пример.** `{"quote_item_ids": [17084]}` → `{"marked": 1}`

**Побочные эффекты.** Отметки просмотра (пользователь × строка КП).
