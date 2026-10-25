# Удалённое соединение/управление в Multica: как работает и насколько сложно вырезать

Дата анализа: 2026-10-21. Ветка идентична `main` (HEAD `1e01173`). Рабочее дерево не изменялось — только этот отчёт.

---

## 1. Что вообще является «удалённым управлением»

В Multica три независимых удалённых канала, которые часто путают:

| Канал | Клиент → Сервер | Назначение | Можно ли без него? |
| --- | --- | --- | --- |
| **A. Daemon control channel** | `multica daemon` (локальный демон на машине пользователя) ↔ API-сервер | Регистрация runtime'ов, получение задач (claim), heartbeat, отчёты о задачах | Это ядро продукта «агент работает на вашей машине» |
| **B. Realtime WS (`/ws`)** | Браузер/desktop/mobile ↔ сервер | Push событий UI (issue обновлён, комментарий…) | Нет — фронт деградирует до polling |
| **C. Внешние интеграции** | Slack/Telegram/Lark/DingTalk/WeCom/GitHub/Composio | Мессенджер-управление агентами | Да, опционально |

Дальше — про **A**, как основной «удалённый контроль над локальной машиной», и отдельно про B/C.

### 1.1 Топология канала A (важно)

Соединение **всегда исходящее от демона к серверу**. Сервер никогда не подключается к машине пользователя: нет входящих портов, NAT traversal, reverse-tunnel. Протокол — «pull-based job queue»: сервер лишь *сигнализирует* через уже открытый WS-канал демона, что есть работа; сам забор задачи и доставка результатов идут через HTTP-API демона. Агентский CLI запускается демоном локально, в изолированном workdir.

### 1.2 Стек демона (клиентская сторона)

Пакет `server/internal/daemon` (~150 файлов, главный `daemon.go` — 10 649 строк):

- **Точка входа**: `cmd/multica/cmd_daemon.go` → `daemon.New(cfg).Run(ctx)`; `run()` (daemon.go:2062) поднимает:
  - loopback health-порт (детект чужого демона, статус для desktop/CLI);
  - `resolveAuth()` — PAT из конфига профиля (`multica login`);
  - `preflightAuth()` — продление PAT + первая синхронизация workspace'ов;
  - фоновые циклы: `workspaceSyncLoop`, `taskWakeupLoop` (WS), `heartbeatLoop`, `pollLoop` (HTTP claim), `gcLoop`, `autoUpdateLoop`, `tokenRenewalLoop`, `agentDiscoveryLoop`, `terminalReportReplayLoop`.
- **`client.go` (1 421 строк)** — HTTP-клиент ~40 эндпоинтов `/api/daemon/*`: register/deregister/heartbeat, `ClaimTask(s)` (+legacy fallback), start/progress/complete/fail/usage/messages, renew token, skill bundles, gc-check, update/model-list/local-skills результаты.
- **`wakeup.go`** — WS-соединение `GET /api/daemon/ws?runtime_ids=…` (http→ws, https→wss), backoff с jitter, pong/write timeouts, лимит чтения 64 MiB (batch-claim ответы велики), поддержка `HTTPS_PROXY`, writer-канал, heartbeat-отправитель, детач при реконнекте.
- **`wsrpc.go`** — generic WS-RPC (MUL-4257): `tasks.claim` гоняется по тому же сокету; при недоступности WS — прозрачный fallback на HTTP.
- **Безопасность исполнения**: `validateTaskIdentity` (несовпадение agent-id в payload ⇒ отказ), пустой `workspace_id` ⇒ отказ, allowlist репозиториев (`repoAllowlist`/`ensureRepoReady`), task-scoped токен (`taskScopedAuthToken`), маркеры execenv для fail-closed CLI, watchdog'и/таймауты, GC артефактов.

### 1.3 Серверная сторона канала A

- **Middleware `DaemonAuth`** (`internal/middleware/daemon_auth.go`, 278 строк): 4 пути аутентификации — `mdt_` daemon-token (hash в БД + Redis-кэш), `mul_` PAT, `mcn_` Cloud-PAT (fail-closed), JWT. Серверные заголовки `X-Actor-Source/X-Agent-ID/X-Task-ID` всегда стираются.
- **Роуты** (`cmd/server/router.go:1560-1617`): группа `/api/daemon/*` — ~40 маршрутов (см. выше) + `GET /api/daemon/ws`.
- **Хендлеры** (`internal/handler/daemon*.go`; `daemon.go` — 6 165 строк): ownership-гейты `requireDaemonWorkspaceAccess/RuntimeAccess/TaskAccess`; claim-pipeline (`ClaimTasksByRuntime`, `buildClaimedTaskResponse` — сборка полного Task payload: инструкции агента, скиллы, контекст issue/чата/autopilot, репозитории, MCP-подключения + выдача короткоживущих токенов `remoteMCPDaemonTokenForClaim`/task-scoped token); heartbeat c pending-actions (update, model list, local skills import); завершение/отчёты задач.
- **`internal/daemonws` (hub.go 1 164 + notifier/rpc_completion/metrics)**: Hub живых WS-соединений демона, скоуп `daemon:<runtimeID>`; `NotifyTaskAvailable/Supplement/PendingWork/RuntimeGone/WorkspacesChanged/RuntimeProfilesChanged`; дедупликация по event-id; leases рантаймов; `RelayNotifier` — fan-out через Redis-relay realtime для мультинодового деплоя; `SetRPCHandler(DaemonRPCHandler)` — WS-claim переиспользует HTTP-hendler in-process (`rpcResponseCapture`).
- **Прочие потребители**: sweeper'ы (`cmd/server/runtime_sweeper.go` — офлайн-перехват, orphan-recovery), `service/task.go` (пуши при постановке задачи в очередь), `handler/runtime*.go`, метрики `internal/metrics/daemonws.go`.

### 1.4 Жизненный цикл задачи (секвенция)

```
UI/API создаёт task → service/task.go кладёт в agent_task_queue
→ hub.NotifyTaskAvailable(runtimeID) [или Redis relay на другой ноде]
→ wakeup-кадр уходит демону по WS
→ демон: batch claim (WS RPC tasks.claim, fallback POST /api/daemon/tasks/claim)
   сервер: гейты ownership → buildClaimedTaskResponse → finalize (атомарно: task-scoped + mdt_ токены)
→ демон: POST /tasks/{id}/start → готовит execenv (workdir, скиллы, repo allowlist)
→ spawn агентского CLI (claude/codex/qwen/…), стрим сообщений
→ POST /tasks/{id}/messages|progress|usage → complete|fail (сессия, ветка, артефакты)
→ heartbeat каждые 15s (WS-first, HTTP fallback); сервер через heartbeat шлёт pending-actions
```

Деградация при обрыве WS: polling-fallback (30s catch-up, 3m ws-claim интервал при здоровом WS), replay терминальных отчётов, reconcile-broadcaster после реконнекта.

### 1.5 Управление демоном со стороны UI

Desktop/Web показывают runtime'ы (online/offline, версии агентов) через обычные user-auth роуты `/api/workspaces/.../runtimes`, умеют: удалить runtime, послать remote-update (`PendingUpdate` через heartbeat-action), список моделей, импорт локальных скиллов — всё это *команды через очереди сервера*, исполняемые демоном добровольно. Autostart демона — per-user (launchd/systemd user unit/Run key), не системный сервис. Self-update скачивает бинарник из GitHub Releases (или self-hosted зеркала) с проверкой checksums.txt.

### 1.6 Отличия каналов B и C

- **B (`/ws`, `internal/realtime`)**: браузерские клиенты, membership-check по workspace, патчит TanStack Query кеши. Полностью отделён от daemonws (общий только Redis-relay слой).
- **C**: `internal/integrations/*` — собственные long-lived WS к мессенджерам (WeCom и др.) + webhook-роуты; управляются env-флагами, при отсутствии конфигурации молча выключены.

---

## 2. Насколько сложно «вырезать»

Формулировка «вырезать» трактуется двояко — разобрал оба сценария.

### Сценарий 1: сделать продукт чисто серверным (убрать внешний контроль над локальными машинами)

Оценка сложности: **высокая, но архитектура это допускает** — код специально разделён по границам пакетов.

Минимально жизнеспособный «вырез» (≈ 9–10 тыс. строк кода + тесты ≈ ещё ~15 тыс.):

1. **Клиент целиком**: пакет `server/internal/daemon` (~150 файлов) удаляется; `cmd/multica` теряет `cmd_daemon*.go` (~20 файлов), autostart-платформы, `agents_probe*`, часть `cmd_agent` (local skills/model discovery).
2. **Сервер**:
   - `internal/daemonws` (≈2 тыс. строк с тестами) — удалить;
   - middleware `daemon_auth.go` — удалить (если остаётся PAT-путь для CLI-пользователей — оставить упрощённо);
   - `handler/daemon*.go` (~8 тыс. строк, из них daemon.go 6 165) — удалить; в `handler.go` убрать поля `DaemonHub`, 5 notifier-интерфейсов и ~10 вызовов `notifyDaemon*`;
   - `router.go`: удалить блок `/api/daemon` (~60 строк) и wiring `daemonHub`/`RelayNotifier`/`SetHeartbeatHandler/SetRPCHandler` (~40 строк);
   - точечные правки: `service/task.go`, `handler/runtime*.go`, `handler/workspace_revoke.go`, `handler/agent.go` (генерация `mdt_`), `cmd/server/runtime_sweeper.go`, `health_realtime.go`, `metrics/daemonws.go`.
3. **Frontend**: экраны Runtimes/Local-directory/agent-presence (`packages/core/runtimes`, `agents/runtime-binding.ts`, views-экраны) — скрыть или удалить.
4. **Миграции трогать нельзя** (история БД): таблицы `agent_runtimes`, `daemon_tokens`, `agent_task_queue` остаются — просто перестают заполняться. Плюс правила проекта: без FK, значит cascade-чистка в коде — её тоже надо сохранить для совместимости старых данных.

Что ломается по существу: сама модель «агент выполняется у вас» исчезает; чтобы продукт жил дальше, нужен серверный execution backend (замена `pkg/agent`-spawn на контейнерный раннер) — это уже новая функциональность, а не удаление.

Точки высокой связанности (главные риски):
- `handler/daemon.go` — монолит на 6 165 строк, переплетён с `service/task.go` (claim/finalize/requeue-логика);
- WS↔HTTP двойные пути claim (`batchClaimUnsupported`, legacy fallback, requeueUndeliveredBatchClaims) — при вырезании легко оставить «висячие» dispatched-задачи;
- heartbeat несёт pending-actions (updates/models/skills) — их потребителей тоже надо убирать;
- desktop-приложение управляет «bundled daemon» — его main-process (`apps/desktop/src/main`) завязан на health-порт и жизненный цикл демона.

Ориентировочная трудоёмкость для инженера, знакомого с кодовой базой: 1–2 недели аккуратного вырезания + регресс (значительная доля Go-тестов — именно daemon-тесты, их удаление тривиально, но e2e-спеки и фикстуры тоже завязаны).

### Сценарий 2: «вырезать» защиту/контроль, получив злоумышленнику доступ к чужим машинам

Здесь вывод обратный: **прямых бэкдоров нет, цепочка аутентификации многослойная**. Для компрометации нужен либо валидный токен жертвы (`mdt_`/PAT/JWT), либо RCE на самом сервере/демоне, либо митм с подменой сервера. Ограничения дизайна работают против злоумышленника: исходящие соединения, ownership-гейты на каждом daemon-эндпоинте, server-set заголовки, allowlist репозиториев, task-scoped токены, fail-closed cloud-PAT. Из ранее найденных уязвимостей (см. аудит) наиболее релевантен путь повышения через слабый `JWT_SECRET`/cloud-Verifier конфигурацию — т.е. риск в операционке self-host'а, а не в протоколе.

### Сценарий 3: отключить только WS-канал (оставив HTTP)

**Тривиально и безопасно**: WS в канале A — чисто latency-оптимизация. Если сервер не отдаёт upgrade (или демон стартует с недоступным `/api/daemon/ws`), логика `errRuntimeSetChanged`/`batchClaimUnsupported` штатно деградирует к HTTP polling (30s). Достаточно не монтировать `h.DaemonWebSocket` (или закрыть маршрут) — демон продолжит работать; пострадает только время pickup'а задач. Аналогично канал B опционален (polling-режим фронта).

---

## 3. Итог

- Удалённое управление = pull-based канал демон↔сервер (HTTP + вспомогательный WS), без входящего доступа на машину; команды «сверху» выполняются демоном добровольно после проверки ownership и allowlist.
- Вырезать полностью (сценарий 1) — дорого из-за объёма daemon-домена (~10 тыс. строк prod-кода + огромный пласт тестов) и связности claim/heartbeat с `service/task.go`, но границы пакетов чистые: `daemon`, `daemonws`, `handler/daemon*`, `middleware/daemon_auth`, один блок роутера. Реалистичный план — feature-flag/deployment-вариант «server-only runtimes», а не физическое удаление.
- Частичное отключение WS (сценарий 3) — копеечное, предусмотренное самой архитектурой.
- Никаких секретных бэкдоров в канале не обнаружено; безопасность держится на токенах и гейтах, поэтому практический риск смещён в корректность self-host конфигурации (секреты, TLS).
