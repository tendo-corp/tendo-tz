# Демо-стенд для Заказчика: развёртывание с нуля

Пошаговая инструкция и требования: как поднять демо-стенд на чистом сервере — состав
контейнеров, порядок старта, миграции, загрузка демо-данных, домены и TLS, переменные
окружения, первый суперадмин, проверка работоспособности и типовые проблемы.

Связанные разделы: [окружения](environments.md), [CI/CD](ci-cd.md),
[архитектура](../07-architecture/architecture.md), [безопасность](../07-architecture/security.md),
[нефункциональные требования](../07-architecture/nonfunctional.md). Открытые вопросы —
`../_oq/INFRA.md`.

Нумерация требований: **DPL-NN**. Метки сложности: **S / M / L / XL**.
Все адреса, имена и значения ниже — **плейсхолдеры** вида `<...>`.

- **DPL-01.** Исполнитель поставляет: прод-образы бэкенда и фронтенда, пример compose-файла и
  конфигурации reverse-proxy, демо-набор данных (дамп + архив вложений или сид-скрипт),
  CLI-команды суперадмина и первого администратора, скрипт накатки демо-данных и этот
  документ, проверенный «с нуля» на чистом сервере. **M**

---

## 1. Требования к серверу

Оценка для демо-стенда **без локальной модели ИИ** (ИИ — внешний OpenAI-совместимый шлюз или
выключен). Допущения: до 30 одновременных пользователей, данные уровня демо-набора, один
экземпляр каждого сервиса.

| Ресурс | Минимум | Рекомендуется | Комментарий |
|---|---|---|---|
| vCPU | 2 | 4 | Бэкенд — 4 процесса uvicorn (`WEB_CONCURRENCY`), по ≈ 1 ядру на 30 RPS |
| RAM | 4 ГБ | 8 ГБ | Postgres 1–2 ГБ, бэкенд ≈ 1–1,5 ГБ на 4 процесса, прочее ≤ 0,5 ГБ; **+2 ГБ**, если включён ClamAV |
| Диск | 40 ГБ SSD | 80 ГБ NVMe | Образы, БД, логи, резервные копии; **+ объём файлов**, если хранилище на этом же сервере |
| ОС | Ubuntu Server 24.04 LTS / Debian 13, x86_64 | | Docker CE + Docker Compose v2 |
| Сеть | Публичный IP, A-записи доменов, открыты 80/443 | | Исходящий HTTPS: registry образов, SMTP, ИИ-шлюз, Telegram (опц.) |
| Время | NTP-синхронизация | | Иначе ломаются TOTP-коды и presigned-ссылки |

Вариант с локальной моделью ИИ (GPU, десятки ГБ под модели) — отдельное согласование с
Заказчиком (см. открытые вопросы).

---

## 2. Состав контейнеров

| Сервис | Образ | Порт (внутри сети) | Тома | Healthcheck | Обязателен |
|---|---|---|---|---|---|
| `proxy` | Reverse-proxy с TLS: Caddy 2 / traefik v3 / nginx (или встроенный в PaaS) | **80, 443 наружу** | сертификаты/состояние ACME | — | да |
| `frontend` | Прод-образ фронта: статика + Caddy 2, пользователь UID 10001 | 3000 | — | `GET http://127.0.0.1:3000/` каждые 10 с, старт 5 с | да |
| `backend` | Прод-образ бэкенда: Python 3.13 (alpine), Atlas CLI, пользователь UID 10001, `WEB_CONCURRENCY=4` | 8000 | нет (при `s3`); каталог `UPLOAD_DIR` при `local` | `GET http://127.0.0.1:8000/health` каждые 10 с, тайм-аут 3 с, **старт 60 с** (миграции), 3 попытки | да |
| `postgres` | `postgres:18-alpine` (закрепить минорную версию, исходная — 18.6) | 5432 | данные: `/var/lib/postgresql` (у PG 18 — подкаталог версии внутри) | `pg_isready -h 127.0.0.1 -U … -d …` каждые 2–5 с | да |
| `s3` | S3-совместимое хранилище: облачное **или** self-hosted (MinIO, SeaweedFS, Garage — закрепить тег) | 9000 (у MinIO) | данные объектов | HTTP-проба живости сервера хранилища | да (или облако) |
| `mail` | Для демо — SMTP-релей провайдера (реальная доставка) **или** ловушка mailpit (веб-интерфейс только за паролем) | SMTP 1025, веб 8025 | база писем | `GET :8025/` | по выбору |
| `clamav` | `clamav/clamav` | 3310 | базы сигнатур | `clamdscan --ping` или аналог | нет (`VIRUS_SCANNER=clamav`) |

- **DPL-02.** S3-эмулятор локального стенда (не проверяет подписи, принимает любые ключи) на
  демо-стенде **не используется**.
- **DPL-03.** Все контейнеры, кроме `proxy`, — только во внутренней сети compose-проекта; порты
  Postgres, хранилища (кроме домена через proxy), почты и ClamAV наружу не публикуются.

---

## 3. Подготовка

1. **DNS.** A-записи: `<APP_DOMAIN>` (SPA и API) и, если хранилище self-hosted, `<S3_DOMAIN>`
   (браузер ходит туда по presigned-ссылкам). Облачному хранилищу отдельный домен не нужен.
2. **Секреты** — сгенерировать и положить в env-файл/секреты PaaS (не в git):
   - `JWT_SECRET` — ≥ 32 случайных байт (например, `openssl rand -hex 32`);
   - `FIELD_ENCRYPTION_KEY` — 32 случайных байта в urlsafe-base64 (формат Fernet):
     `openssl rand -base64 32 | tr '+/' '-_'`;
   - `POSTGRES_PASSWORD` — `openssl rand -hex 16`;
   - ключи хранилища, учётка SMTP, ключ ИИ — от соответствующих сервисов.
   Ключ шифрования полей сохранить в хранилище секретов: без него дампы с ПДн нечитаемы.
3. **Доступ к образам.** Вход в приватный registry (`docker login <REGISTRY>`) или загрузка
   образов архивом (`docker load`) для изолированного контура. Теги — SHA коммитов
   (`<REGISTRY>/core:<sha>`, `<REGISTRY>/web:<sha>`).
4. **Бакет.** Создать бакет (`<BUCKET>`), сервисную учётку с правами чтения/записи **только** на
   него, CORS бакета (§8.3). Для self-hosted можно `S3_CREATE_BUCKET=true` — бэкенд создаст бакет
   на старте.

---

## 4. Порядок старта

```mermaid
flowchart LR
  pg["postgres<br/>healthy"] --> s3["s3<br/>healthy"]
  s3 --> av["clamav<br/>опционально"]
  av --> be["backend<br/>миграции → uvicorn → healthy"]
  s3 --> be
  be --> fe["frontend"]
  fe --> rp["proxy<br/>TLS"]
  be --> rp
```

1. `postgres` → ждать `healthy`.
2. Хранилище (если self-hosted) → `healthy`; ClamAV, если включён (первый старт качает базы
   сигнатур — минуты).
3. **Демо-данные** (§6) — до первого старта бэкенда или с остановленным бэкендом.
4. `backend`: при старте контейнер выполняет `atlas migrate apply` и только затем запускает
   uvicorn; в lifespan — проверки конфигурации, создание бакета (если включено), запуск воркера и
   слушателя realtime. Ждать `healthy` (до 60 с + время миграций).
5. `frontend`, затем `proxy` (выпуск TLS-сертификата при первом обращении).

В compose — `depends_on` с `condition: service_healthy`.

---

## 5. Миграции БД

- **DPL-04.** Миграции — версионированные SQL-файлы (Atlas), применяются **автоматически при
  старте контейнера бэкенда** командой вида
  `atlas migrate apply -u "<DATABASE_URL без +asyncpg>?sslmode=disable&search_path=public"` до
  запуска API; провал миграции — контейнер не стартует (видно в логах). Таблица ревизий хранится
  в той же БД. **S**
- **DPL-05.** Миграции — **только вперёд**, без down-скриптов; дамп старше кода доводится до
  актуальной схемы на старте. Дамп **новее** кода (ревизии, неизвестные образу) — ошибка: нужно
  выкатить более новый образ.
- **DPL-06.** Строка подключения миграций в исходном образе содержит `sslmode=disable` — БД должна
  быть доступна без TLS во внутренней сети. Для управляемой БД с обязательным TLS режим SSL
  выносится в настройку. **S**
- **DPL-07.** Часть объектов, которые декларативная схема не выражает, живёт только в
  SQL-миграциях: триггеры append-only на журнале ИБ и истории квалификации, `UNLOGGED` у таблиц
  кэша и счётчиков лимитов, последовательность событий чата.

---

## 6. Загрузка демо-данных

Демо-набор — артефакт поставки: дамп БД `init.dump` (`pg_dump -Fc`) и архив вложений
`uploads.tar` (объекты бакета с ключами `<тип сущности>/<id>/<uuid>_<имя>`). Содержание —
обезличенные портфели, объекты, контрагенты, тендеры с КП нескольких версий и раундов, учётки всех
ролей (состав — в разделе продукта).

- **DPL-08.** Накатка — скриптом, повторяющим процедуру исходной системы: **M**

```bash
# переменные: POSTGRES_USER, POSTGRES_DB заданы; бэкенд остановлен
STAMP=$(sha256sum init.dump | cut -d' ' -f1)
docker compose stop backend

# DROP и restore одной транзакцией; COMMIT — только если pg_restore дочитал дамп до конца
{ echo "SET client_min_messages = warning; BEGIN; DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
  docker compose exec -T postgres pg_restore --no-owner --no-acl -f - < init.dump && echo "COMMIT;"; } \
  | docker compose exec -T postgres psql -v ON_ERROR_STOP=1 -q -U "$POSTGRES_USER" -d "$POSTGRES_DB"

# отметка: какой дамп накатан
echo "SELECT format('COMMENT ON DATABASE %I IS %L', current_database(), 'dump $STAMP') \gexec" \
  | docker compose exec -T postgres psql -qtAX -U "$POSTGRES_USER" -d "$POSTGRES_DB"

# вложения → бакет (существующие объекты не удаляются)
mkdir -p ./uploads && tar -xf uploads.tar -C ./uploads
docker run --rm -v "$PWD/uploads:/files" \
  -e AWS_ACCESS_KEY_ID=<S3_ACCESS_KEY> -e AWS_SECRET_ACCESS_KEY=<S3_SECRET_KEY> -e AWS_DEFAULT_REGION=<S3_REGION> \
  --network <сеть_compose> amazon/aws-cli:<тег> --endpoint-url <S3_ENDPOINT> s3 sync /files s3://<BUCKET>

docker compose start backend   # миграции доведут схему, если дамп старше кода
```

- **DPL-09.** Скрипт идемпотентен по отметке: если `COMMENT ON DATABASE` уже равен
  `dump <sha256>`, накатка пропускается. При любом исходе ранее работавший бэкенд запускается
  снова; ожидание `healthy` — до 150 с.
- **DPL-10.** Дамп и ключ шифрования полей — пара: значения, зашифрованные другим ключом, читаются
  как пустые (в т.ч. секреты MFA — таким пользователям администратор сбрасывает MFA). Демо-набор
  должен хранить ПДн-поля открытым текстом (они зашифруются при следующей записи) или
  поставляться вместе с ключом.
- **DPL-11.** Пароли демо-учёток задаются при подготовке набора и передаются Заказчику отдельным
  каналом; в документации и на экране входа их нет (демо-подсказки показывают только адреса
  учёток при `VITE_DEMO_TOOLS=1`).

---

## 7. Переменные окружения

Любое несекретное поле конфигурации переопределяется переменной с тем же именем в верхнем
регистре, вложенные — через `__`. Ниже — имена и назначение; **значений секретов нет**.
Колонка «Демо» — рекомендуемая установка для демо-стенда.

### 7.1. Бэкенд — обязательные и топология

| Переменная | Назначение | Демо |
|---|---|---|
| `CONFIG_FILE` | Базовый YAML (`prod.yaml` по умолчанию, `dev.yaml`) | `prod.yaml` |
| `DATABASE_URL` | `postgresql+asyncpg://<user>:<pass>@postgres:5432/<db>` (секрет) | обязательно |
| `JWT_SECRET` | Подпись JWT (секрет, без умолчания) | обязательно |
| `FIELD_ENCRYPTION_KEY` | Ключ шифрования полей (секрет, без умолчания) | обязательно |
| `PUBLIC_APP_URL` | Публичный URL приложения (`https://…`): ссылки в письмах, проверка Origin | `https://<APP_DOMAIN>` |
| `CORS_ORIGINS` | JSON-список разрешённых origin (не пустой, без `*`) | `["https://<APP_DOMAIN>"]` (+ origin лендинга, если есть) |
| `WEB_CONCURRENCY` | Число процессов uvicorn (в образе 4) | 4 |
| `FORWARDED_ALLOW_IPS` | Доверенные прокси для `X-Forwarded-*` (в образе — loopback и частные сети) | умолчание образа |
| `COOKIE_SECURE` | Флаг `Secure` у cookie | `true` |

### 7.2. Бэкенд — хранилище и файлы

| Переменная | Назначение |
|---|---|
| `STORAGE_BACKEND` | `local` / `s3` (демо — `s3`) |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET` | Адрес хранилища для бэкенда, регион, бакет |
| `S3_ACCESS_KEY`, `S3_SECRET_KEY` | Ключи (секрет) |
| `S3_PRESIGN_ENDPOINT` | Адрес для подписи ссылок браузеру, если отличается от `S3_ENDPOINT` |
| `S3_CREATE_BUCKET` | Создать бакет на старте (self-hosted) |
| `S3_PRESIGN_TTL_SEC` | Срок жизни ссылок, 60–3600 (900) |
| `MAX_UPLOAD_MB` | Лимит вложения (25) |
| `MAX_EXCEL_UPLOAD_MB` | Лимит Excel мастера тендера (50) |
| `UPLOAD_PENDING_TTL_HOURS` | Уборка незавершённых загрузок (24) |
| `UPLOAD_DIR` | Каталог файлов в режиме `local` |
| `VIRUS_SCANNER`, `CLAMAV_HOST`, `CLAMAV_PORT` | Антивирус: `noop` / `clamav`, адрес clamd |

### 7.3. Бэкенд — почта

| Переменная | Назначение | Демо |
|---|---|---|
| `MAIL_ENABLED` | Рубильник всей почты | `true` |
| `SMTP_SETTINGS_IN_UI` | SMTP из интерфейса (`true`) или из окружения (`false`) | `false` |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USE_TLS` | Сервер, порт (465 — SMTPS), STARTTLS | релей или mailpit |
| `SMTP_USERNAME`, `SMTP_PASSWORD` | Учётка SMTP (секрет) | |
| `SMTP_FROM_EMAIL` | Адрес отправителя | `noreply@<APP_DOMAIN>` |
| `ALLOW_LOCAL_EMAIL_DOMAINS` | Разрешить адреса на служебных доменах (`.local`, `.localhost`, `.test`) | `true` (демо-учётки) |

### 7.4. Бэкенд — безопасность и сессии

| Переменная | Назначение (умолчание) |
|---|---|
| `MFA_ENFORCED` | Обязательная MFA для администратора и владельца (`true`) |
| `ACCESS_TOKEN_MINUTES` | Срок JWT доступа (15) |
| `SESSION_IDLE_DAYS`, `SESSION_ABSOLUTE_DAYS` | Срок простоя (14) и потолок сессии (30) |
| `MAX_FAILED_LOGIN_ATTEMPTS`, `LOGIN_THROTTLE_MAX_MINUTES` | Порог неудач (5) и максимум паузы (60) |
| `CONTACTS_MASK_ENABLED` | Маска контактов контрагента (`false`) |
| `SHARED_RATE_LIMIT` | Счётчики лимитов в БД (`true`) |

### 7.5. Бэкенд — ИИ и биллинг

| Переменная | Назначение |
|---|---|
| `AI_ENABLED` | Общий рубильник ИИ |
| `AI_BASE_URL`, `AI_API_KEY` | Адрес OpenAI-совместимого API и ключ (секрет) |
| `AI_HTTP_PROXY` | Прокси только для запросов к ИИ |
| `AI_MODELS__MECHANICAL`, `AI_MODELS__LANGUAGE`, `AI_MODELS__EMBEDDING` | Модель по ролям |
| `AI_PIPELINES__NEGOTIATION_AGENDA`, `…__PAIR_DUEL`, `…__POSITION_MATCHING`, `…__SUPPLIER_LETTER`, `…__TABLE_QA` | Включение пайплайнов |
| `AI_STEP_BUDGET_SECONDS`, `AI_RUN_BUDGET_SECONDS` | Бюджеты времени (60 / 180) |
| `AI_MATCHING_CONFIDENT`, `AI_MATCHING_REVIEW` | Пороги сопоставления позиций (0,90 / 0,70) |
| `BILLING_ENABLED` | Клиентская часть биллинга ИИ (демо — `true`) |
| `CLIENT_CODE` | Код клиента установки |
| `BILLING_WELCOME_BONUS` | Приветственный бонус в токенах |
| `BILLING_OWNER_EMAIL`, `BILLING_TELEGRAM_ENABLED`, `BILLING_TELEGRAM_CHAT_ID` | Куда писать владельцу о заявках и пороге |
| `AI_PROMPT_LOG_ENABLED`, `AI_PROMPT_LOG_DAYS` | Хранить тексты промптов и срок (30) |

### 7.6. Бэкенд — функции, тексты, бренд

| Переменная | Назначение |
|---|---|
| `LEADS_ENABLED` | Заявки с лендинга `POST /leads` и роль «Лид» (демо — `true`; без него ручка отвечает 404 и скрыта из схемы API) |
| `BRAND_NAME` | Имя площадки в письмах и выгрузках |
| `CLIENT_NAME` | Имя клиента-владельца («Обратитесь к представителю …») |
| `TOTP_ISSUER` | Издатель в приложении-аутентификаторе (пусто — `BRAND_NAME`) |
| `PROTOCOL_PLACE`, `PROTOCOL_CHAIRMAN` | Значения по умолчанию тендерного протокола |
| `VAT_RATE_PCT` | Ставка НДС в текстах (22) |
| `FKP_FORM_LOCKED` | Неизменяемая форма ФКП у участника (`true`) |
| `CREATION_DRAFT_TTL_DAYS` | Срок черновиков мастера тендера (30) |

### 7.7. Бэкенд — логи, алерты, интеграции

| Переменная | Назначение |
|---|---|
| `LOG_LEVEL`, `LOG_FORMAT` | Уровень; `text` / `jsonl` |
| `ALERTS_ENABLED`, `ALERTS_DEDUPE_SECONDS` | Техалерты в Telegram, окно дедупа (300) |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | Бот и чат (секрет) |
| `TELEGRAM_HTTP_PROXY` | Прокси только для Telegram |
| `LOG_PATTERNS_ENABLED`, `LOG_PATTERNS_REFRESH_INTERVAL` | Паттерны логов → Telegram, обновление правил (30 с) |
| `LOG_FORWARD_ENABLED`, `LOG_FORWARD_URL`, `LOG_FORWARD_TOKEN` | Пересылка техлогов (только при `WEB_CONCURRENCY=1`) |
| `LOG_FORWARD_BATCH_SIZE`, `LOG_FORWARD_INTERVAL`, `LOG_FORWARD_BUFFER_SIZE` | Пачка (200), период (10 с), буфер (5000) |
| `SIEM_DRAIN_INTERVAL` | Период передачи журнала ИБ в SIEM (15 с) |
| `BUGREPORT_GITHUB_ENABLED`, `BUGREPORT_GITHUB_OWNER`, `BUGREPORT_GITHUB_REPO`, `BUGREPORT_GITHUB_LABELS`, `BUGREPORT_JOB_MAX_ATTEMPTS` | Баг-репорты во внешний трекер |
| `GITHUB_BUGREPORT_TOKEN`, `GITHUB_BUGREPORT_WEBHOOK_SECRET` | Токен трекера и секрет подписи вебхука (секрет) |

### 7.8. Бэкенд — производительность

`DB_POOL_SIZE`, `DB_MAX_OVERFLOW`, `DB_POOL_RECYCLE`, `DB_POOL_TIMEOUT`, `DB_COMMAND_TIMEOUT`,
`DB_STATEMENT_TIMEOUT_MS`, `DB_JIT`, `DB_APPLICATION_NAME`, `SLOW_REQUEST_MS`,
`REQUEST_TIMEOUT_SECONDS`, `GZIP_MIN_BYTES`, `GZIP_LEVEL`, `HEAVY_READ_CACHE_SECONDS`,
`HEAVY_READ_RATE_LIMIT`, `HEAVY_READ_RATE_WINDOW`, `PARSE_JOB_HANG_SECONDS`,
`PARSE_JOB_MAX_ATTEMPTS`, `PARSE_JOB_POLL_INTERVAL` — значения и смысл в
[scalability.md](../07-architecture/scalability.md) и [nonfunctional.md](../07-architecture/nonfunctional.md).

### 7.9. Postgres

`POSTGRES_USER`, `POSTGRES_PASSWORD` (секрет), `POSTGRES_DB`; `TZ=UTC`.

### 7.10. Фронтенд — build-args (вшиваются в бандл при сборке образа)

| Аргумент | Назначение | Демо | Прод Заказчика |
|---|---|---|---|
| `VITE_API_URL` | Адрес API (по умолчанию `/api` на том же домене) | `/api` | `/api` |
| `VITE_APP_VERSION` | Версия сборки (SHA коммита) | SHA | SHA |
| `VITE_DEMO_TOOLS` | Демо-учётки на входе, стенд плотной матрицы, пример файла КП, «Заполнить за меня» | `1` | `0` |
| `VITE_LEADS` | Роль «Лид», фильтр «Источник», экран `/welcome` (пара к `LEADS_ENABLED`) | `1` | `0` |
| `VITE_ENABLE_SIEM_TRANSFER` | Показ настройки передачи в SIEM | `0` | по требованию СБ |
| `VITE_CLIENT_NAME`, `VITE_CLIENT_GENITIVE` | Имя клиента в текстах и его родительный падеж | пусто (нейтральная заглушка) | имя Заказчика |
| `VITE_CLIENT_LOGO`, `VITE_CLIENT_LOGO_ON_DARK` | Пути логотипов клиента внутри статики | пусто | логотипы Заказчика |
| `VITE_CLIENT_EMAIL_DOMAIN` | Домен в подсказках полей почты | пусто | домен Заказчика |

- **DPL-12.** Флаги фронта не являются секретами и по умолчанию выключены; возможность, которой не
  место в боевом контуре, включается только флагом. **S**

---

## 8. Домены, маршрутизация, TLS

### 8.1. Схема

| Адрес | Куда | Примечание |
|---|---|---|
| `https://<APP_DOMAIN>/` и любой путь кроме `/api` | `frontend:3000` | SPA-фолбэк на `index.html` |
| `https://<APP_DOMAIN>/api/*` | `backend:8000` **со срезанием префикса** `/api` | Бэкенд знает префикс (`root_path=/api`), Swagger — `/api/docs`; путь cookie — `/api/…` |
| `https://<S3_DOMAIN>/` | хранилище (self-hosted) | Только для presigned-ссылок браузера; `S3_PRESIGN_ENDPOINT=https://<S3_DOMAIN>` |
| `http://*:80` | редирект на HTTPS | |

- **DPL-13.** Reverse-proxy должен: терминировать TLS (Let's Encrypt или сертификат Заказчика),
  передавать `X-Forwarded-For` и `X-Forwarded-Proto`, срезать `/api`, пропускать тела до 60 МБ
  (Excel до 50 МБ + запас), **не буферизовать и не сжимать** `text/event-stream`, держать
  простаивающее соединение не меньше 60 с (пинг SSE — раз в 15 с). **S**

### 8.2. Пример конфигурации reverse-proxy (Caddy)

```caddyfile
<APP_DOMAIN> {
	request_body {
		max_size 60MB
	}
	handle_path /api/* {
		reverse_proxy backend:8000 {
			flush_interval -1
		}
	}
	handle {
		reverse_proxy frontend:3000
	}
}

<S3_DOMAIN> {
	reverse_proxy s3:9000
}
```

Эквивалент для nginx: `location /api/ { proxy_pass http://backend:8000/; proxy_buffering off;
proxy_read_timeout 3600s; client_max_body_size 60m; proxy_set_header X-Forwarded-Proto $scheme; … }`.
Для traefik — метки `PathPrefix(/api)` + middleware `stripprefix`.

### 8.3. CORS бакета

Без него браузер не сможет выполнить прямой `PUT` (фронт тогда молча загружает через API —
файлы доезжают, но мимо смысла прямой загрузки). Скачивание по ссылке CORS не требует.

```json
{"CORSRules": [{
  "AllowedOrigins": ["https://<APP_DOMAIN>"],
  "AllowedMethods": ["PUT", "GET", "HEAD"],
  "AllowedHeaders": ["Content-Type"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}]}
```

Применяется консолью провайдера или `aws s3api put-bucket-cors --endpoint-url <S3_ENDPOINT>
--bucket <BUCKET> --cors-configuration file://cors.json`; у self-hosted серверов, не
поддерживающих правила бакета, — настройкой сервера (разрешённый origin = `https://<APP_DOMAIN>`).
Бакет — не публичный. Рекомендуется lifecycle-правило: удалять `jobs/` старше суток.

---

## 9. Пример compose-файла (обобщённый)

```yaml
name: tendo-demo

services:
  postgres:
    image: postgres:18-alpine          # закрепить минорную версию
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      TZ: UTC
    volumes: [pg-data:/var/lib/postgresql]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h 127.0.0.1 -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 3s
      retries: 30

  s3:                                   # не нужен при облачном хранилище
    image: <образ S3-совместимого сервера>:<тег>
    restart: unless-stopped
    environment: {}                     # учётка и настройки сервера хранилища
    volumes: [s3-data:/data]
    healthcheck:
      test: ["CMD-SHELL", "<проба живости сервера хранилища>"]
      interval: 5s
      timeout: 3s
      retries: 30

  backend:
    image: <REGISTRY>/core:${CORE_TAG}
    restart: unless-stopped
    env_file: backend.secrets.env       # JWT_SECRET, FIELD_ENCRYPTION_KEY, S3_*_KEY, SMTP_*, AI_API_KEY …
    environment:
      CONFIG_FILE: prod.yaml
      DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      PUBLIC_APP_URL: https://<APP_DOMAIN>
      CORS_ORIGINS: '["https://<APP_DOMAIN>"]'
      STORAGE_BACKEND: s3
      S3_ENDPOINT: http://s3:9000
      S3_PRESIGN_ENDPOINT: https://<S3_DOMAIN>
      S3_BUCKET: <BUCKET>
      S3_REGION: <S3_REGION>
      S3_CREATE_BUCKET: "true"
      MAIL_ENABLED: "true"
      SMTP_SETTINGS_IN_UI: "false"
      SMTP_HOST: <SMTP_HOST>
      SMTP_PORT: "587"
      SMTP_FROM_EMAIL: noreply@<APP_DOMAIN>
      ALLOW_LOCAL_EMAIL_DOMAINS: "true"
      LEADS_ENABLED: "true"
      BILLING_ENABLED: "true"
      AI_ENABLED: "false"               # "true" + AI_BASE_URL и ключ, если ИИ согласован
      LOG_FORMAT: jsonl
    depends_on:
      postgres: {condition: service_healthy}
      s3: {condition: service_healthy}

  frontend:
    image: <REGISTRY>/web:${WEB_TAG}    # собран с VITE_DEMO_TOOLS=1, VITE_LEADS=1
    restart: unless-stopped
    depends_on:
      backend: {condition: service_healthy}

  proxy:
    image: caddy:2-alpine               # закрепить тег
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on: [frontend, backend]

volumes:
  pg-data:
  s3-data:
  caddy-data:
  caddy-config:
```

Healthcheck бэкенда и фронтенда встроены в образы (§2). Логи контейнеров — с ротацией драйвера
(`max-size`/`max-file`) или сборщиком логов.

---

## 10. Первый суперадмин и администратор

- **DPL-14.** Владелец площадки (суперадмин, раздел биллинга ИИ `/owner`) выдаётся CLI-командой в
  контейнере бэкенда (вида `docker compose exec backend <cli> superadmin <owner-email>`).
  Нет учётки — создаётся с одноразовым паролем, который печатается **один раз**; есть — получает
  признак. MFA настраивается при первом входе (обязательна). Суперадмину доступны только `/auth/*`
  и `/owner*`. **S**
- **DPL-15.** Первый администратор Заказчика: в исходной системе начальная миграция создаёт
  демо-учётки всех ролей (включая администратора) с общим известным паролем. В воссоздаваемой
  системе — CLI-команда `create-admin <email>` с одноразовым паролем и обязательной сменой при
  первом входе; демо-учётки — только в демо-наборе (см.
  [security.md SEC-18](../07-architecture/security.md#16-особые-учётные-записи)). **S**
- **DPL-16.** Сразу после развёртывания администратор: настраивает MFA, проверяет SMTP, задаёт
  адреса уведомлений в «Настройках», выдаёт роли и доступ к портфелям/проектам.

---

## 11. Проверка работоспособности

| # | Проверка | Ожидаемо |
|---|---|---|
| 1 | `curl -fsS https://<APP_DOMAIN>/api/health` | `{"status":"ok"}` |
| 2 | `GET /api/health/storage` (из внутренней сети или под администратором) | `"ok": true`, `"storage_backend": "s3"`, `"direct_upload": true` |
| 3 | `GET /api/health/llm` | ИИ выключен — `"ai_enabled": false`; включён — `"ok": true`, `"generate_ok": true` |
| 4 | Логи бэкенда при старте | Строка эффективной конфигурации с `***` вместо секретов; «worker: процесс стал лидером очереди заданий»; «realtime: слушаем канал app_realtime»; предупреждение о несертифицированном СКЗИ; ни одного ERROR |
| 5 | Открыть `https://<APP_DOMAIN>/` | Экран входа (при демо-флаге — подсказки демо-учёток); редирект http → https |
| 6 | Вход администратором | Принудительная настройка MFA (если не настроена), затем рабочий экран; cookie `HttpOnly; Secure; SameSite=Lax` |
| 7 | Загрузка вложения в тендер | В инструментах разработчика: `presign-upload` 200 → `PUT` на `<S3_DOMAIN>` 200 → `confirm` 200; файл скачивается |
| 8 | Загрузка Excel при создании тендера (до 50 МБ) | Задание разбора `done`, разделы и позиции появились |
| 9 | Создать пользователя → письмо активации | Письмо пришло (релей — в ящик, mailpit — в веб-интерфейс); ссылка ведёт на `https://<APP_DOMAIN>/…` |
| 10 | Живые события | Два браузера: сообщение в чате/уведомление появляется без перезагрузки |
| 11 | Свагер | `https://<APP_DOMAIN>/api/docs` доступен только как решено (SEC-51) |
| 12 | Перезапуск `backend` | Через ≤ 60 с снова `healthy`, прерванные задания вернулись в очередь |

- **DPL-17.** Проверки 1–3 и 5 выполняются задачей выкатки автоматически (см. [ci-cd.md](ci-cd.md)),
  остальные — по чек-листу при первом развёртывании и после крупных обновлений. **S**

---

## 12. Типовые проблемы

| Симптом | Причина | Что сделать |
|---|---|---|
| Бэкенд не стартует: «Конфигурация не готова: …» | Не задан/небезопасен параметр (CORS, `PUBLIC_APP_URL` без https, dev-секрет, `LOG_FORWARD_ENABLED` при нескольких процессах, s3 без ключей) | Исправить перечисленные переменные |
| «CONFIG_FILE=…: нет файла» | Опечатка в имени YAML | `prod.yaml` или `dev.yaml` |
| Контейнер бэкенда перезапускается, в логе ошибка Atlas | БД недоступна, требует TLS (а миграции идут с `sslmode=disable`), или дамп новее кода | Проверить сеть и `DATABASE_URL`; выкатить образ не старше дампа |
| `unhealthy` в первые минуты | Долгие миграции дольше `start-period` (60 с) | Подождать / увеличить `start-period` |
| 404 на `/api/...`, Swagger просит `/openapi.json` у фронта | Proxy не срезает `/api` или срезает дважды | Проверить правило `/api` и strip-prefix |
| Вход «проходит», но следующий запрос — 401 | Cookie `Secure` по http, домен/путь не совпадает, часы сервера ушли | Только HTTPS; `PUBLIC_APP_URL` = фактический адрес; NTP |
| 403 «Запрос с чужого источника отклонён» | Фактический origin не в `CORS_ORIGINS`/`PUBLIC_APP_URL` | Привести к одному адресу (с протоколом, без слеша в конце) |
| Лимиты входа срабатывают «на всех сразу» | Бэкенд видит IP прокси: прокси вне `FORWARDED_ALLOW_IPS` | Добавить адрес прокси (не `*`) |
| Прямая загрузка падает с ошибкой CORS или `SignatureDoesNotMatch` | Нет CORS бакета; `S3_PRESIGN_ENDPOINT` не тот адрес, который видит браузер | Настроить CORS (§8.3); подпись строится на адрес браузера |
| 413 при загрузке Excel | Лимит тела на прокси (у nginx по умолчанию 1 МБ) | Поднять до 60 МБ на `/api` |
| Бейджи и чат не обновляются, поток рвётся каждые N секунд | Прокси буферизует/сжимает `text/event-stream` или режет простаивающее соединение | Отключить буферизацию для `/api/*/stream`, тайм-аут простоя ≥ 60 с |
| Письма не уходят | `MAIL_ENABLED=false`, пустой `SMTP_HOST`, почта на паузе в настройках, адрес на зарезервированном/служебном домене без `ALLOW_LOCAL_EMAIL_DOMAINS` | Причина видна в следе доставки (задание) и в карточке получателя |
| ИИ-шлюз отвечает 403 «Access denied by security policy» | Региональная блокировка провайдера | `AI_HTTP_PROXY` на прокси в разрешённом регионе или другой провайдер |
| После накатки дампа пустые почты/телефоны контрагентов, MFA-коды не подходят | Дамп зашифрован другим `FIELD_ENCRYPTION_KEY` | Использовать ключ дампа; MFA — сбросить администратором |
| `too many connections` в Postgres | `WEB_CONCURRENCY` × пул > `max_connections` | Пересчитать бюджет ([scalability.md §3](../07-architecture/scalability.md#3-пул-соединений-и-тайм-ауты-бд)) |
| Демо-учётки на `*.local` отвергаются формами | `ALLOW_LOCAL_EMAIL_DOMAINS=false` | `true` на демо |
| Загрузки отклоняются при включённом антивирусе | clamd не поднят или ещё качает базы | Дождаться `healthy` ClamAV; недоступный антивирус — отказ по требованию |

---

## 13. Оценка

| Работа | Сложность |
|---|---|
| Прод-образы бэкенда и фронтенда (не root, healthcheck, миграции на старте) | S |
| Пример compose, конфигурация reverse-proxy, CORS бакета | S |
| Скрипт накатки демо-данных (транзакция, отметка sha256, вложения) | S |
| Подготовка демо-набора данных (обезличенные портфели, контрагенты, тендеры с КП и раундами, учётки ролей) | M |
| CLI суперадмина и первого администратора | S |
| Проверка «с нуля» на чистом сервере и правка документации | S |
