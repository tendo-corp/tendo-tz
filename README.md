# Техническое задание: платформа тендерных закупок Tendo

Полное ТЗ на воссоздание с нуля веб-платформы закупок работ у подрядчиков. По этому ТЗ
исполнитель должен понять, зачем нужен продукт, нарисовать каждый экран, реализовать каждую
ручку API, поднять демо-стенд и оценить стоимость и сроки работ. Доступ к исходному коду
для этого не нужен.

| | |
|---|---|
| **Версия ТЗ** | 1.0 от 15.10.2026 |
| **Основа** | работающая система (код и демо-стенд с обезличенными данными) по состоянию на 15.10.2026 |
| **Язык** | русский; имена полей, ручек и таблиц — как в системе |
| **Объём** | 18 доменов бизнес-логики · 333 операции HTTP API на 270 путях · 82 таблицы БД · 41 маршрут интерфейса · скриншоты экранов на реальных обезличенных данных |

## Как читать

| Кто читает | С чего начать | Затем |
|---|---|---|
| Оценщик, руководитель проекта | [01-product/vision.md](01-product/vision.md) → [01-product/scope-and-mvp.md](01-product/scope-and-mvp.md) → [09-estimation/work-breakdown.md](09-estimation/work-breakdown.md) → [09-estimation/complexity-map.md](09-estimation/complexity-map.md) | [99-open-questions.md](99-open-questions.md): вопросы с пометкой «влияет на оценку» закрыть до фиксации цены |
| Аналитик, тестировщик | [01-product/glossary.md](01-product/glossary.md) → [01-product/users-and-roles.md](01-product/users-and-roles.md) → [02-business-logic/end-to-end-scenarios.md](02-business-logic/end-to-end-scenarios.md) | доменные файлы [02-business-logic](02-business-logic/tenders.md) |
| Бэкенд-разработчик | [07-architecture/architecture.md](07-architecture/architecture.md) → [04-api/overview.md](04-api/overview.md) → [05-data-model/erd.md](05-data-model/erd.md) | доменные файлы 02, ручки 04, таблицы 05, ИИ 06, безопасность 07 |
| Фронтенд-разработчик | [03-screens/screen-map.md](03-screens/screen-map.md) → [03-screens/shell.md](03-screens/shell.md) → экраны | правила из 02, ручки из 04 |
| DevOps | [08-deploy/demo-deploy.md](08-deploy/demo-deploy.md) → [08-deploy/environments.md](08-deploy/environments.md) → [08-deploy/ci-cd.md](08-deploy/ci-cd.md) | [07-architecture/scalability.md](07-architecture/scalability.md), [07-architecture/nonfunctional.md](07-architecture/nonfunctional.md) |

## Состав

```
tz/
├── README.md                  этот файл
├── 01-product/                что за продукт
│   ├── vision.md              проблема, пользователи, ценность, принципы
│   ├── users-and-roles.md     7 ролей, устройство доступа, матрица «роль × действие»
│   ├── glossary.md            словарь терминов
│   └── scope-and-mvp.md       приоритеты (MoSCoW), MVP, что не воспроизводить
├── 02-business-logic/         правила по доменам: сущности, статусы и переходы,
│                              валидации, формулы, права, события, фоновые задачи, примеры
├── 03-screens/                карта маршрутов и описание каждого экрана со скриншотами
│   └── assets/                скриншоты (PNG, 1440 px) с нумерованными метками
├── 04-api/                    общие соглашения, все ручки по группам, openapi.json как есть
├── 05-data-model/             ER-диаграммы и справочник всех таблиц
├── 06-ai/                     ИИ-функции, проверка чисел, деградация, учёт стоимости
├── 07-architecture/           архитектура, масштабирование, безопасность, НФТ
├── 08-deploy/                 окружения, демо-деплой с нуля, CI/CD
├── 09-estimation/             этапы и критерии приёмки, карта сложности, стек
└── 99-open-questions.md       открытые вопросы к Заказчику
```

### 02 — бизнес-логика

| Файл | Домен |
|---|---|
| [identity](02-business-logic/identity.md) | вход, сессии, MFA, пользователи, роли и права, лиды |
| [audit](02-business-logic/audit.md) | журнал действий и каталог событий |
| [billing](02-business-logic/billing.md) | биллинг ИИ-токенов, кабинет владельца площадки |
| [catalog](02-business-logic/catalog.md) | справочники: виды работ, единицы, бюджетные позиции, статусы |
| [portfolios](02-business-logic/portfolios.md) | портфели, проекты, подпроекты, доступ к ним |
| [notifications](02-business-logic/notifications.md) | уведомления в интерфейсе и по почте |
| [files](02-business-logic/files.md) | вложения, S3 и presigned-загрузка |
| [suppliers](02-business-logic/suppliers.md) | контрагенты, анкета, квалификация, кабинет поставщика |
| [fkp-import](02-business-logic/fkp-import.md) | распознавание Excel-сметы в ФКП, загрузка КП файлом |
| [tenders](02-business-logic/tenders.md) | индекс домена тендеров и статусная модель |
| ↳ [tenders-lifecycle](02-business-logic/tenders-lifecycle.md) | создание, статусы, победитель, версии ФКП |
| ↳ [tenders-invitations](02-business-logic/tenders-invitations.md) | участники, персональные ссылки, стадии |
| ↳ [tenders-quotes](02-business-logic/tenders-quotes.md) | КП, версии, корректировки, НДС, подача |
| ↳ [tenders-comparison](02-business-logic/tenders-comparison.md) | сравнительная таблица и все формулы |
| ↳ [tenders-rounds](02-business-logic/tenders-rounds.md) | раунды переторжки |
| ↳ [tenders-documents](02-business-logic/tenders-documents.md) | документы, протокол DOCX, выгрузки |
| [budget](02-business-logic/budget.md) | бюджет объектов |
| [analytics](02-business-logic/analytics.md) | отчёты и выгрузки |
| [insights](02-business-logic/insights.md) | ИИ-анализ тендера |
| [comments](02-business-logic/comments.md) | комментарии к позициям |
| [chat](02-business-logic/chat.md) | чат специалиста и поставщика |
| [security](02-business-logic/security.md) | журнал ИБ, инциденты, выгрузки для регулятора |
| [settings](02-business-logic/settings.md) | настройки площадки, почта, шаблоны писем |
| [bug-reports](02-business-logic/bug-reports.md) | обращения об ошибках |
| [end-to-end-scenarios](02-business-logic/end-to-end-scenarios.md) | **сквозные сценарии с настоящими числами** — основа приёмки |

### 04 — API

Общие соглашения (аутентификация на cookie, ошибки, пагинация, лимиты, SSE) —
[04-api/overview.md](04-api/overview.md). Там же карта всех файлов ручек с числом операций.
Каждая операция описана отдельным заголовком `### МЕТОД /путь`. Схема OpenAPI в том виде,
в каком её отдаёт работающий бэкенд, — [04-api/openapi.json](04-api/openapi.json).

## Соглашения

- **«Система должна…»** — обязательное требование. Пронумерованные требования
  (`SEC-12`, `ARC-30`, `AI-22`, `DM-05` и т. п.) можно цитировать в договоре и при приёмке.
- **S / M / L / XL** — относительная сложность (шкала — в
  [09-estimation/complexity-map.md](09-estimation/complexity-map.md)). Часов и денег в ТЗ нет.
- **«По коду» / «расхождение»** — ТЗ описывает поведение работающей системы. Где оно
  расходится с документацией или само по себе ошибочно, в тексте есть требование к правильному
  поведению, а вопрос вынесен в [99-open-questions.md](99-open-questions.md).
- **Имена переменных окружения** (`AI_API_KEY`, `LEADS_ENABLED`…) даны без значений.
  Значения задаются при развёртывании.
- **Данные на скриншотах и в примерах** — обезличенные демо-данные: подрядчики «Альфа Фасад» …
  «Каппа Ремонт», объекты «ЖК «Северный парк»», «ЖК «Лесной»», «Школа №1», e-mail вида
  `user@example.com`. В углу части снимков карточки тендера видна кнопка отладочной панели
  dev-сборки — в продукт она не входит.
- **Флаги функций.** Часть экранов и ручек видна только при включённом флаге (демо-инструменты,
  заявки с лендинга, биллинг ИИ, передача в SIEM). У таких элементов флаг указан явно.

## Чего нет в ТЗ

- Маркетинговый лендинг и любые сборки под конкретных клиентов.
- Исходный код. Примеры JSON, схемы таблиц и формулы приведены ровно в том объёме, который
  нужен для реализации.
- Часы и стоимость: их определяет исполнитель по [09-estimation](09-estimation/work-breakdown.md).
