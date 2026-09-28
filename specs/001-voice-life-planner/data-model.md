# Data Model: SGX PLANNER MVP (001-voice-life-planner)

**Хранилище**: Cloudflare D1 (SQLite), схема в Drizzle ORM (`worker/db/schema.ts`), миграции в
`migrations/`. Временные медиа — KV с TTL 24 часа (не таблица).

**Общие соглашения**
- Идентификаторы записей — строки ULID (сортируются по времени создания).
- Моменты времени (`*_at`) — UTC в ISO 8601. Локальные даты (`date`) — `YYYY-MM-DD` в часовом поясе
  пользователя на момент создания или правки.
- У каждой пользовательской записи есть `user_id`; каждый запрос фильтрует по нему (FR-033).
- `source`: `voice` | `text` | `photo` | `manual` — откуда появилась запись.
- Черновик: `is_draft = 1` и `draft_reason` — что уточнить (FR-008). Черновик виден в ленте с
  пометкой и подтверждается правкой.

## Сущности

### users — пользователь

| Поле | Тип | Правила |
|---|---|---|
| `id` | integer PK | Telegram user id |
| `first_name` | text | из Telegram, для обращения |
| `username` | text, null | из Telegram, без `@`, в нижнем регистре |
| `timezone` | text | IANA, по умолчанию `Europe/Moscow` |
| `timezone_confirmed` | integer 0/1 | 1 после подтверждения или определения по устройству |
| `reminder_minutes` | integer | одно из 0 (выкл), 5, 15, 30, 60; по умолчанию 15 |
| `calorie_goal` | integer, null | 500–10000 ккал |
| `created_at` / `last_seen_at` | text | |

Создаётся при первом обращении тестировщика к боту или Mini App.

### testers — список доступа (FR-003, FR-003a)

| Поле | Тип | Правила |
|---|---|---|
| `id` | integer PK autoincrement | |
| `telegram_id` | integer, null, unique | |
| `username` | text, null, unique | без `@`, нижний регистр |
| `added_at` | text | |

Хотя бы одно из `telegram_id` / `username` задано. При первом сообщении пользователя с совпавшим
`username` в строку дописывается `telegram_id`. Владелец задаётся переменной окружения
`OWNER_TELEGRAM_ID` и всегда имеет доступ.

### inputs — входящее сообщение

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | ULID |
| `user_id` | integer FK → users | |
| `channel` | text | `chat` \| `app` |
| `kind` | text | `voice` \| `text` \| `photo` |
| `status` | text | см. переходы ниже |
| `source_ref` | text, null | `file_id` Telegram или ключ KV; обнуляется после успеха (FR-032) |
| `text` | text, null | текст сообщения, подпись к фото или распознанная речь |
| `duration_sec` | integer, null | для голоса; > 120 — отказ (FR-004) |
| `chat_message_id` | integer, null | id итогового сообщения бота — для правки после отмены |
| `error_code` | text, null | `stt_failed`, `parse_failed`, `vision_failed`, `not_food`, `too_long` |
| `neurons` | integer | оценка потраченного бюджета ИИ |
| `received_at` / `processed_at` / `undone_at` | text | |

**Переходы `status`**

```text
queued ──► processing ──► done ──► undone          (кнопка «Отменить», ≤ 24 ч от received_at)
   ▲           │
   │           ├──► empty                            (ничего не найдено, FR-012 / FR-015)
   │           └──► failed ──► queued                (кнопка «Повторить», ≤ 24 ч; иначе источник истёк)
   └── deferred ◄── queued                           (исчерпан дневной бюджет ИИ, FR-034a)
```

### tasks — задача

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `user_id` | integer FK | |
| `input_id` | text FK → inputs, null | null для ручных |
| `title` | text | 1–200 символов |
| `due_date` | text, null | локальная дата; null — задача без даты |
| `status` | text | `open` \| `done` |
| `completed_at` | text, null | |
| `is_draft` / `draft_reason` | integer / text | |
| `source`, `created_at`, `updated_at` | | |

Правила показа в ленте (FR-024):
- в ленте дня D — задачи с `due_date = D`;
- в ленте «Сегодня» дополнительно:
  - открытые задачи без даты, созданные раньше сегодняшнего дня, с пометкой «перенесена»;
  - открытые задачи с `due_date` < сегодня с пометкой «просрочена»;
  - задачи без даты, созданные сегодня.

### events — встреча

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `user_id`, `input_id` | | как у задач |
| `title` | text | 1–200 символов |
| `date` | text | локальная дата встречи |
| `start_at` | text, null | UTC; null только у черновика без времени |
| `duration_min` | integer, null | 5–1440 |
| `location` | text, null | место или ссылка, до 300 символов |
| `is_draft` / `draft_reason` | | |
| `source`, `created_at`, `updated_at` | | |

Пересечение по времени с другой встречей не запрещено, но помечается в ленте.

### meals — приём пищи

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `user_id`, `input_id` | | |
| `date` | text | локальная дата |
| `eaten_at` | text | UTC |
| `meal_type` | text | `breakfast` \| `lunch` \| `dinner` \| `snack` |
| `kcal`, `protein_g`, `fat_g`, `carbs_g` | real | сумма позиций; пересчитывается при любом изменении позиций |
| `is_estimate` | integer 0/1 | 1 для значений от ИИ (FR-016) |
| `is_draft` / `draft_reason` | | |
| `source`, `created_at`, `updated_at` | | |

Тип без подписи определяется по местному времени (FR-014):
- 05:00–10:59 — завтрак;
- 11:00–15:59 — обед;
- 16:00–21:59 — ужин;
- остальное — перекус.

### meal_items — позиция блюда

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `meal_id` | text FK → meals, on delete cascade | |
| `position` | integer | порядок в списке |
| `name` | text | 1–120 символов |
| `grams` | real | 1–3000 |
| `kcal` | real | 0–5000 |
| `protein_g`, `fat_g`, `carbs_g` | real | 0–500 |
| `low_confidence` | integer 0/1 | помечается в итоге и ленте (US3-4) |

### diary_entries — запись в дневнике

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `user_id`, `input_id` | | |
| `date` | text | локальная дата |
| `written_at` | text | UTC |
| `text` | text | 1–5000 символов |
| `source`, `created_at`, `updated_at` | | |

### reminders — напоминание

| Поле | Тип | Правила |
|---|---|---|
| `id` | text PK | |
| `event_id` | text FK → events, unique, on delete cascade | одно активное на встречу |
| `user_id` | integer | |
| `send_at` | text | UTC = `start_at` − `reminder_minutes` |
| `status` | text | `scheduled` \| `sent` \| `cancelled` |
| `sent_at` | text, null | |

Правила (FR-026–FR-028):
- Напоминание создаётся или пересчитывается при создании и правке встречи и при смене
  `reminder_minutes`.
- Не создаётся для черновиков, встреч без `start_at` и при `reminder_minutes = 0`.
- Если `send_at` уже прошло, а встреча ещё впереди, напоминание уходит в ближайший запуск cron.
- При удалении встречи напоминание удаляется каскадом.

### usage_daily и ai_budget_daily — учёт лимитов (FR-034, FR-034a)

| Таблица | Поля |
|---|---|
| `usage_daily` | `user_id`, `day` (UTC-дата), `voice_text_count`, `photo_count`; PK (`user_id`, `day`) |
| `ai_budget_daily` | `day` (UTC-дата) PK, `neurons` |

## Связи

```text
users 1─* inputs 1─* {tasks, events, meals, diary_entries}   (input_id null у ручных записей)
meals 1─* meal_items
events 1─0..1 reminders
users 1─* usage_daily
```

Отмена (`inputs.status → undone`) удаляет все записи с этим `input_id` одной транзакцией
(`batch` в D1). «Удалить все мои данные» (FR-031) удаляет все строки пользователя во всех таблицах,
кроме `testers`, и ключи KV с его префиксом.
