# API: бюджет (budget)

Ручки раздела «Бюджет»: план/факт по тендерам, сводка бюджета за год с сигналами и метриками по дереву бюджетных позиций, распределение («выделено») по статьям на год и его правка. CRUD самих бюджетных позиций (`/budget-positions`) — в другом теге и в этот файл не входит. Общие соглашения (аутентификация, ошибки, конверт `{items, total, limit, offset}`, кэш тяжёлых чтений с `ETag`) — в [overview.md](overview.md).

Связанные разделы: бизнес-логика — [../02-business-logic/budget.md](../02-business-logic/budget.md); экраны — [../03-screens/budget.md](../03-screens/budget.md), [../03-screens/budget-position.md](../03-screens/budget-position.md).

## Сводка

| Метод | Путь | Кратко | Доступ |
|---|---|---|---|
| GET | `/budget` | План/факт по тендерам (срез для обзора) | право `budget.view`; тендеры доступных проектов |
| GET | `/budget/overview` | Сводка бюджета за год: итоги, сигналы, метрики позиций | `budget.view`; лимит частоты; кэш с `ETag` |
| GET | `/budget/allocations` | Распределение бюджета по статьям за год | `budget.view` |
| PATCH | `/budget/allocations/{allocation_id}` | Изменить сумму/статус/комментарий распределения | `budget.manage` |

Права по умолчанию: `budget.view` — тендерный специалист и администратор; `budget.manage` — только администратор (настраивается в матрице прав). Отказ — `403` «Недостаточно прав для этого действия — обратитесь к администратору системы» + событие `permission_denied` в журнале ИБ; нет сессии — `401`.

Сложность блока: **M** (расчёт метрик по дереву — L вместе с экранами).

### Как считаются метрики (кратко; полностью — в бизнес-логике)

- **Отчётный год тендера** — год `start_date`, а если её нет — год `created_at`. По умолчанию — текущий год.
- **Выделено** (`allocated`) — единственная хранимая величина: сумма «позиция × год».
- **В работе** (`in_work`) — сумма плановых бюджетов (`budget_amount`) тендеров позиции в статусах `draft`, `open`.
- **Израсходовано** (`spent`) — по тендерам в статусе `closed`: минимальный итог среди поданных текущих версий КП, а если КП нет — плановый бюджет тендера, иначе 0. (Понятия «победитель» в расчёте нет.)
- **Свободно** (`free`) = выделено − в работе − израсходовано.
- Значения с префиксом `branch_` — то же по ветке (позиция + все потомки). Дерево позиций — не глубже 5 уровней.

---

### GET /budget

**Назначение.** План/факт по каждому тендеру (используется как срез на экране обзора бюджета). Факт — минимальный итог среди поданных текущих версий КП тендера.

**Доступ.** `budget.view`. В выборку попадают только тендеры проектов, доступных пользователю (администратор — все).

**Параметры запроса:**

| Параметр | Тип | Обяз. | Смысл |
|---|---|---|---|
| `limit` | integer | нет | 1–200; без него — до 200 строк |
| `offset` | integer | нет | ≥ 0 |

Страница режется по уже собранному отчёту (строка — тендер с агрегатами по его КП). Фильтра по году нет — все тендеры доступных проектов.

**Ответ `200`:** конверт `{items, total, limit, offset}`, элемент:

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | id тендера |
| `number` | string | номер тендера |
| `title` | string | название |
| `status` | string | статус тендера |
| `project_id` / `project_name` | integer / string | объект |
| `portfolio_id` / `portfolio_name` | integer \| null / string | портфель |
| `work_type_name` | string | вид работ (пусто, если не задан) |
| `budget_position_id` / `budget_position_name` | integer \| null / string | бюджетная позиция |
| `plan_amount` | number \| null | плановый бюджет тендера |
| `actual_amount` | number \| null | минимальный итог поданных КП; `null`, если КП нет |
| `deviation` | number \| null | `actual_amount − plan_amount` (отрицательное — экономия) |

**Ошибки.** 401, 403, 422 (`limit`/`offset` вне диапазона).

**Пример.**

```json
{"items": [{"id": 100, "number": "T-2026-100", "title": "Витражи, дворовой фасад", "status": "open",
  "project_id": 24, "project_name": "МКД-1", "portfolio_id": 13, "portfolio_name": "ЖК «Северный парк»",
  "work_type_name": "Витражи и оконные блоки", "budget_position_id": 25, "budget_position_name": "Витражи и оконные блоки",
  "plan_amount": 269100000.0, "actual_amount": 222878133.64, "deviation": -46221866.36}],
 "total": 6, "limit": 2, "offset": 0}
```

**Побочные эффекты.** Нет.

---

### GET /budget/overview

**Назначение.** Сводка бюджета за год: итоги по всем активным позициям, сигналы (перерасход, нулевой бюджет при наличии тендеров, тендеры без бюджетной позиции) и полный список метрик позиций для дерева.

**Доступ.** `budget.view`.

**Лимит частоты.** Общий лимит «тяжёлых чтений» на пользователя: `HEAVY_READ_RATE_LIMIT` запросов (по умолчанию 60) за `HEAVY_READ_RATE_WINDOW` секунд (60). Превышение — `429` «Слишком много запросов — попробуйте немного позже».

**Кэш.** Ответ кэшируется на сервере на `HEAVY_READ_CACHE_SECONDS` (60 с) по ключу «год + набор доступных проектов + версия расчёта цен + отпечаток данных (последние изменения тендеров, КП, журнала распределения, число позиций)». Ответ несёт `ETag` (слабый) и `Cache-Control: private, no-cache`; при совпадении `If-None-Match` — `304` без тела.

**Параметры запроса:**

| Параметр | Тип | Обяз. | Смысл |
|---|---|---|---|
| `year` | integer | нет | отчётный год; по умолчанию текущий |

**Ответ `200`:**

| Поле | Тип | Смысл |
|---|---|---|
| `period_year` | integer | год |
| `totals` | object | `{allocated, in_work, spent, free}` — суммы по активным позициям (собственные значения, не ветки) |
| `signals.overrun` | array | позиции с `free < 0`: `{id, path, free, allocated}` |
| `signals.zero_budget_with_tenders` | array | позиции без выделенного бюджета, но с тендерами: `{id, path, tenders_total}` |
| `signals.tenders_without_position` | array | тендеры года без бюджетной позиции (только доступных проектов): `{id, number, title, status, project_id}` |
| `positions` | array | метрики каждой активной позиции (см. ниже), порядок — по id |

Элемент `positions`:

| Поле | Тип | Смысл |
|---|---|---|
| `id`, `name`, `code`, `parent_id`, `is_active` | | реквизиты позиции |
| `path` | string | путь от корня через « › » |
| `depth` | integer | глубина (0 — корень) |
| `period_year` | integer | год |
| `allocation_id` / `allocation_status` / `allocation_comment` | integer \| null / `open`\|`closed`\|null / string \| null | распределение на год |
| `allocated`, `in_work`, `spent`, `free` | number | собственные метрики |
| `has_funds` | boolean | `free > 0` |
| `branch_allocated`, `branch_in_work`, `branch_spent`, `branch_free` | number | метрики ветки |
| `tenders_total`, `tenders_open`, `tenders_closed` | integer | тендеры позиции за год (открытые — `draft`/`open`, закрытые — `closed`) |
| `branch_tenders_total` | integer | тендеры ветки |

Метрики позиций считаются по всем тендерам года с бюджетной позицией, без ограничения доступными проектами; ограничение применяется только к сигналу `tenders_without_position` (см. открытые вопросы).

**Ошибки.** 401, 403, 429.

**Пример (сокращён).**

```json
{"period_year": 2026,
 "totals": {"allocated": 1149000000.0, "in_work": 793500000.0, "spent": 150031118.06, "free": 205468881.94},
 "signals": {"overrun": [], "zero_budget_with_tenders": [], "tenders_without_position": []},
 "positions": [{"id": 18, "name": "Школа №1", "code": "SCH-01", "parent_id": null, "is_active": true, "path": "Школа №1",
   "depth": 0, "period_year": 2026, "allocation_id": 39, "allocation_status": "open", "allocation_comment": "Лимит 2026",
   "allocated": 8000000.0, "in_work": 0.0, "spent": 0.0, "free": 8000000.0, "has_funds": true,
   "branch_allocated": 125000000.0, "branch_in_work": 105100000.0, "branch_spent": 0.0, "branch_free": 19900000.0,
   "tenders_total": 0, "tenders_open": 0, "tenders_closed": 0, "branch_tenders_total": 3}]}
```

**Побочные эффекты.** Запись в серверный кэш тяжёлых чтений.

---

### GET /budget/allocations

**Назначение.** Распределение бюджета по статьям за год с метриками использования; отдельно — активные статьи, которым на этот год бюджет ещё не назначен.

**Доступ.** `budget.view`. Фильтра по доступным проектам нет: распределение привязано к статье и году, а не к проекту.

**Параметры запроса:**

| Параметр | Тип | Обяз. | Смысл |
|---|---|---|---|
| `year` | integer | нет | год; по умолчанию текущий |
| `limit` | integer | нет | 1–200 (страница режется по `items`) |
| `offset` | integer | нет | ≥ 0 |

**Ответ `200`:** конверт `{items, total, limit, offset}` + `period_year` + `without_allocation`.

Элемент `items`:

| Поле | Тип | Смысл |
|---|---|---|
| `id` | integer | id распределения |
| `budget_position_id` | integer | статья |
| `parent_id` | integer \| null | родитель статьи (форма назначения считает «сколько можно распределить» по свободному остатку родителя) |
| `period_year` | integer | год |
| `allocated_amount` | number | выделено |
| `status` | string | `open` \| `closed` |
| `comment` | string \| null | комментарий |
| `path` | string | путь статьи |
| `code` | string \| null | код статьи |
| `in_work`, `spent`, `free` | number | метрики (как в сводке) |
| `is_active` | boolean | статья активна |

Элемент `without_allocation` (идёт целиком, вне пагинации): `{budget_position_id, path, code, parent_id, period_year, allocated_amount: 0, status: null, free: 0, in_work, spent}`.

**Ошибки.** 401, 403, 422.

**Пример.**

```json
{"items": [{"id": 38, "budget_position_id": 17, "parent_id": 36, "period_year": 2026, "allocated_amount": 12000000.0,
  "status": "open", "comment": null, "path": "ЖК «Северный парк» › Корпус ОКН", "code": "NP-OKN",
  "in_work": 0.0, "spent": 0.0, "free": 12000000.0, "is_active": true}],
 "total": 15, "limit": 1, "offset": 0, "period_year": 2026, "without_allocation": []}
```

**Побочные эффекты.** Нет.

---

### PATCH /budget/allocations/{allocation_id}

**Назначение.** Изменить выделенную сумму, статус периода (открыт/закрыт) или комментарий распределения. Изменение суммы пишется в историю.

**Доступ.** `budget.manage`.

**Параметры пути:**

| Параметр | Тип | Смысл |
|---|---|---|
| `allocation_id` | integer | id распределения «статья × год» |

**Тело запроса** (`application/json`, частичное; без строгой схемы):

| Поле | Тип | Обяз. | Ограничения / смысл |
|---|---|---|---|
| `allocated_amount` | number | нет | ≥ 0; у закрытого периода менять нельзя, если в том же запросе период не открывается (`status: "open"`) |
| `status` | string | нет | `open` \| `closed` |
| `comment` | string \| null | нет | комментарий распределения; пустое → `null`. Он же — комментарий записи истории при смене суммы (по умолчанию «Изменение суммы») |

**Ответ `200`:** `{"item": {id, budget_position_id, period_year, allocated_amount, status, comment}}`.

**Ошибки:**

| Код | Когда | `detail` |
|---|---|---|
| 404 | распределения нет | «Бюджет периода не найден» |
| 400 | смена суммы у закрытого периода | «Закрытый период нельзя менять» |
| 400 | сумма < 0 | «Сумма не может быть отрицательной» |
| 400 | статус не `open`/`closed` | «Статус: open или closed» |
| 401 / 403 | см. общее | |

Нечисловое `allocated_amount` сервером не валидируется отдельно (см. открытые вопросы).

**Пример.**

```json
{"allocated_amount": 15000000, "comment": "Перенос с резерва по решению комитета"}
```

```json
{"item": {"id": 38, "budget_position_id": 17, "period_year": 2026, "allocated_amount": 15000000.0,
          "status": "open", "comment": "Перенос с резерва по решению комитета"}}
```

**Побочные эффекты.** При изменении суммы — запись в историю распределения (`old_amount`, `new_amount`, комментарий, кто изменил); это же меняет отпечаток данных сводки, и кэш `GET /budget/overview` инвалидируется. В журнал ИБ событие не пишется.
