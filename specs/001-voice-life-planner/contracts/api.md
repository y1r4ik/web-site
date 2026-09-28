# Contract: HTTP API Mini App

**Базовый путь**: `/api` на том же домене, что и Mini App. **Формат**: JSON (UTF-8), кроме загрузки
медиа (`multipart/form-data`). Схемы запросов и ответов описываются `zod` в `src/shared/schemas.ts`
и используются и клиентом, и сервером.

## Авторизация

Каждый запрос содержит заголовок:

```http
Authorization: tma <initData>
```

Сервер проверяет подпись `initData` ключом из токена бота и `auth_date` не старше 24 часов.
Пользователь определяется только по проверенному `user.id`. Если пользователь не владелец и не в
списке тестировщиков, сервер отвечает `403 forbidden_not_tester`.

## Ошибки

```json
{ "error": { "code": "validation_failed", "message": "Название не может быть пустым" } }
```

| HTTP | `code` | Когда |
|---|---|---|
| 401 | `unauthorized` | нет или неверна подпись `initData`, истёк `auth_date` |
| 403 | `forbidden_not_tester` | пользователь не в списке доступа |
| 404 | `not_found` | записи нет или она чужая (чужие записи неотличимы от несуществующих) |
| 409 | `undo_expired` | прошло больше 24 часов, отмена или повтор недоступны |
| 413 | `payload_too_large` | голос длиннее 120 с или файл больше 10 МБ |
| 415 | `unsupported_media` | неподдерживаемый тип файла |
| 422 | `validation_failed` | тело не прошло схему |
| 429 | `rate_limited` | исчерпан личный дневной лимит (FR-034) |

`message` — готовый текст на русском для показа пользователю.

## Профиль и настройки

### `GET /api/me`

```json
{
  "user": { "id": 123, "firstName": "Ярослав", "timezone": "Asia/Vladivostok",
            "timezoneConfirmed": true, "reminderMinutes": 15, "calorieGoal": 2200 },
  "limits": { "voiceTextLeft": 47, "photoLeft": 20 }
}
```

### `PATCH /api/me/settings`

Тело — любые из полей: `timezone` (IANA), `reminderMinutes` (0|5|15|30|60), `calorieGoal`
(500–10000 или `null`). Ответ — как `GET /api/me`. Смена `reminderMinutes` пересчитывает все
будущие напоминания.

### `GET /api/me/data` (FR-030)

Сводка хранимых данных: количество записей каждого типа, дата первой записи, настройки и явный
список того, что **не** хранится (исходные голосовые и фото после обработки).

### `DELETE /api/me` (FR-031)

Тело: `{ "confirm": "УДАЛИТЬ" }`. Удаляет все данные пользователя. Ответ `204`.

## Лента дня

### `GET /api/day?date=YYYY-MM-DD`

`date` — локальная дата пользователя; по умолчанию сегодня.

```json
{
  "date": "2026-09-30",
  "isToday": true,
  "events": [ { "id": "01J…", "title": "Созвон с Лёшей", "date": "2026-09-30",
                "startAt": "2026-09-30T00:00:00Z", "durationMin": null, "location": null,
                "isDraft": false, "draftReason": null, "overlaps": false, "source": "voice" } ],
  "tasks":  [ { "id": "01J…", "title": "Купить молоко", "dueDate": null, "status": "open",
                "flag": "carried_over", "isDraft": false, "source": "voice" } ],
  "meals":  [ { "id": "01J…", "mealType": "lunch", "eatenAt": "…", "kcal": 520,
                "proteinG": 18, "fatG": 17, "carbsG": 70, "isEstimate": true, "isDraft": false,
                "items": [ { "id": "01J…", "name": "Борщ", "grams": 300, "kcal": 180,
                             "proteinG": 7, "fatG": 8, "carbsG": 20, "lowConfidence": false } ] } ],
  "diary":  [ { "id": "01J…", "writtenAt": "…", "text": "День был тяжёлый, но продуктивный" } ],
  "totals": { "kcal": 520, "proteinG": 18, "fatG": 17, "carbsG": 70, "goal": 2200 }
}
```

`tasks[].flag`: `null` | `carried_over` | `overdue`. Время в ответе — UTC; клиент показывает в
часовом поясе пользователя.

## Ввод из Mini App (FR-004a, FR-013a)

### `POST /api/inputs`

- Голос: `multipart/form-data`, поля `kind=voice`, `file` (`audio/wav`, 16 кГц моно, ≤ 120 с).
- Фото: `multipart/form-data`, поля `kind=photo`, `file` (`image/jpeg` | `image/png` |
  `image/webp`, ≤ 10 МБ; клиент уменьшает до 1600 px по длинной стороне), `caption` (необязательно).
- Текст: JSON `{ "kind": "text", "text": "…" }` (1–4000 символов).

Ответ `202`:

```json
{ "inputId": "01J…", "status": "queued" }
```

### `GET /api/inputs/:id`

Клиент опрашивает раз в секунду до 30 секунд.

```json
{
  "inputId": "01J…", "status": "done", "transcript": "завтра в 10 созвон с Лёшей…",
  "created": { "events": ["01J…"], "tasks": ["01J…"], "meals": ["01J…"], "diary": ["01J…"] },
  "drafts": [ { "type": "event", "id": "01J…", "reason": "Уточните время" } ],
  "summary": "Встреча: Созвон с Лёшей — завтра 10:00\nЗадача: Купить молоко\n…",
  "canUndoUntil": "2026-10-01T03:12:00Z",
  "error": null
}
```

`status`: `queued` | `processing` | `done` | `empty` | `failed` | `deferred` | `undone`.

### `POST /api/inputs/:id/undo`

Удаляет все записи этого ввода (FR-011). `200` с новым состоянием; `409 undo_expired` после 24 ч.

### `POST /api/inputs/:id/retry`

Только для `failed`. `202`; `409 undo_expired`, если исходник истёк.

## Записи (FR-020–FR-023)

Для каждого типа — одинаковый набор. Тела валидируются по правилам из [data-model.md](../data-model.md).

| Метод и путь | Назначение |
|---|---|
| `POST /api/tasks` · `PATCH /api/tasks/:id` · `DELETE /api/tasks/:id` | задачи; `PATCH` с `{ "status": "done" }` отмечает выполненной |
| `POST /api/events` · `PATCH /api/events/:id` · `DELETE /api/events/:id` | встречи; поля даты и времени — локальные (`date`, `time` `HH:mm`), сервер переводит в UTC |
| `POST /api/meals` · `PATCH /api/meals/:id` · `DELETE /api/meals/:id` | приёмы пищи (тип, дата, время) |
| `POST /api/meals/:id/items` · `PATCH /api/meal-items/:id` · `DELETE /api/meal-items/:id` | позиции блюда; итог приёма пересчитывается |
| `POST /api/diary` · `PATCH /api/diary/:id` · `DELETE /api/diary/:id` | записи дневника |

Правка черновика с заполненными обязательными полями снимает `isDraft` (для встреч — создаёт
напоминание). Ответ на `POST`/`PATCH` — полная запись в формате ленты; на `DELETE` — `204`.

## Служебное

- `POST /bot/webhook` — вход Telegram, проверяется заголовок `X-Telegram-Bot-Api-Secret-Token`
  (см. [bot.md](./bot.md)). Не является частью API Mini App.
