# Implementation Plan: EZ Planner — голосовой планировщик жизни в Telegram (MVP)

**Branch**: `001-voice-life-planner` | **Date**: 2026-09-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-voice-life-planner/spec.md`

## Summary

MVP личного планировщика в Telegram. Голосовое, текст или фото — в чате с ботом или прямо в Mini
App — превращаются в задачи, встречи, приёмы пищи с калориями и записи дневника. Бот присылает
итог с отменой. Mini App показывает ленту дня с правкой, а бот напоминает о встречах.

Техническое решение ([research.md](./research.md)): один Cloudflare Worker на бесплатном тарифе.
Он делает всё:
- отдаёт статическую сборку Mini App (Next.js 16, `output: 'export'`);
- обслуживает API Mini App (Hono) и вебхук бота (grammY);
- обрабатывает очередь входящих сообщений (Cloudflare Queues);
- шлёт напоминания по cron раз в минуту.

Данные хранятся в D1 (SQLite, Drizzle ORM), временные медиа — в KV с TTL 24 часа. ИИ работает на
Workers AI:
- Whisper large v3 turbo — речь;
- Llama 3.3 70B в JSON Mode — разбор текста;
- Llama 4 Scout — еда на фото.

Все ответы моделей проходят валидацию `zod` и детерминированные правила.

## Technical Context

**Language/Version**: TypeScript 5 (strict, без `any`); Node.js 24 для сборки и инструментов;
среды выполнения — браузер Telegram WebView (Mini App) и Cloudflare Workers (workerd).

**Primary Dependencies**:
- Mini App: Next.js 16, React 19, Tailwind CSS v4, shadcn/ui, `@tma.js/sdk-react`.
- Worker: Hono, grammY, Drizzle ORM, zod, `date-fns` + `@date-fns/tz`.
- Инструменты: Wrangler, drizzle-kit.

**Storage**: Cloudflare D1 (записи, настройки, учёт лимитов); Workers KV (медиа из Mini App,
TTL 24 ч); Cloudflare Queues (задачи обработки).

**Testing**: Vitest (unit: подпись `initData`, правила дат, валидация ответов ИИ, суммы КБЖУ,
доступ). Оценочные наборы `tests/eval/` для SC-003 и SC-004. CI GitHub Actions: lint, typecheck,
test, build.

**Target Platform**: Telegram-клиенты iOS и Android (Mini App в WebView, экран от 320 px) и
Cloudflare Workers.

**Project Type**: веб-приложение — Mini App и serverless-бэкенд в одном репозитории.

**Performance Goals**:
- итог по голосовому до 30 с — не позже 15 с в 90 % случаев (SC-002);
- API ленты дня — p95 < 500 мс;
- первый экран Mini App < 2 с на 4G (без сетевых ограничений).

**Constraints**:
- Workers Free: 10 мс CPU на вызов; 100 000 запросов в сутки; 50 запросов к D1 и 50 подзапросов
  на вызов.
- Workers AI: 10 000 нейронов в сутки, это ~60 голосовых на весь сервис.
- Queues: 10 000 операций в сутки. KV: 1 000 записей в сутки.
- Размещение за рубежом: у части пользователей из России Mini App может не открываться (решение
  владельца, риск записан).
- Голос до 120 с; секреты только в Worker secrets.

**Scale/Scope**: закрытый тест, до 20 тестировщиков, до ~60 обработок ИИ в сутки. Mini App —
2 экрана (лента дня, настройки) плюс листы ввода и редактирования.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Конституция v1.1.0. Проверка до исследования и повторно после проектирования (Phase 1).

| Принцип | Как план его выполняет | Статус |
|---|---|---|
| I. Telegram-first | Бот и Mini App. Вход по `initData` без регистрации. Тема, safe area, «Назад» и главная кнопка через `@tma.js/sdk-react`. Вёрстка от 320 px. | PASS |
| II. Голос и фото — главный ввод | Голос, текст и фото — в чате и в Mini App. Итог после каждого ввода, отмена 24 ч, черновики при неуверенности, ручные формы как запасной путь. | PASS |
| III. Собственный дизайн по DESIGN.md | Задачи стилизации UI заблокированы до выбора DESIGN.md. До этого — функциональная вёрстка на токенах-заглушках. Чужих логотипов нет. | PASS (с условием) |
| IV. Приватность | Секреты — Worker secrets. Подпись `initData` проверяется на каждом запросе. Фильтр по `user_id` везде. Медиа удаляются после обработки (KV TTL 24 ч, из чата хранится только `file_id`). Есть «Удалить всё» и сводка данных. | PASS |
| V. Простота | Один Worker, один деплой, один репозиторий. Очередь и KV обоснованы ниже. OpenNext, отдельный бэкенд и внешняя БД отклонены. | PASS |
| VI. Качество | TypeScript strict. CI: lint, typecheck, test, build. Сценарии приёмки → [quickstart.md](./quickstart.md). Оценочные наборы для SC-003 и SC-004. | PASS |
| Технологические рамки | Next.js 16 из шаблона сохранён. Доступность из России оценена, риски записаны ([research.md](./research.md)). Продукт не рекламирует обход ограничений. | PASS |

**Повторная проверка после Phase 1**: модель данных, контракты и quickstart не вводят новых
сервисов и не нарушают принципы. Сохраняется условие по принципу III: задачи визуальной стилизации
идут после выбора DESIGN.md.

## Project Structure

### Documentation (this feature)

```text
specs/001-voice-life-planner/
├── plan.md              # этот файл
├── research.md          # Phase 0: решения и альтернативы
├── data-model.md        # Phase 1: таблицы D1, правила, переходы статусов
├── quickstart.md        # Phase 1: развёртывание и сценарии проверки
├── contracts/
│   ├── api.md           # HTTP API Mini App
│   ├── bot.md           # команды, сообщения и кнопки бота
│   └── ai-output.md     # схемы ответов моделей и постобработка
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
src/                          # Mini App (Next.js 16, статический экспорт)
├── app/
│   ├── layout.tsx            # провайдер Telegram SDK, тема, шрифты
│   ├── page.tsx              # лента дня (?date=YYYY-MM-DD, ?input=<id>)
│   ├── settings/page.tsx     # настройки, мои данные, удаление
│   └── globals.css           # токены из DESIGN.md (до выбора — заглушки)
├── components/
│   ├── ui/                   # shadcn/ui, стилизованные под DESIGN.md
│   ├── day/                  # блоки ленты: встречи, задачи, питание, дневник, навигация по дням
│   ├── input/                # запись голоса, фото, текст, лист результата ввода
│   └── records/              # формы создания и правки записей
├── lib/
│   ├── api-client.ts         # fetch + заголовок initData + zod-проверка ответов
│   ├── telegram.ts           # обёртка над @tma.js/sdk-react
│   ├── wav-recorder.ts       # запись через Web Audio → WAV 16 кГц моно
│   ├── image-resize.ts       # уменьшение фото до 1600 px
│   └── utils.ts              # cn() из шаблона
└── shared/                   # общий код Mini App и Worker
    ├── schemas.ts            # zod-схемы API (contracts/api.md)
    └── types.ts

worker/                       # Cloudflare Worker
├── index.ts                  # fetch (Hono + вебхук), queue, scheduled
├── env.ts                    # типы биндингов: DB, MEDIA, INPUTS, AI, ASSETS, секреты
├── api/                      # маршруты /api/*: me, day, inputs, tasks, events, meals, diary
├── bot/                      # grammY: команды, приём сообщений, итоги, callback-кнопки
├── auth/                     # проверка initData, список доступа
├── pipeline/                 # consumer очереди: stt → parse/food → rules → save → notify
│   ├── schemas.ts            # zod-схемы ответов ИИ (contracts/ai-output.md)
│   └── rules.ts              # детерминированные правила дат, черновиков, сумм
├── ai/                       # адаптеры Workers AI + учёт бюджета нейронов
├── reminders/                # пересчёт и отправка напоминаний
└── db/
    ├── schema.ts             # Drizzle-схема (data-model.md)
    └── repo/                 # запросы по сущностям, всегда с user_id

migrations/                   # SQL-миграции D1 (drizzle-kit generate)
scripts/
├── bot-setup.ts              # setWebhook, setMyCommands, setChatMenuButton
└── eval.ts                   # прогон оценочных наборов
tests/
├── unit/                     # Vitest
└── eval/                     # phrases.jsonl, photos/, expected.json
wrangler.jsonc                # Worker, assets (out/), D1, KV, Queue, AI, cron
```

**Structure Decision**: один репозиторий и один деплой. Корень остаётся Next.js-приложением из
шаблона (Mini App), серверный код живёт в `worker/`. Общие схемы лежат в `src/shared/` и
импортируются обеими сторонами через алиас `@shared/*`. Сборка: `next build` → `out/`, затем
`wrangler deploy` публикует Worker с `out/` как статическими ассетами. Запросы `/api/*` и
`/bot/*` сначала идут в Worker (`run_worker_first`), остальное отдаётся как SPA.

Наследие шаблона (`/clone-website`, `Dockerfile`, `docker-compose.yml`) не используется продуктом
и не мешает. Удаление — отдельной задачей по желанию владельца.

## Порядок реализации (для /speckit-tasks)

1. **Основа**:
   - Wrangler и биндинги;
   - Drizzle-схема и миграции;
   - проверка `initData` и список доступа;
   - каркас Hono и grammY;
   - CI с тестами;
   - spike: OGG из Telegram → Whisper (решение по R4).
2. **US1 (P1)**: приём ввода (чат и Mini App) → очередь → STT → разбор → правила → сохранение →
   итог, отмена и повтор.
3. **US2 (P2)**: лента дня, CRUD записей, перенос задач, навигация по дням.
4. **US3 (P3)**: фото → Llama 4 Scout → приём пищи; оценочный набор фото.
5. **US4 (P4)**: напоминания (cron, пересчёт при правке).
6. **US5 (P5)**: настройки, сводка данных, удаление всего.
7. **Дизайн**: выбор DESIGN.md → токены в `globals.css` → стилизация компонентов (принцип III).
8. **Полировка**: лимиты и бюджет, `/today`, оценка SC-003 и SC-004, прогон quickstart.

## Complexity Tracking

| Отступление от минимума | Зачем нужно | Почему проще не подходит |
|---|---|---|
| Очередь (Cloudflare Queues) помимо Worker | Вебхук обязан быстро ответить Telegram; обработка ИИ идёт до ~15 с; нужны повторы при сбоях («сообщение не теряется молча») | `ctx.waitUntil` не повторяет задачу при сбое и ограничен 30 с |
| KV для временных медиа | Сообщение очереди ≤ 128 КБ, а голос из Mini App до ~3,8 МБ. TTL 24 ч автоматически выполняет FR-032 | Хранить в D1 — лимит 2 МБ на строку и ручная очистка; R2 — больше настройки |
| Размещение за рубежом (риск, не нарушение) | Решение владельца: простота и почти нулевая стоимость для закрытого теста | Вариант «Россия + релей + ИИ Яндекса» — два сервера и платно; остаётся планом перед публичным запуском |
