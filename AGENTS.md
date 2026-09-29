<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# EZ Planner

## Что это
EZ Planner — планер дня внутри Telegram (бот @ezplaner_bot + Telegram Mini App): вертикальная
шкала дня с задачами по времени, полоса недели, Входящие, повторы и напоминания; дополнительно —
задачи голосом или текстом через ИИ и калории по фото блюда. Актуальная спецификация —
`specs/002-day-planner/` (001 заменена). Репозиторий вырос из шаблона AI Website Cloner
(Next.js + shadcn/ui + Tailwind v4).

**Главный документ — конституция `.specify/memory/constitution.md`.** Она имеет приоритет над
этим файлом. Перед любой работой прочитай её и спецификацию текущей функции в `specs/`.

## Процесс (spec-kit)
- Каждая функция: `/speckit-specify` → `/speckit-clarify` (по необходимости) → `/speckit-plan`
  → `/speckit-tasks` → `/speckit-implement` → `/speckit-converge`.
- Код пишется только по задачам из `specs/<функция>/tasks.md`. Ничего «заодно».
- Документация (спецификации, планы, задачи) — на русском языке.

## Tech Stack
- **Framework:** Next.js 16 (App Router, React 19, TypeScript strict)
- **UI:** shadcn/ui (Radix primitives, Tailwind CSS v4, `cn()` utility) — только как основа,
  внешний вид задаёт `DESIGN.md`
- **Icons:** Lucide React
- **Styling:** Tailwind CSS v4 with oklch design tokens
- **Платформа:** Telegram Mini App + бот; серверная часть и хостинг выбираются в плане первой
  функции

## Commands
- `npm run dev` — Start dev server
- `npm run build` — Production build
- `npm run lint` — ESLint check
- `npm run typecheck` — TypeScript check
- `npm run check` — Run lint + typecheck + build

## Code Style
- TypeScript strict mode, no `any`
- Named exports, PascalCase components, camelCase utils
- Tailwind utility classes, no inline styles
- 2-space indentation
- Responsive: mobile-first

## Design Principles
- **Один DESIGN.md** — весь интерфейс строится по `DESIGN.md` в корне репозитория (токены цвета,
  типографики, отступов, компонентов). Пока файл не выбран — только функциональная вёрстка.
- **Никаких «дефолтных» видов** — shadcn/ui без стилизации под DESIGN.md не выпускается.
- **Никаких чужих брендов** — логотипы, названия и закрытые шрифты из референсов не используются.
- **Telegram-first** — мобильный экран от 320 px, тема Telegram, safe area, кнопки «Назад» и
  главная кнопка Telegram.
- **Русский интерфейс** — 24-часовое время, метрическая система, ккал.

## Project Structure
```
src/
  app/              # Next.js routes
  components/       # React components
    ui/             # shadcn/ui primitives
    icons.tsx       # Extracted SVG icons as React components
  lib/
    utils.ts        # cn() utility (shadcn)
  types/            # TypeScript interfaces
  hooks/            # Custom React hooks
public/
  images/           # Downloaded images from target site
  videos/           # Downloaded videos from target site
  seo/              # Favicons, OG images, webmanifest
docs/
  research/         # Inspection output (design tokens, components, layout)
  design-references/ # Screenshots and visual references
scripts/            # Asset download scripts
specs/              # Спецификации, планы и задачи функций (spec-kit)
.specify/           # Конституция, шаблоны и скрипты spec-kit
.agents/
  skills/
    clone-website/  # Canonical cross-agent cloning workflow (наследие шаблона)
.claude/
  commands/
    clone-website.md # Thin Claude Code invocation bridge
  skills/
    speckit-*/      # Команды spec-kit для Claude Code
```

## Agent Workflow
- Команды spec-kit лежат в `.claude/skills/speckit-*`; не редактируй их вручную — они
  обновляются через `specify`.
- `/clone-website` — наследие шаблона. Для EZ Planner используется только для разбора
  референсов, не для копирования чужих сайтов в продукт.
- Edit `.agents/skills/clone-website/` for cloning-workflow changes. It is the canonical skill used by Codex, Cursor, and OpenCode.
- Keep `.claude/commands/clone-website.md` as a thin Claude Code bridge to the canonical skill; do not duplicate the workflow there.
