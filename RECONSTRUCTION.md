# Реконструкция кода репозитория Multica

Документ описывает результаты анализа (реконструкции) структуры, архитектуры и соглашений репозитория **Multica** — source-available платформы управления задачами, где AI-агенты (Claude Code, Codex и др.) работают как «цифровые сотрудники» на общей доске задач.

Дата анализа: 2026-10-21. Ветка: `qwen-code-ccecacad-…` (идентична `main`, diff = 0).

---

## 1. Общие сведения

| Параметр | Значение |
| --- | --- |
| Название / модуль Go | `github.com/multica-ai/multica/server` |
| Версия пакета | `0.2.0` (private monorepo) |
| Менеджер пакетов | `pnpm@10.28.2`, Node `>=22` (`catalog:`-зависимости в `pnpm-workspace.yaml`) |
| Сборка оркеструется | Turborepo (`turbo.json`) + `Makefile` |
| Языки | Go 1.26.9 (бэкенд), TypeScript/React (фронтенд), SQL (миграции) |
| Масштаб | ~6 700 файлов; ~4 650 исходников `.go/.ts/.tsx`; только непро тестовый Go-код ≈ 365 тыс. строк; 1 218 файлов миграций БД |
| Лицензия | Source-available (см. `LICENSE`, `NOTICE`) |

Назначение: канбан/issue-трекер, в котором задачи назначаются AI-кодинг-агентам. Агент «берёт» issue, работает на контролируемом runtime (через локальный демон), комментирует, поднимает блокировки и передаёт результат на ревью. Поддерживается self-hosting (Docker Compose, Helm).

## 2. Верхнеуровневая структура

```
/workspace
├── apps/                  # Клиентские приложения
│   ├── web/               # Next.js (веб-приложение, маршруты/лейауты)
│   ├── desktop/           # Electron (renderer + main, платформенные адаптеры)
│   ├── mobile/            # Expo / React Native (самостоятельный клиент: свой UI, стейт, i18n, релизы)
│   ├── docs/              # Fumadocs — сайт документации
│   └── ui-lab/            # Лаборатория/галерея UI-примитивов
├── packages/              # Общие пакеты монорепо
│   ├── core/              # Headless-логика: API-клиент, TanStack Query хуки, Zustand-сторы, i18n, типы
│   ├── ui/                # UI-примитивы (shadcn) и стили. БЕЗ бизнес-логики и импортов @multica/core
│   ├── views/             # Общие страницы/бизнес-компоненты для web+desktop (без next/* и react-router-dom)
│   ├── plugin-sdk/        # SDK для плагинов
│   ├── eslint-config/     # Общий ESLint
│   └── tsconfig/          # Базовые tsconfig
├── server/                # Go-бэкенд
│   ├── cmd/               # Исполняемые файлы: server (API), multica (CLI/демон), migrate, maintenance, backfill-*
│   ├── internal/          # ~45 пакетов: handler, daemon, realtime, auth, events, integrations, scheduler и т.д.
│   ├── pkg/               # Публичные пакеты: llm, composio, featureflag, plugincontract, publicapi/v1, agent, remotemcp и др.
│   ├── migrations/        # 1218 SQL-миграций (NNN_name.up/down.sql), без транзакций для индексов
│   └── sqlc.yaml          # Генерация типобезопасного Go-кода из SQL (`make sqlc`)
├── e2e/                   # Playwright-тесты (включая perf и iframe-наборы)
├── scripts/               # Dev/CI-скрипты (install.sh, dev-env.sh, test-go.sh, check-*.mjs …), каждый с .test.*
├── deploy/helm/multica/   # Helm-чарт (backend, frontend, postgres, ingress, configmap)
├── docker/ , docker-compose.selfhost.yml , Dockerfile , Dockerfile.web
├── examples/plugins/      # Примеры плагинов (plugincontract)
├── docs/ , .agents/skills # Инженерная документация и скиллы для coding-агентов
└── корневые конфиги       # turbo.json, knip.jsonc, playwright*.config.ts, .goreleaser.yml, .env.example (~40 КБ)
```

## 3. Архитектура

### 3.1 Бэкенд (Go)
- **HTTP-фреймворк**: Chi v5 (+ go-chi/cors), WebSocket — gorilla/websocket (`server/internal/realtime`, `daemonws`).
- **БД**: PostgreSQL через `jackc/pgx/v5` + **sqlc** (типобезопасные запросы; после изменения SQL — `make sqlc`). Redis (`go-redis`) для real-time/очередей/лок.
- **Аутентификация**: JWT (`golang-jwt/v5`), OAuth-провайдеры, S3 + SecretsManager (aws-sdk-go-v2) для хранилища.
- **Точки входа** (`server/cmd`):
  - `server` — основной API-сервер: `router.go`, слушатели событий (`listeners.go`, notification/inbox/subscriber/activity listeners), планировщик (`autopilot_schedule_job`, `runtime_sweeper`, `source_context_sweeper`, quota reconcilers), graceful shutdown.
  - `multica` — CLI и демон: управление агентами/issues/комментариями/скиллами/runtime-профилями, автозапуск демона (linux/darwin/windows), профиле-локи, discovery агентских CLI.
  - `migrate`, `maintenance`, `backfill_*` — одноразовые/операционные команды.
- **Ключевые домены `internal/`**: `handler` (REST), `agent`+`pkg/agent` (инвокация внешних CLI-агентов: Claude Code, Codex, Cursor/Kiro/Hermes/CodeBuddy/ACP и др.), `daemon`/`daemonws` (локальные рантаймы), `events`/`eventcontract`, `integrations` (Slack, Telegram, Lark/DingTalk, WeCom, GitHub, Composio), `issue*` (status, workflow, position, property, guard), `skill`, `scheduler`, `featureflags`, `entitlement`/`seatcapacity` (биллинг), `selfexec`, `metrics`, `profiling`.
- **Публичный API**: `pkg/publicapi/v1` — контракт с OpenAPI-спекой (`openapi.yaml`), RFC 7807 problem+json.
- **Плагины**: `pkg/plugincontract` (манифест, bundle, capabilities) + `packages/plugin-sdk` + `examples/plugins`.

### 3.2 Фронтенд (TypeScript/React)
Направление зависимостей строго: **`views → core + ui`**; `core` и `ui` независимы друг от друга.

- `packages/core` — единственный источник серверного состояния: zod-схемы API-ответов + `parseWithFallback` (никаких `as T`), Query-хуки (TanStack Query владеет серверными данными), Zustand-сторы (клиентское состояние: фильтры, черновики, модалки), `StorageAdapter` вместо `localStorage`, никаких `process.env` и `react-dom`.
- `packages/ui` — shadcn-примитивы; добавление через `pnpm ui:add`.
- `packages/views` — общие экраны; навигация только через адаптеры (`NavigationAdapter`, `useNavigation()`, `<AppLink>`); запрещены импорты `next/*`, определение сторов.
- `apps/web` — Next.js App Router; платформенные адаптеры в `apps/web/platform/`.
- `apps/desktop` — Electron; маршрутизация рабочего пространства через вкладки + `WindowOverlay` для one-shot флоу; `setCurrentWorkspace` из layout'ов.
- `apps/mobile` — полностью автономный Expo-клиент (свой UI/стейт/i18n/патчи, напр. `expo-modules-core@57.0.21.patch`); разделяет только core-типы и чистые утилиты.

### 3.3 Модель данных и БД
- Workspace-центрированная многопользовательская модель: users/workspaces/members, issues (канбан-статусы, свойства, метки, проекты, позиции), comments, agents + runtimes + sessions, skills, squads, autopilots, inbox, plugins, billing (entitlements/seats), integrations.
- **Правила миграций**: без FK/cascade (целостность в коде приложения, при необходимости — транзакция); все индексы — `CREATE INDEX CONCURRENTLY`, по одному индексу на файл-миграцию; идемпотентный DDL для условных объектов; учёт в `schema_migrations`.
- **UUID-правила в handlers**: три источника — resolve через лоадеры (`loadIssueForUser`…), `parseUUIDOrBadRequest` для пользовательского ввода, `parseUUID` (panic) только для доверенных round-trip sqlc/фикстур; вне handlers — `util.ParseUUID` с проверкой ошибки. Workspace-скоупинг по `workspace_id` + заголовок `X-Workspace-ID`; assignee полиморфный (`assignee_type` + `assignee_id`).

### 3.4 Real-time и события
WebSocket-события патчат/инвалидируют кечи TanStack Query (не складывают серверные payload в Zustand). Оптимистичные патчи — только при предсказуемом результате, редких отказах и тривиальном rollback; create/delete всегда ждут сервер.

## 4. Инструменты сборки и проверки

| Область | Команды |
| --- | --- |
| Окружение | `make up` / `make status` / `make down` (данные сохраняются) / `make destroy`; worktree-изоляция БД/портов через `.env.worktree` |
| Фронтенд | `pnpm dev:web|desktop|mobile|docs`, `pnpm build|typecheck|lint|test` (turbo, кроме mobile), `pnpm knip` (мёртвый код) |
| Бэкенд | `make test` (Go, c guard-скриптом `go-test-with-agent-cli-guard.sh`), `make sqlc` |
| Комплексно | `make check` |
| E2E | `pnpm exec playwright test` (+ `playwright.iframe.config.ts`, `playwright.perf.config.ts`) |
| Кастомные проверки | `scripts/check-ui-wildcard-exports.mjs`, `check-ui-radius-tokens.mjs`, `check-image-budget.mjs`, `ci-scope.mjs`; почти каждый скрипт имеет собственный `.test.*` |
| Релизы | GoReleaser (`.goreleaser.yml`), install.sh/install.ps1 (с тестами), Helm-чарт, Docker Compose selfhost |
| CI | GitHub Actions (`.github/`) |

## 5. Тестование
- Go: юнит+интеграционные тесты повсеместно (соотношение тестовых/нетестовых файлов ~1:1 по количеству; фикстуры `pkg/remotemcp/remotemcptest`, `internal/testutil`).
- TS: vitest-тесты рядом с исходниками; контракты API — «malformed-response» тесты обязательны при изменении эндпоинта.
- E2E: ~35 spec-файлов Playwright (issues, comments, agent routing, onboarding, settings, плагины/security).
- Скрипты инфраструктуры тоже тестируются (shell/ps1/mjs `.test.*`).

## 6. Конвенции и ограничения (из AGENTS.md)
1. Документация для агентов: корневой `AGENTS.md` + вложенные (`apps/mobile/AGENTS.md`); `CLAUDE.md` — только импорт.
2. Границы пакетов неизменяемы (см. таблицу в §3.2); общие фичи подключаются и в web-маршруты, и в desktop-роутер.
3. API-совместимость: установленные desktop-клиенты могут работать с более новыми бэкендами → все JSON проходят zod + fallback; неизвестные enum'ы не должны ломать UI.
4. UI-копирайтинг: описания по умолчанию отсутствуют; не дублировать заголовки/действия; важные последствия (стоимость, разрушительность) держать видимыми; мультиязычность (EN/ZH конвенции в `apps/docs/content/docs/developers/conventions*.mdx`).
5. Сторы: только auth/workspace сторы могут вызывать `api.*` напрямую; селекторы Zustand — стабильные ссылки (shallow-сравнение).
6. Сообщения — видимое pending-состояние + retry.

## 7. Деплой
- **Self-host**: `docker-compose.selfhost.yml` (+ build-вариант), `SELF_HOSTING*.md`, скрипт ожидания готовности `scripts/selfhost-wait.sh`.
- **Kubernetes**: `deploy/helm/multica` (Chart: backend, frontend, postgres, ingress, configmap; тест конфигурации `scripts/helm-config.test.sh`).
- **Облако/Vercel**: `Dockerfile.web`, `.vercelignore`; мобильные сборки — Expo/EAS-скрипты (`ios:mobile:*`).

## 8. Риски и технические долги (замеченные)
- Известный race при leaving workspace в `packages/views/settings/components/workspace-tab.tsx` (задокументирован как долг).
- Огромное число backfill-пакетов/команд (`attributionbackfill`, `chatoriginbackfill`, `delegatedrecoverybackfill`, `issueactivitybackfill`, `taskusagebackfill`) — признак эволюционной миграции данных.
- 1218 миграций — медленный «холодный» перенос схемы; новые индексы требуют отдельных файлов.
- Очень широкий список интеграций (Slack, Telegram, Lark, DingTalk, WeCom, GitHub, Composio…) — большая поверхность поддержки.

## 9. Как начать работу
```bash
make up                 # поднять managed-окружение (PostgreSQL и пр.)
pnpm install
pnpm dev:web            # веб-клиент
cd server && go run ./cmd/server   # API (или make-цели из Makefile)
make test && pnpm typecheck && pnpm lint   # минимальная проверка изменений
```
Перед правками прочитать `AGENTS.md` (и `apps/mobile/AGENTS.md` для мобильных изменений).
