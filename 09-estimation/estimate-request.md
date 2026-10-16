# Запрос оценки для исполнителя

Оцените **результат**: по каждому этапу назовите фиксированную цену и срок, за которые вы сдадите
этап по его критериям приёмки. Как организовать работу внутри, решаете вы — часы и раскладка по
специалистам нам не нужны.

## С чего начать

1. [Видение продукта](../01-product/vision.md) — что за система и зачем.
2. [Объём и MVP](../01-product/scope-and-mvp.md) — что обязательно, что нет.
3. Этот файл — этапы, которые нужно оценить. У каждого этапа ниже перечислены файлы ТЗ,
   где описан его состав: правила, экраны, API.
4. [Сквозные сценарии](../02-business-logic/end-to-end-scenarios.md) — как система работает
   целиком; по ним принимается каждый этап.
5. [Модель данных](../05-data-model/erd.md) и [таблицы](../05-data-model/tables.md) — общие для
   всех этапов.

## Как сдать оценку

1. Скачайте шаблон [estimate-template.xlsx](estimate-template.xlsx) (на странице файла —
   кнопка «Download raw file»).
2. Заполните жёлтые ячейки. Этапы, их порядок и столбцы не меняйте: так мы сравниваем оценки
   разных исполнителей между собой.
3. Загрузите файл на Google Диск, назовите его `Оценка Tendo — <ваше имя или компания>`.
4. Откройте доступ «Редактор» или «Комментатор» для **mtertishniy@gmail.com** и пришлите
   ссылку тому, от кого получили ТЗ.

Файл в формате `.xlsx`; если загрузили на Диск как Google Таблицу — тоже подходит.

## Что входит в цену любого этапа

1. Всё, что перечислено в составе этапа, включая автотесты.
2. Выкатка этапа на демо-стенд.
3. Общие правила приёмки — [декомпозиция работ, §0](work-breakdown.md#0-общие-правила-приёмки-для-всех-этапов).
4. Сквозные сценарии, реализованные к этому моменту, проходят на демо-стенде.
5. На момент сдачи нет открытых дефектов критичности «блокер» и «высокая».

Отдельного этапа «стабилизация» нет: пункты 4–5 входят в приёмку каждого этапа.

## Этапы

Порядок и зависимости этапов — [декомпозиция работ, §1](work-breakdown.md#1-сводка-этапов).
Значения с пометкой «предварительно» уточним до фиксации цены. Если ваша цена сильно зависит
от такого значения, укажите в столбце «Вопросы и риски», при каком значении она меняется.

### П. Платформа и стенд (этапы 0 и 12 из декомпозиции)

**Результат, который принимаем:** Локальный стенд поднимается одной командой; демо-стенд поднимается с нуля по инструкции; CI собирает образы; восстановление из бэкапа проверено; откат на предыдущую версию занимает минуты; фоновая задача-пример переживает рестарт; тестовое событие realtime доходит до браузера; архитектурный тест падает при запрещённом импорте

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 0](work-breakdown.md#этап-0-каркас--l), [Этап 12](work-breakdown.md#этап-12-деплой-и-демо--m-сквозной) |
| Экраны | [shell](../03-screens/shell.md) |
| API | [overview](../04-api/overview.md), [endpoints-system](../04-api/endpoints-system.md) |
| Ещё | [architecture](../07-architecture/architecture.md), [nonfunctional](../07-architecture/nonfunctional.md), [environments](../08-deploy/environments.md), [demo-deploy](../08-deploy/demo-deploy.md), [ci-cd](../08-deploy/ci-cd.md) |

### 1. Вход и доступ

**Результат, который принимаем:** Вход, сессии, MFA, пользователи и роли; требования SEC-* по аутентификации покрыты тестами; автотест проверяет права каждой роли на каждую ручку; выход действует сразу на всех вкладках

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 1](work-breakdown.md#этап-1-вход-и-доступ--l) |
| Бизнес-логика | [identity](../02-business-logic/identity.md), [audit](../02-business-logic/audit.md) |
| Экраны | [login](../03-screens/login.md), [reset-password](../03-screens/reset-password.md), [profile](../03-screens/profile.md) |
| API | [endpoints-identity](../04-api/endpoints-identity.md) |
| Ещё | [users-and-roles](../01-product/users-and-roles.md), [security](../07-architecture/security.md) |

### 2. Справочники, портфели и объекты

**Результат, который принимаем:** CRUD с мягким удалением; защита от циклов дерева; доступ к объекту ограничивает видимость во всех списках

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 2](work-breakdown.md#этап-2-справочники-портфели-и-объекты--m) |
| Бизнес-логика | [catalog](../02-business-logic/catalog.md), [portfolios](../02-business-logic/portfolios.md) |
| Экраны | [portfolios](../03-screens/portfolios.md), [object-card](../03-screens/object-card.md), [catalog-work-types](../03-screens/catalog-work-types.md), [catalog-units](../03-screens/catalog-units.md), [catalog-statuses](../03-screens/catalog-statuses.md), [catalog-budget-positions](../03-screens/catalog-budget-positions.md) |
| API | [endpoints-catalog](../04-api/endpoints-catalog.md), [endpoints-portfolios](../04-api/endpoints-portfolios.md) |

### 3. Контрагенты

**Результат, который принимаем:** Реестр, карточка, анкета, квалификация, кабинет поставщика; проверки ИНН/КПП/ОГРН/телефона/e-mail одинаковы на фронте и сервере; ПДн контактов в БД зашифрованы

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 3](work-breakdown.md#этап-3-контрагенты--l) |
| Бизнес-логика | [suppliers](../02-business-logic/suppliers.md) |
| Экраны | [suppliers](../03-screens/suppliers.md), [supplier-card](../03-screens/supplier-card.md), [register](../03-screens/register.md), [questionnaires](../03-screens/questionnaires.md), [cabinet](../03-screens/cabinet.md) |
| API | [endpoints-suppliers](../04-api/endpoints-suppliers.md), [endpoints-suppliers-registration](../04-api/endpoints-suppliers-registration.md) |

### 4. Тендер и ФКП

**Результат, который принимаем:** Тендер создаётся мастером по шагам; **не менее 80 % (предварительно)** из 20–30 реальных смет Заказчика распознаются без ручной правки структуры, остальные правятся в предпросмотре без повторной загрузки файла

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 4](work-breakdown.md#этап-4-тендер-и-фкп--xl) |
| Бизнес-логика | [tenders](../02-business-logic/tenders.md), [fkp-import](../02-business-logic/fkp-import.md), [tenders-lifecycle](../02-business-logic/tenders-lifecycle.md), [tenders-documents](../02-business-logic/tenders-documents.md), [files](../02-business-logic/files.md) |
| Экраны | [tenders-registry](../03-screens/tenders-registry.md), [tender-create](../03-screens/tender-create.md), [tender-card](../03-screens/tender-card.md), [tender-card-overview](../03-screens/tender-card-overview.md), [tender-card-structure](../03-screens/tender-card-structure.md), [tender-card-documents](../03-screens/tender-card-documents.md), [templates](../03-screens/templates.md) |
| API | [endpoints-tenders](../04-api/endpoints-tenders.md), [endpoints-tenders-import](../04-api/endpoints-tenders-import.md), [endpoints-tenders-templates](../04-api/endpoints-tenders-templates.md), [endpoints-files](../04-api/endpoints-files.md) |

### 5. Приглашения и КП

**Результат, который принимаем:** Поставщик подаёт КП по персональной ссылке; ни через интерфейс, ни через API не видит чужих КП, цен и участников (автотест); файл КП с изменённой формой отклоняется со списком ошибок

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 5](work-breakdown.md#этап-5-приглашения-и-кп--xl) |
| Бизнес-логика | [tenders-invitations](../02-business-logic/tenders-invitations.md), [tenders-quotes](../02-business-logic/tenders-quotes.md) |
| Экраны | [tender-card-participants](../03-screens/tender-card-participants.md), [fill](../03-screens/fill.md), [quotes-inbox](../03-screens/quotes-inbox.md), [quote-card](../03-screens/quote-card.md) |
| API | [endpoints-tenders-invite](../04-api/endpoints-tenders-invite.md), [endpoints-quotes](../04-api/endpoints-quotes.md) |

### 6. Сравнение, раунды, итог

**Результат, который принимаем:** Числа сценария 1 совпадают до копейки в сравнительной таблице, выгрузке Excel и протоколе; **таблица 300 позиций × 10 поставщиков открывается не дольше 3 с (предварительно)** и прокручивается без задержек

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 6](work-breakdown.md#этап-6-сравнение-раунды-итог--xl) |
| Бизнес-логика | [tenders-comparison](../02-business-logic/tenders-comparison.md), [tenders-rounds](../02-business-logic/tenders-rounds.md) |
| Экраны | [tender-card-comparison](../03-screens/tender-card-comparison.md), [tender-card-rounds](../03-screens/tender-card-rounds.md), [tender-card-lifecycle](../03-screens/tender-card-lifecycle.md) |
| API | [endpoints-tenders-comparison](../04-api/endpoints-tenders-comparison.md), [endpoints-rounds](../04-api/endpoints-rounds.md) |
| Ещё | [end-to-end-scenarios](../02-business-logic/end-to-end-scenarios.md) |

### 7. Уведомления, комментарии, чат

**Результат, который принимаем:** Событие доходит до открытой вкладки за секунды; после обрыва связи пропущенные сообщения чата догружаются; письмо уходит в фоне и не задерживает действие

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 7](work-breakdown.md#этап-7-уведомления-комментарии-чат--l) |
| Бизнес-логика | [notifications](../02-business-logic/notifications.md), [comments](../02-business-logic/comments.md), [chat](../02-business-logic/chat.md) |
| Экраны | [notifications](../03-screens/notifications.md), [chats](../03-screens/chats.md), [tender-card-messages](../03-screens/tender-card-messages.md), [tender-card-activity](../03-screens/tender-card-activity.md) |
| API | [endpoints-notifications](../04-api/endpoints-notifications.md), [endpoints-comments](../04-api/endpoints-comments.md), [endpoints-chat](../04-api/endpoints-chat.md) |

### 8. Бюджет, отчёты, главная

**Результат, который принимаем:** Суммы отчётов совпадают со сравнительной таблицей; без права на финансы суммы не видны

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 8](work-breakdown.md#этап-8-бюджет-отчёты-главная--m) |
| Бизнес-логика | [budget](../02-business-logic/budget.md), [analytics](../02-business-logic/analytics.md) |
| Экраны | [home](../03-screens/home.md), [budget](../03-screens/budget.md), [budget-position](../03-screens/budget-position.md), [reports](../03-screens/reports.md) |
| API | [endpoints-budget](../04-api/endpoints-budget.md), [endpoints-reports](../04-api/endpoints-reports.md) |

### 9. Безопасность и администрирование

**Результат, который принимаем:** Запись журнала ИБ нельзя изменить или удалить даже из БД без отключения триггера; инцидент создаётся по правилу автоматически; настройки площадки и обращения об ошибках работают

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 9](work-breakdown.md#этап-9-безопасность-и-администрирование--l) |
| Бизнес-логика | [security](../02-business-logic/security.md), [settings](../02-business-logic/settings.md), [bug-reports](../02-business-logic/bug-reports.md) |
| Экраны | журнал ИБ, настройки, пользователи — описания дописываются |
| API | [endpoints-security](../04-api/endpoints-security.md), [endpoints-settings](../04-api/endpoints-settings.md), [endpoints-bug-reports](../04-api/endpoints-bug-reports.md) |

### 10. ИИ

**Результат, который принимаем:** Без ключа ИИ система работает полностью и показывает детерминированный результат; сценарии ИИ проходят на заглушке шлюза; ни одно число из ответа модели не попадает пользователю без сверки с данными

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 10](work-breakdown.md#этап-10-ии--xl) |
| Бизнес-логика | [insights](../02-business-logic/insights.md) |
| Экраны | [tender-card-analysis](../03-screens/tender-card-analysis.md) |
| API | [endpoints-tenders-analysis](../04-api/endpoints-tenders-analysis.md) |
| Ещё | [ai-features](../06-ai/ai-features.md) |

### 11. Биллинг ИИ, кабинет владельца, заявки с лендинга

**Результат, который принимаем:** Успешная операция ИИ списывается ровно один раз, неуспешная — не списывается; записи баланса не меняются и не удаляются. Этап необязательный (Could) — оцените его отдельно

| Что | Где в ТЗ |
|---|---|
| Состав и приёмка | [Этап 11](work-breakdown.md#этап-11-биллинг-ии-кабинет-владельца-заявки-с-лендинга--l-could) |
| Бизнес-логика | [billing](../02-business-logic/billing.md) |
| Экраны | [welcome](../03-screens/welcome.md) |
| API | [endpoints-billing](../04-api/endpoints-billing.md) |

## Как заполнить шаблон

| Столбец | Что писать |
|---|---|
| Цена, ₽ | Фиксированная цена за сданный этап, с налогами. Одно число, без вилки |
| Срок, недель | Календарный срок этапа при вашем составе команды |
| Не входит в цену | Что из состава этапа вы не делаете или считаете отдельно |
| Нужно от Заказчика | Доступы, данные, решения, без которых этап не начать или не сдать |
| Вопросы и риски | Что в ТЗ неясно и как это может изменить цену или срок |

Над таблицей:

- **Исполнитель** — имя или компания и контакт.
- **Состав команды** — кто будет работать, одной строкой. Нужен, чтобы понять срок, а не
  чтобы проверять часы.

Под таблицей:

- **Итого** — сумма этапов П–10, считается сама. Этап 11 в итог не входит.
- **Общий срок, недель** — с учётом этапов, которые вы ведёте параллельно.

Открытые вопросы ТЗ собраны в [`_oq/`](../_oq/). Если вопрос влияет на вашу оценку, примите
своё допущение и запишите его в «Вопросы и риски».
