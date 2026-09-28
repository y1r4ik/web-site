# Contract: ответы ИИ

Внутренний контракт между моделями Workers AI и конвейером обработки. Схемы задаются `zod` в
`worker/pipeline/schemas.ts`. Из них же генерируется JSON Schema для `response_format`.

Ответ, не прошедший валидацию:
1. Повторяется один раз.
2. При второй неудаче ввод получает статус `failed` (`parse_failed` или `vision_failed`).

Никакие данные из ответа модели не сохраняются без валидации.

## 1. Разбор текста — `ParseResult`

**Вход модели** (системный промпт + сообщение):
- распознанный текст;
- текущие локальные дата, время и день недели пользователя;
- часовой пояс IANA;
- правила: абсолютные даты в формате `YYYY-MM-DD`, время `HH:mm`, не выдумывать то, чего не
  сказано, при сомнении — `uncertain: true` и причина.

```ts
type ParseResult = {
  records: Array<
    | { type: "task"; title: string; dueDate: string | null;
        uncertain: boolean; reason: string | null }
    | { type: "event"; title: string; date: string | null; time: string | null;
        durationMin: number | null; location: string | null;
        uncertain: boolean; reason: string | null }
    | { type: "meal"; mealType: "breakfast" | "lunch" | "dinner" | "snack" | null;
        date: string | null; time: string | null;
        items: Array<{ name: string; grams: number; kcal: number;
                       proteinG: number; fatG: number; carbsG: number;
                       uncertain: boolean }>;
        uncertain: boolean; reason: string | null }
    | { type: "diary"; text: string; date: string | null }
  >;
};
```

**Постобработка** (детерминированные правила, покрыты unit-тестами):

| Правило | Результат |
|---|---|
| `event.date = null` или `event.time = null` | черновик «Уточните дату/время» (FR-008) |
| событие сегодня, время уже прошло | черновик «Время уже прошло — может быть, завтра?» |
| `meal.date = null` | дата сообщения |
| `meal.mealType = null` | по местному времени (см. data-model) |
| `diary.date = null` | дата сообщения |
| пустой `records` | ввод `empty` (FR-012) |
| итог приёма пищи | сумма позиций, округление до целых ккал и 1 знака для БЖУ |
| диапазоны полей | вне диапазонов data-model → запись становится черновиком, значение обрезается |

## 2. Еда на фото — `FoodResult`

**Вход модели**:
- изображение (JPEG, ≤ 1600 px);
- подпись пользователя, если есть;
- правила: русские названия блюд, граммы на порцию на фото, типичные рецептуры российской
  кухни, `isFood: false`, если еды нет.

```ts
type FoodResult = {
  isFood: boolean;
  mealTypeHint: "breakfast" | "lunch" | "dinner" | "snack" | null; // из подписи
  items: Array<{ name: string; grams: number; kcal: number;
                 proteinG: number; fatG: number; carbsG: number;
                 confidence: number /* 0..1 */ }>;
};
```

**Постобработка**:
- `isFood = false` или пустой `items` → ввод `empty` с текстом «Не нашёл еды на фото» (FR-015);
- `confidence < 0,6` → `low_confidence = 1`;
- приём пищи всегда `is_estimate = 1`.

## 3. Распознавание речи

`@cf/openai/whisper-large-v3-turbo`, параметры `language: "ru"`, `vad_filter: true`. Результат —
строка `text`. Пустая строка или только шум → ввод `empty`.

## Оценочные наборы

- `tests/eval/phrases.jsonl` — 50 фраз с ожидаемыми записями (SC-003).
- `tests/eval/photos/` + `expected.json` — 30 фото с эталонными калориями (SC-004).

Скрипт `npm run eval` прогоняет наборы через реальные модели и печатает метрики критериев успеха.
