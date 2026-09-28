---

description: "Task list for EZ Planner MVP (001-voice-life-planner)"
---

# Tasks: EZ Planner — голосовой планировщик жизни в Telegram (MVP)

**Input**: Design documents from `/specs/001-voice-life-planner/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: план (Technical Context, R11) и конституция (принцип VI) требуют unit-тестов на Vitest
для чистой логики: подпись `initData`, доступ, правила дат и черновиков, схемы ответов ИИ, лента
дня, напоминания. Эти тесты включены и пишутся до реализации соответствующего модуля. Оценочные
наборы ИИ — отдельные задачи (SC-003, SC-004).

**Organization**: задачи сгруппированы по пользовательским историям, чтобы каждую можно было
реализовать и проверить отдельно.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: можно делать параллельно (разные файлы, нет зависимостей от незавершённых задач)
- **[Story]**: к какой истории относится задача (US1…US5)
- Пути — от корня репозитория, структура — по [plan.md](./plan.md)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: зависимости, конфигурация сборки, Worker и CI

- [ ] T001 Добавить зависимости в `package.json`. Рантайм: `hono`, `grammy`, `drizzle-orm`, `zod`
  (v4), `date-fns`, `@date-fns/tz`, `@tma.js/sdk-react`, `ulidx`. Dev: `wrangler`, `drizzle-kit`,
  `vitest`, `@cloudflare/workers-types`, `tsx`. Выполнить `npm install` и закоммитить
  `package-lock.json`.
- [ ] T002 Включить статический экспорт в `next.config.ts`: `output: 'export'`,
  `images: { unoptimized: true }`, `trailingSlash: false`. Проверить, что `npm run build` создаёт
  `out/`.
- [ ] T003 Создать `wrangler.jsonc`:
  - `name: "ez-planner"`, `main: "worker/index.ts"`, `compatibility_date: "2026-09-01"`,
    `compatibility_flags: ["nodejs_compat"]`;
  - `assets`: `directory: "./out"`, `binding: "ASSETS"`,
    `not_found_handling: "single-page-application"`, `run_worker_first: ["/api/*", "/bot/*"]`;
  - `d1_databases` (binding `DB`), `kv_namespaces` (binding `MEDIA`);
  - `queues`: producer `INPUTS` и consumer очереди `ez-inputs` с `max_batch_size: 1`,
    `max_retries: 3`;
  - `ai` (binding `AI`), `triggers.crons: ["* * * * *"]`, `vars.MINI_APP_URL`;
  - окружение `staging` со своими ресурсами.
- [ ] T004 [P] Настроить TypeScript:
  - в `tsconfig.json` алиас `@shared/*` → `src/shared/*` и исключение `worker/`;
  - создать `worker/tsconfig.json` (наследует корень, `types: ["@cloudflare/workers-types"]`,
    include `worker/**`, `src/shared/**`).
- [ ] T005 [P] Создать `vitest.config.ts` с алиасом `@shared` и папкой `tests/unit/`.
- [ ] T006 [P] Добавить скрипты в `package.json`:
  - разработка и деплой: `worker:dev`, `deploy` (`next build && wrangler deploy`), `deploy:staging`;
  - база: `db:generate`, `db:migrate:local`, `db:migrate:remote`;
  - сервисные: `bot:setup` (`tsx scripts/bot-setup.ts`), `eval` (`tsx scripts/eval.ts`);
  - проверки: `test` (`vitest run`), `typecheck` (корень и `worker/tsconfig.json`), `check`
    (lint + typecheck + test + build).
- [ ] T007 [P] Добавить шаги `npm test` и проверку типов Worker в `.github/workflows/ci.yml`.
- [ ] T008 [P] Создать `drizzle.config.ts`: `dialect: "sqlite"`, схема `worker/db/schema.ts`,
  вывод `migrations/`.
- [ ] T009 [P] Создать `.dev.vars.example` с `BOT_TOKEN`, `WEBHOOK_SECRET`, `OWNER_TELEGRAM_ID`;
  добавить `.dev.vars` в `.gitignore`.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: база, авторизация, каркас API, бота и Mini App — нужны всем историям

**⚠️ CRITICAL**: работа над историями начинается только после этой фазы

- [ ] T010 Описать биндинги и секреты в `worker/env.ts`:
  - биндинги `DB: D1Database`, `MEDIA: KVNamespace`, `INPUTS: Queue<InputJob>`, `AI: Ai`,
    `ASSETS: Fetcher`;
  - секреты и переменные `BOT_TOKEN`, `WEBHOOK_SECRET`, `OWNER_TELEGRAM_ID`, `MINI_APP_URL`;
  - тип `InputJob = { inputId: string }`.
- [ ] T011 Описать Drizzle-схему всех таблиц из `data-model.md` в `worker/db/schema.ts`:
  - таблицы: `users`, `testers`, `inputs`, `tasks`, `events`, `meals`, `meal_items`,
    `diary_entries`, `reminders`, `usage_daily`, `ai_budget_daily`;
  - значения по умолчанию: `timezone` «по умолчанию `Europe/Moscow`», `reminder_minutes`
    «одно из 0 (выкл), 5, 15, 30, 60; по умолчанию 15»;
  - внешние ключи: `meal_items.meal_id` и `reminders.event_id` с `on delete cascade`;
    `reminders.event_id` unique;
  - индексы: `tasks(user_id, due_date)`, `events(user_id, date)`, `meals(user_id, date)`,
    `diary_entries(user_id, date)`, `reminders(status, send_at)`, `inputs(user_id, received_at)`.
- [ ] T012 Сгенерировать миграцию `migrations/0000_init.sql` через `npm run db:generate` и применить
  локально (`npm run db:migrate:local`).
- [ ] T013 [P] Описать zod-схемы API из `contracts/api.md` в `src/shared/schemas.ts`: запросы и
  ответы `me`, `day`, `inputs`, CRUD-записей, коды ошибок `unauthorized`,
  `forbidden_not_tester`, `not_found`, `undo_expired`, `payload_too_large`, `unsupported_media`,
  `validation_failed`, `rate_limited`. Типы вывести в `src/shared/types.ts`.
- [ ] T014 [P] Написать тесты проверки `initData` в `tests/unit/init-data.test.ts`: валидная
  подпись, изменённое поле, неверный хеш, `auth_date` старше 24 часов, нет `user`.
- [ ] T015 Реализовать проверку `initData` в `worker/auth/init-data.ts`:
  - `secret = HMAC_SHA256("WebAppData", BOT_TOKEN)`, хеш по отсортированной строке
    `key=value\n…` без `hash`;
  - сравнение с постоянным временем; `auth_date` не старше 24 ч;
  - всё через WebCrypto. Тесты T014 проходят.
- [ ] T016 [P] Написать тесты решения о доступе в `tests/unit/access.test.ts`: владелец, тестировщик
  по id, тестировщик по username (регистр и `@` игнорируются), посторонний.
- [ ] T017 Реализовать доступ в `worker/auth/access.ts`:
  - чистая функция `decideAccess(ownerId, testers, user)`;
  - `ensureUser(db, tgUser)` — upsert в `users`;
  - при совпадении по username дописывает `telegram_id` в `testers`.
- [ ] T018 Создать каркас API в `worker/api/app.ts`:
  - Hono с префиксом `/api`;
  - middleware авторизации по заголовку `Authorization: tma <initData>`: 401 или 403
    `forbidden_not_tester`;
  - помощник валидации тела через zod → 422;
  - общий обработчик ошибок в формате `{ error: { code, message } }` с русскими сообщениями.
- [ ] T019 Создать каркас бота в `worker/bot/bot.ts`:
  - фабрика grammY `Bot` с `parse_mode: "HTML"`;
  - middleware доступа: посторонним отвечать «Сейчас EZ Planner в закрытом тестировании. Ваш ID:
    <id> — отправьте его владельцу, чтобы получить доступ.» и ничего не сохранять;
  - `webhookCallback(bot, "cloudflare-mod", { secretToken: WEBHOOK_SECRET })`.
- [ ] T020 Собрать точку входа `worker/index.ts`:
  - `fetch`: `/api/*` → Hono, `POST /bot/webhook` → бот, остальное → `env.ASSETS.fetch`;
  - `queue` → `worker/pipeline/consumer.ts` (заглушка до US1);
  - `scheduled` → `worker/reminders/send.ts` (заглушка до US4).
- [ ] T021 [P] Реализовать помощники времени в `worker/lib/time.ts`: `nowLocal(tz)`,
  `localDate(instant, tz)`, `toUtc(date, time, tz)`, `isValidTimeZone(tz)` на `@date-fns/tz`.
  Тесты `tests/unit/time.test.ts`: переходы через полночь, пояса `Europe/Moscow` и
  `Asia/Vladivostok`.
- [ ] T022 [P] Создать генератор идентификаторов ULID в `worker/lib/ids.ts`.
- [ ] T023 [P] Создать обёртку Telegram SDK в `src/lib/telegram.ts` и подключить провайдер в
  `src/app/layout.tsx`:
  - инициализация `@tma.js/sdk-react`, получение `initDataRaw`;
  - привязка параметров темы Telegram к CSS-переменным и safe area;
  - кнопки «Назад» и главная кнопка;
  - вне Telegram — понятная заглушка «Откройте приложение из бота».
- [ ] T024 [P] Создать клиент API в `src/lib/api-client.ts`: заголовок `Authorization: tma <initData>`,
  разбор ответа через схемы `@shared/schemas`, перевод ошибок в русский текст для показа.
- [ ] T025 [P] Реализовать команды владельца в `worker/bot/handlers/admin.ts`: `/allow <id|@username>`,
  `/deny <id|@username>`, `/testers`; только для `OWNER_TELEGRAM_ID` (FR-003a).
- [ ] T026 Создать скрипт настройки бота `scripts/bot-setup.ts`:
  - `setWebhook(<MINI_APP_URL>/bot/webhook, secret_token, allowed_updates: ["message",
    "callback_query"])`;
  - `setMyCommands` (`start`, `today`, `settings`);
  - `setChatMenuButton` типа `web_app` на `MINI_APP_URL`.
- [ ] T027 Проверить, принимает ли Whisper голосовые Telegram (spike):
  - прогнать реальный OGG/Opus-голосовой из Telegram через `@cf/openai/whisper-large-v3-turbo`
    (`wrangler dev --remote`, одноразовый `scripts/spike-stt.ts`);
  - если не принимает — проверить `@cf/deepgram/nova-3` с русским;
  - записать решение в раздел R4 файла `specs/001-voice-life-planner/research.md`.

**Checkpoint**: база, авторизация, доступ и каркасы готовы — можно начинать истории

---

## Phase 3: User Story 1 — Одна голосовая раскладывается на записи (Priority: P1) 🎯 MVP

**Goal**: голосовое или текст в чате либо в Mini App превращаются в задачи, встречи, приёмы пищи и
дневник. Итог приходит с кнопками «Отменить» и «Открыть и исправить».

**Independent Test**: отправить боту голосовое из примера спецификации и проверить итог (4 записи,
верные типы, дата и время). «Отменить» удаляет записи. Та же фраза из Mini App даёт тот же
результат (quickstart, сценарии 1–6).

### Tests for User Story 1

- [ ] T028 [P] [US1] Написать тесты правил в `tests/unit/rules.test.ts`:
  - «в 10» → ближайшие 10:00 в будущем;
  - встреча без даты или времени → черновик «Уточните дату/время»;
  - встреча сегодня с прошедшим временем → черновик «Время уже прошло — может быть, завтра?»;
  - тип приёма пищи по времени: «05:00–10:59 — завтрак; 11:00–15:59 — обед; 16:00–21:59 — ужин;
    остальное — перекус»;
  - суммы КБЖУ «округление до целых ккал и 1 знака для БЖУ»;
  - пустой `records` → `empty`;
  - значения вне диапазонов → черновик и обрезка.
- [ ] T029 [P] [US1] Написать тесты схемы `ParseResult` в `tests/unit/ai-schemas.test.ts`: валидные
  ответы всех четырёх типов проходят; лишние поля, неверные даты (`YYYY-MM-DD`) и время (`HH:mm`)
  отклоняются.

### Implementation for User Story 1

- [ ] T030 [P] [US1] Описать zod-схему `ParseResult` из `contracts/ai-output.md` в
  `worker/pipeline/schemas.ts` и экспорт JSON Schema через `z.toJSONSchema` для `response_format`.
- [ ] T031 [US1] Реализовать `applyRules(parse, { now, tz, messageDate })` в
  `worker/pipeline/rules.ts` по таблице постобработки из `contracts/ai-output.md`. Ограничения из
  `data-model.md`:
  - `title` «1–200 символов», `location` «до 300 символов», `duration_min` «5–1440»;
  - `grams` «1–3000», `kcal` «0–5000», БЖУ «0–500», название блюда «1–120 символов»;
  - текст дневника «1–5000 символов».

  Тесты T028 проходят.
- [ ] T032 [P] [US1] Реализовать распознавание речи в `worker/ai/stt.ts`:
  - `transcribe(bytes, mime)` через модель, выбранную в T027;
  - параметры `language: "ru"`, `vad_filter: true`, base64 через `Buffer`;
  - возвращает `{ text, neurons }` (≈ 46,63 нейрона за минуту).
- [ ] T033 [P] [US1] Реализовать разбор текста в `worker/ai/parse.ts`:
  - `parseText(text, ctx)` на `@cf/meta/llama-3.3-70b-instruct-fp8-fast` с
    `response_format: { type: "json_schema" }`;
  - системный промпт на русском: текущие дата, время, день недели и пояс; абсолютные даты; не
    выдумывать; `uncertain` с причиной;
  - валидация zod, один повтор, оценка нейронов.
- [ ] T034 [P] [US1] Реализовать учёт лимитов в `worker/ai/budget.ts`:
  - `checkUserQuota` и `incrementUsage` по `usage_daily` («50 голосовых и текстовых сообщений и
    20 фото» в сутки);
  - `canSpend` и `recordSpend` по `ai_budget_daily` с порогом 9 000 нейронов в сутки UTC;
  - `delayUntilReset()` — секунды до 00:05 UTC.
- [ ] T035 [US1] Реализовать репозиторий вводов в `worker/db/repo/inputs.ts`:
  - `create`, `get` (только свои), `setStatus` по переходам из `data-model.md`;
  - `undo`: batch-удаление всех записей с `input_id`, только если «≤ 24 ч от `received_at`»,
    иначе `undo_expired`;
  - `markRetry` для `failed` с живым источником.
- [ ] T036 [US1] Реализовать сохранение записей в `worker/db/repo/records.ts`:
  `insertParsedRecords(db, userId, inputId, normalized)` одним `db.batch` (задачи, встречи,
  приёмы пищи с позициями и суммами, дневник). Возвращает id созданных записей и черновиков.
- [ ] T037 [US1] Реализовать обработчик очереди в `worker/pipeline/consumer.ts`:
  1. Загрузить ввод, статус `processing`.
  2. Проверить бюджет: нет бюджета → `deferred`, `message.retry({ delaySeconds:
     min(delayUntilReset(), 43200) })` (задержка Queues — не больше 12 ч; после неё бюджет
     проверяется снова), уведомление — только при первой отсрочке.
  3. Получить медиа: `getFile` + скачивание по `file_id` или `MEDIA.get(key)`.
  4. STT → `parseText` → `applyRules` → `insertParsedRecords`.
  5. Статус `done` или `empty`; при исключении — `failed` с `error_code`.
  6. После успеха удалить ключ KV и обнулить `source_ref` (FR-032).
  7. Вызвать уведомление.
- [ ] T038 [US1] Реализовать итог в `worker/pipeline/notify.ts`:
  - `formatSummary(result)` по образцу из `contracts/bot.md` (маркеры типов, строка «⚠️ Уточните»
    для черновиков, HTML-экранирование);
  - для `channel = chat` — ответ на исходное сообщение с кнопками «Отменить» (`undo:<id>`) и
    «Открыть и исправить» (`web_app` `…/?input=<id>`), сохранить `chat_message_id`;
  - тексты для `empty`, `failed` (кнопка `retry:<id>`) и `deferred`.
- [ ] T039 [US1] Реализовать приём сообщений в `worker/bot/handlers/messages.ts`:
  - `voice` ≤ 120 с → реакция, `input` (`channel=chat`, `kind=voice`, `source_ref=file_id`),
    проверка квоты, `INPUTS.send`;
  - `voice` > 120 с → «Голосовое длиннее 2 минут — разбейте, пожалуйста, на части»;
  - текст (не команда) → `kind=text`;
  - прочие типы → подсказка из `contracts/bot.md`.
- [ ] T040 [US1] Реализовать `/start` в `worker/bot/handlers/start.ts`:
  - приветствие, что умеет бот, пример фразы, кнопка `web_app` «Открыть EZ Planner»;
  - если пояс не подтверждён — кнопки «Москва (UTC+3)» (`tz:Europe/Moscow`) и «Определить в
    приложении» (`web_app`).
- [ ] T041 [US1] Реализовать кнопки в `worker/bot/handlers/callbacks.ts`:
  - `undo:<id>` → отмена, правка итога в «Отменено ✖» без кнопок; после 24 ч — «Отменить можно
    в течение 24 часов»;
  - `retry:<id>` → `markRetry` + `INPUTS.send`, всплывающее «Повторяю…»;
  - `tz:<IANA>` → сохранить пояс;
  - чужой ввод → «Недоступно».
- [ ] T042 [US1] Реализовать ввод из Mini App в `worker/api/inputs.ts` (по `contracts/api.md`):
  - `POST /api/inputs`:
    - голос: `multipart`, `audio/wav`, ≤ 120 с по размеру WAV 16 кГц моно → 413;
    - текст: JSON, «1–4000 символов»;
    - ключ KV `media:<userId>:<inputId>` с `expirationTtl: 86400`;
  - `GET /api/inputs/:id`;
  - `POST /api/inputs/:id/undo`, `POST /api/inputs/:id/retry`;
  - квоты → 429 `rate_limited`.
- [ ] T043 [P] [US1] Реализовать запись голоса в `src/lib/wav-recorder.ts`: `getUserMedia` → Web
  Audio → понижение до 16 кГц моно → WAV PCM 16 бит, автостоп на 120 с, отдаёт `Blob` и
  длительность.
- [ ] T044 [US1] Создать кнопку записи `src/components/input/VoiceRecorder.tsx`:
  - запись с таймером и отменой, загрузка через `api-client`;
  - при отказе в микрофоне — объяснение, как разрешить доступ, и переход к тексту (US1-10).
- [ ] T045 [P] [US1] Создать поле текстового ввода `src/components/input/TextInput.tsx` (1–4000
  символов, отправка в `POST /api/inputs`).
- [ ] T046 [US1] Создать лист результата `src/components/input/InputResultSheet.tsx`:
  - опрос `GET /api/inputs/:id` раз в секунду до 30 с;
  - показ итога и черновиков, кнопки «Отменить» и «Исправить»;
  - состояния `empty`, `failed` (кнопка «Повторить»), `deferred`, `undone`.
- [ ] T047 [US1] Собрать минимальный главный экран в `src/app/page.tsx`: панель ввода (голос,
  текст) и открытие `InputResultSheet` по параметру `?input=<id>`. Полная лента — в US2.

**Checkpoint**: MVP — голосовое или текст (чат и Mini App) → записи → итог с отменой

---

## Phase 4: User Story 2 — Мой день в Mini App (Priority: P2)

**Goal**: лента дня с блоками встреч, задач, питания и дневника; правка, отметка, удаление,
ручное создание, переход по дням.

**Independent Test**: создать записи вручную через формы (без ИИ), проверить ленту, отметку задачи,
правку, удаление, перенос задач без даты и переход на другие дни (quickstart, сценарий 7).

### Tests for User Story 2

- [ ] T048 [P] [US2] Написать тесты ленты дня в `tests/unit/day-feed.test.ts`:
  - флаги задач `carried_over` и `overdue` по правилам `data-model.md`;
  - суммы калорий и БЖУ за день;
  - признак `overlaps` у пересекающихся встреч;
  - `goal` при заданной норме.

### Implementation for User Story 2

- [ ] T049 [US2] Реализовать выборку дня в `worker/db/repo/day.ts`: `getDay(db, userId, date,
  today)` — записи дня, задачи с флагами «перенесена» и «просрочена», пересечения встреч, итоги.
  Чистые функции выделены для T048.
- [ ] T050 [US2] Реализовать `GET /api/day?date=YYYY-MM-DD` в `worker/api/day.ts` по формату
  ответа из `contracts/api.md` (по умолчанию — сегодня в поясе пользователя).
- [ ] T051 [US2] Реализовать CRUD записей в `worker/api/records.ts` (по `contracts/api.md`):
  - все запросы с фильтром `user_id`, чужие и несуществующие записи → 404;
  - ограничения: `title` «1–200 символов», `location` «до 300 символов», `duration_min`
    «5–1440», `grams` «1–3000», `kcal` «0–5000», БЖУ «0–500», название блюда «1–120 символов»,
    текст дневника «1–5000 символов»;
  - встречи принимают локальные `date` и `time` и переводят в UTC;
  - итоги приёма пищи пересчитываются при изменении позиций;
  - черновик с заполненными полями становится обычной записью.
- [ ] T052 [P] [US2] Создать навигацию по дням `src/components/day/DayNav.tsx`: «вчера» и
  «завтра», выбор даты в календаре, синхронизация с `?date=`, кнопка «Сегодня».
- [ ] T053 [P] [US2] Создать блок встреч `src/components/day/EventsBlock.tsx`: сортировка по
  времени в поясе пользователя, пометки черновика и пересечения, открытие формы.
- [ ] T054 [P] [US2] Создать блок задач `src/components/day/TasksBlock.tsx`: отметка с
  оптимистичным обновлением, пометки «перенесена», «просрочена» и «черновик».
- [ ] T055 [P] [US2] Создать блок питания `src/components/day/MealsBlock.tsx`: приёмы пищи с
  позициями, сумма ккал и БЖУ за день, пометка «≈ оценка», прогресс к норме при `goal`.
- [ ] T056 [P] [US2] Создать блок дневника `src/components/day/DiaryBlock.tsx`.
- [ ] T057 [US2] Создать формы записей в `src/components/records/`: `TaskForm.tsx`, `EventForm.tsx`,
  `MealForm.tsx` (с позициями), `DiaryForm.tsx` и общий `RecordSheet.tsx`. Создание, правка,
  удаление с подтверждением; ограничения полей как в T051.
- [ ] T058 [US2] Собрать ленту дня в `src/app/page.tsx`:
  - блоки T052–T056, кнопка «Добавить» с выбором типа, панель ввода из US1;
  - кнопка «Назад» Telegram закрывает листы;
  - вёрстка от 320 px.
- [ ] T059 [US2] Реализовать `/today` в `worker/bot/handlers/today.ts`: записи сегодняшнего дня
  текстом — запасной путь, когда Mini App не открывается.

**Checkpoint**: US1 и US2 работают и проверяются независимо

---

## Phase 5: User Story 3 — Фото еды — калории (Priority: P3)

**Goal**: фото еды из чата или Mini App → приём пищи с блюдами, граммами, калориями и БЖУ.

**Independent Test**: отправить фото тарелки с подписью и без, фото без еды; сверить с эталоном
на оценочном наборе (quickstart, сценарии 8–9; SC-004).

**Зависимость**: использует конвейер из US1 (T035–T038).

### Tests for User Story 3

- [ ] T060 [P] [US3] Написать тесты в `tests/unit/food.test.ts`:
  - валидация `FoodResult`;
  - `confidence < 0,6` → `low_confidence = 1`;
  - `isFood = false` или пустой `items` → `empty`;
  - тип приёма из подписи, без подписи — по времени;
  - `is_estimate = 1`.

### Implementation for User Story 3

- [ ] T061 [P] [US3] Добавить zod-схему `FoodResult` из `contracts/ai-output.md` в
  `worker/pipeline/schemas.ts` и правила постобработки фото в `worker/pipeline/rules.ts`.
- [ ] T062 [US3] Реализовать распознавание еды в `worker/ai/food.ts`:
  - `recognizeFood(imageBytes, caption)` на `@cf/meta/llama-4-scout-17b-16e-instruct`;
  - промпт на русском: русские названия, граммы на порцию, типичные рецептуры российской кухни;
  - разбор JSON, валидация zod, один повтор, оценка нейронов.
- [ ] T063 [US3] Добавить ветку `kind = photo` в `worker/pipeline/consumer.ts`: самое крупное фото
  по `file_id` или из KV → `recognizeFood` → приём пищи с `is_estimate = 1`. Итог — через
  `notify.ts` («Не нашёл еды на фото» для `empty`).
- [ ] T064 [US3] Добавить приём `photo` в `worker/bot/handlers/messages.ts`: самый крупный размер,
  подпись → `text`, квота фото.
- [ ] T065 [US3] Добавить `kind=photo` в `POST /api/inputs` в `worker/api/inputs.ts`: `image/jpeg`
  | `image/png` | `image/webp`, «≤ 10 МБ» → иначе 413 или 415, необязательный `caption`.
- [ ] T066 [P] [US3] Реализовать уменьшение фото на клиенте в `src/lib/image-resize.ts`: до
  1600 px по длинной стороне, JPEG качества 0,85.
- [ ] T067 [US3] Создать выбор фото `src/components/input/PhotoPicker.tsx`: `<input type="file"
  accept="image/*" capture="environment">`, подпись, загрузка, открытие `InputResultSheet`.
- [ ] T068 [US3] Собрать оценочный набор фото: `tests/eval/photos/` (30 фото типовых блюд) и
  `tests/eval/photos/expected.json`. Режим `--set photos` в `scripts/eval.ts` печатает долю фото с
  ошибкой калорий ≤ 25 % (цель SC-004 ≥ 70 %).

**Checkpoint**: фото еды работает из чата и из Mini App

---

## Phase 6: User Story 4 — Напоминания о встречах (Priority: P4)

**Goal**: бот напоминает о встречах за заданный интервал и учитывает правки и удаление.

**Independent Test**: создать встречу через 20 минут, изменить время, удалить — напоминание
приходит только по актуальному времени и только для существующей встречи (quickstart, сценарии
10–11).

### Tests for User Story 4

- [ ] T069 [P] [US4] Написать тесты в `tests/unit/reminders.test.ts`:
  - `send_at = start_at − reminder_minutes`;
  - нет напоминания для черновиков, встреч без `start_at` и при `reminder_minutes = 0`;
  - прошедший `send_at` при будущей встрече → отправка в ближайший запуск.

### Implementation for User Story 4

- [ ] T070 [US4] Реализовать планирование в `worker/reminders/schedule.ts`:
  `upsertReminderForEvent(db, event, user)`, `cancelForEvent`, `recomputeForUser` (при смене
  `reminder_minutes`).
- [ ] T071 [US4] Вызвать планирование при изменении встреч:
  - в `worker/db/repo/records.ts` — при вставке из конвейера;
  - в `worker/api/records.ts` — при `POST`/`PATCH` встреч и подтверждении черновика;
  - удаление — каскадом.
- [ ] T072 [US4] Реализовать отправку в `worker/reminders/send.ts` (обработчик `scheduled`):
  - выбрать `status = scheduled AND send_at ≤ now` (до 40 за запуск — лимит подзапросов Free);
  - отправить текст по `contracts/bot.md` с кнопкой «Открыть день», отметить `sent`;
  - при 403 (бот заблокирован) — `cancelled`.

**Checkpoint**: напоминания работают

---

## Phase 7: User Story 5 — Настройки и контроль над данными (Priority: P5)

**Goal**: часовой пояс, интервал напоминаний, норма калорий, сводка данных и удаление всего.

**Independent Test**: изменить каждую настройку и увидеть эффект; удалить все данные — ни одной
записи не осталось, бот начинает со старта (quickstart, сценарий 12).

### Implementation for User Story 5

- [ ] T073 [US5] Реализовать профиль в `worker/api/me.ts` (по `contracts/api.md`):
  - `GET /api/me`;
  - `PATCH /api/me/settings`: `timezone` — валидный IANA; `reminderMinutes` — «одно из 0 (выкл),
    5, 15, 30, 60»; `calorieGoal` — «500–10000 ккал» или `null`; вызывает `recomputeForUser`;
  - `GET /api/me/data`;
  - `DELETE /api/me` с `{ "confirm": "УДАЛИТЬ" }`: batch-удаление всех строк пользователя, кроме
    `testers`, и ключей KV с префиксом `media:<userId>:`.
- [ ] T074 [US5] Создать экран настроек `src/app/settings/page.tsx`:
  - часовой пояс с кнопкой «Определить автоматически», интервал напоминаний, норма калорий;
  - раздел «Мои данные» — сводка и что не хранится;
  - «Удалить все мои данные» с подтверждением вводом слова.
- [ ] T075 [US5] При первом открытии Mini App с `timezoneConfirmed = false` отправлять
  `Intl.DateTimeFormat().resolvedOptions().timeZone` в `PATCH /api/me/settings`. Логика — в
  `src/lib/telegram.ts` или хуке `src/hooks/useBootstrap.ts`.
- [ ] T076 [US5] Реализовать `/settings` в `worker/bot/handlers/settings.ts`: текущие настройки
  текстом и кнопка `web_app` на `/settings`.

**Checkpoint**: все пять историй работают

---

## Phase 8: Дизайн по DESIGN.md (Конституция, принцип III)

**Purpose**: заменить функциональную вёрстку на собственный стиль

**⚠️ Блокер**: владелец продукта выбирает стиль из VoltAgent/awesome-design-md

- [ ] T077 Скопировать выбранный `DESIGN.md` в корень репозитория с шапкой-указанием источника и
  лицензии (MIT); удалить из него упоминания брендов, которые не используются.
- [ ] T078 Перенести токены DESIGN.md в `src/app/globals.css`: цвета в oklch, типографика, отступы,
  радиусы, тени. Согласовать со светлой и тёмной темой Telegram.
- [ ] T079 Стилизовать компоненты `src/components/ui/` под спецификации компонентов DESIGN.md;
  убрать внешний вид shadcn/ui по умолчанию.
- [ ] T080 Применить стиль к `src/components/day/`, `src/components/input/`,
  `src/components/records/` и `src/app/settings/page.tsx`. Проверить ширину 320 px и safe area.
- [ ] T081 Проверить контраст (WCAG AA) и размеры зон касания (≥ 44 px) в обеих темах; исправить
  найденное.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [ ] T082 [P] Собрать оценочный набор фраз `tests/eval/phrases.jsonl` (50 фраз с ожидаемыми
  записями, включая относительные даты и неоднозначности). Режим `--set phrases` в
  `scripts/eval.ts` печатает долю верных типов (цель SC-003 ≥ 85 %) и дат и времени встреч
  (≥ 90 %).
- [ ] T083 [P] Добавить очистку в `worker/reminders/send.ts` (тот же cron): у вводов `failed`
  старше 24 ч обнулять `source_ref`, помечать повтор недоступным.
- [ ] T084 [P] Добавить в `/start` и `/today` подсказку для случая, когда Mini App не открывается:
  голосовые и текст в чате работают без Mini App. Способы обхода ограничений не упоминаются
  (конституция 1.1.0).
- [ ] T085 [P] Переписать `README.md` под EZ Planner: что это, как запустить (ссылка на
  quickstart), где спецификации.
- [ ] T086 Провести проверку безопасности:
  - все запросы в `worker/db/repo/` фильтруют по `user_id`;
  - в `out/` нет секретов (поиск по шаблону токена бота);
  - вебхук отклоняет запросы без секрета;
  - поддельный `initData` → 401 (quickstart, сценарий 13).
- [ ] T087 Проверить нагрузку на CPU через `wrangler tail`: нет ошибок превышения 10 мс на Free. Если
  есть — записать в `research.md` и предложить владельцу Workers Paid.
- [ ] T088 Пройти сценарии 1–13 из `quickstart.md` на staging и записать результаты в
  `specs/001-voice-life-planner/checklists/acceptance.md`.

---

## Покрытие требований

| Требования | Задачи |
|---|---|
| FR-001, FR-033 (вход через Telegram, только свои данные) | T015, T017, T018, T051, T086 |
| FR-002 (старт и часовой пояс) | T040, T041, T075 |
| FR-003, FR-003a (закрытый тест) | T016, T017, T019, T025 |
| FR-004, FR-004a (голос и текст в чате и Mini App) | T039, T042–T047 |
| FR-005–FR-009 (разбор, даты, черновики, сохранение сразу) | T021, T028–T033, T036, T037 |
| FR-010–FR-012 (итог, отмена, пусто) | T035, T038, T041, T046 |
| FR-013–FR-016, FR-013a (фото еды) | T060–T067 |
| FR-017–FR-025 (Mini App: лента, дни, правка, перенос, тема) | T023, T048–T058, T078–T080 |
| FR-026–FR-028 (напоминания) | T069–T072 |
| FR-029–FR-031 (настройки, данные, удаление) | T073–T076 |
| FR-032 (медиа не хранятся) | T037, T042, T083 |
| FR-034, FR-034a (лимиты и бюджет) | T034, T037, T039, T042 |
| SC-002, SC-006, SC-007 | T086, T088 |
| SC-003, SC-004 | T082, T068 |
| SC-008 (понятность итога) | T088: опрос тестировщиков после первой голосовой |

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** — сразу.
- **Foundational (Phase 2)** — после Setup, блокирует все истории. T027 (spike) блокирует T032.
- **US1 (Phase 3)** — после Foundational.
- **US2 (Phase 4)** — после Foundational. Проверяется независимо через ручные формы; итоговую
  панель ввода берёт из US1 (T044–T047).
- **US3 (Phase 5)** — после US1: использует конвейер T035–T038.
- **US4 (Phase 6)** — после Foundational. Для проверки нужны встречи из US1 или US2.
- **US5 (Phase 7)** — после Foundational; `recomputeForUser` из US4 (T070) вызывается, если US4
  готова.
- **Design (Phase 8)** — после US2 (есть экраны для стилизации) и выбора DESIGN.md владельцем.
- **Polish (Phase 9)** — после нужных историй.

### Within Each User Story

- Тесты пишутся раньше модуля и сначала падают.
- Схемы → правила и репозитории → конвейер и API → бот и интерфейс.
- История завершается и проверяется до перехода к следующему приоритету.

### Parallel Opportunities

- Setup: T004–T009 параллельно после T001–T003.
- Foundational: T013, T014, T016, T021–T025 параллельно; T015 после T014, T017 после T016.
- US1: T028–T030, T032–T034, T043, T045 параллельно; затем T031, T035–T042, T044, T046–T047.
- US2: T048, T052–T056 параллельно; T049 → T050, T051 → T057 → T058.
- US3: T060, T061, T066 параллельно.
- Разные истории после Foundational можно вести параллельно, кроме US3 (ждёт US1).

---

## Parallel Example: User Story 1

```bash
# Тесты и независимые модули US1 одновременно:
Task: "Тесты правил в tests/unit/rules.test.ts"
Task: "Тесты схемы ParseResult в tests/unit/ai-schemas.test.ts"
Task: "Схема ParseResult в worker/pipeline/schemas.ts"
Task: "Распознавание речи в worker/ai/stt.ts"
Task: "Разбор текста в worker/ai/parse.ts"
Task: "Учёт лимитов в worker/ai/budget.ts"
Task: "Запись голоса в src/lib/wav-recorder.ts"
Task: "Текстовый ввод в src/components/input/TextInput.tsx"
```

## Parallel Example: User Story 2

```bash
Task: "Тесты ленты в tests/unit/day-feed.test.ts"
Task: "DayNav в src/components/day/DayNav.tsx"
Task: "EventsBlock в src/components/day/EventsBlock.tsx"
Task: "TasksBlock в src/components/day/TasksBlock.tsx"
Task: "MealsBlock в src/components/day/MealsBlock.tsx"
Task: "DiaryBlock в src/components/day/DiaryBlock.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 и Phase 2, включая spike T027.
2. Phase 3 (US1).
3. **Стоп и проверка**: сценарии 1–6 из quickstart на staging-боте.
4. Показать тестировщикам: голосовое → итог → отмена уже полезны без ленты.

### Incremental Delivery

1. Setup + Foundational → основа.
2. US1 → проверка → демо (MVP).
3. US2 → лента дня → демо.
4. Выбор DESIGN.md владельцем → Phase 8, можно сразу после US2.
5. US3 → фото еды → оценка SC-004.
6. US4 → напоминания. US5 → настройки и данные.
7. Polish → прогон quickstart и оценочных наборов.

---

## Notes

- [P] — разные файлы и нет зависимостей от незавершённых задач.
- Метки [USn] связывают задачу с историей из spec.md.
- Коммит после каждой задачи или логической группы; CI должен быть зелёным (конституция VI).
- Next.js 16 в шаблоне может отличаться от привычного: перед кодом читать
  `node_modules/next/dist/docs/` (см. `AGENTS.md`).
