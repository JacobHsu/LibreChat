---
title: Scheduled Chats
icon: CalendarClock
description: Turn on Scheduled Chats, set limits and project rules, allow unattended MCP access, and meet the Redis and checkpointer requirements.
---

> [!NOTE]
> **官方原文副本** — 線上版：<https://www.librechat.ai/docs/configuration/scheduled_chats>（官方沒有中文版）
> 原始檔：[LibreChat-AI/librechat.ai `content/docs/configuration/scheduled_chats.mdx`](https://github.com/LibreChat-AI/librechat.ai/blob/408527900701f755d2817a02d0c9c6befe0f57df/content/docs/configuration/scheduled_chats.mdx)，commit `4085279`，2026-11-25 下載。
> 標示 **📝 本機附註** 的區塊為本機補充，其餘為原文。本機處理：[issue #5](https://github.com/JacobHsu/LibreChat/issues/5)。

[Scheduled Chats](/docs/features/scheduled_chats) are experimental and off until you opt in. Scheduled runs use model, tool, and Code Interpreter resources without anyone watching, so check the deployment requirements below before enabling them.

## Enable

Add `interface.schedules` to `librechat.yaml` and restart LibreChat:

```yaml filename="librechat.yaml"
interface:
  schedules:
    use: true
    create: true
    maxPerUser: 10
    minIntervalMinutes: 60
    autoDisableAfterFailures: 5
    fireConcurrency: 5
    requireProject: false
    # projectId: '000000000000000000000000'
```

`use` and `create` seed the `SCHEDULES` permissions for the built-in `USER` role at startup; after that, manage them per role in the [Admin Panel](/docs/features/admin_panel). `use` lets a user see their schedules. `create` also lets them create, edit, enable, run, and delete them. The other keys are runtime limits:

<OptionTable
  options={[
    ['maxPerUser', 'Number', 'Maximum schedules each user can own, counting disabled ones. Set to 0 to prevent creation.', '10'],
    ['minIntervalMinutes', 'Number', 'Shortest allowed gap between two runs of one schedule. Applies to every cadence, including custom cron.', '60'],
    ['autoDisableAfterFailures', 'Number', 'Consecutive failed runs before a schedule disables itself.', '5'],
    ['fireConcurrency', 'Number', 'Maximum scheduled runs admitted at the same time across the deployment.', '5'],
    ['requireProject', 'Boolean', 'Every schedule must resolve to a Chat Project, checked when it is saved and before each run.', 'false'],
    ['projectId', 'String', 'Pins every scheduled conversation to one Chat Project and implies requireProject.', ''],
  ]}
/>

`schedules: true` enables the feature with these defaults. `schedules: false`, or `use: false`, is a deployment-wide stop that role, group, and user overrides cannot lift.

Project rules are rechecked at every run. A schedule that no longer satisfies `requireProject`, or whose project was deleted, disables itself instead of starting an unscoped conversation. Chat Projects belong to one user, so a deployment-wide `projectId` only works for that project's owner; set it in role or user overrides when owners need different destinations.

The full field list, including `admissionConcurrency`, `mcpPreflightConcurrency`, and `mcpPreflightTimeoutMs` for bounding admission and MCP readiness checks separately from generation, is in [`interface.schedules`](/docs/configuration/librechat_yaml/object_structure/interface#schedules).

## Unattended MCP access

Before a run starts, LibreChat checks every MCP tool the scheduled agent and its subagents need. A run that would need an interactive login does not start; the schedule shows which agent needs attention.

To let users authorize MCP tools for unattended runs, turn on `interface.schedules.mcpConsent`:

- `mcpConsent.enabled` turns the feature on.
- `mcpConsent.maxLifetimeHours` caps how long an authorization lasts, from 1 to 8760 hours.
- `mcpConsent.resources` lists the MCP resources that may be authorized.
- `mcpConsent.readOnlyPolicy` lists trusted read-only tool declarations, keyed by server and upstream tool name.
- `mcpReceiptRetry.baseMs` and `mcpReceiptRetry.maxMs` set the backoff for retrying consent receipts.

Users then see a **Scheduled MCP authorization** section in the schedule editor. An authorization covers only the listed agent and resources until its deadline and never permits writes.

## Deployment requirements

Scheduled Chats coordinate through MongoDB leases, so several replicas can share the work. They also need:

- **Redis streams in multi-replica deployments.** Set `USE_REDIS_STREAMS=true`. Without shared streams, saving a schedule fails rather than starting work another replica could not resume or stop.
- **A durable checkpointer for agents that can pause.** If the scheduled agent can use Ask User or needs tool approval, it needs Redis streams and a shared checkpointer even on a single process. The default MongoDB [`checkpointer`](/docs/configuration/librechat_yaml/object_structure/agents#checkpointer) works; `type: memory` does not. LibreChat rejects such a run before any model call when either is missing.
- **The standard server entrypoint.** The legacy experimental clustered entrypoint does not start the schedule engine.

Two environment variables control the engine (also listed in [dotenv](/docs/configuration/dotenv#scheduled-chats)):

<OptionTable
  options={[
    ['SCHEDULES_SINGLE_PROCESS', 'Boolean', 'Allows Scheduled Chats without Redis. Set it only when the deployment truly runs one LibreChat process.', '# SCHEDULES_SINGLE_PROCESS=true'],
    ['SCHEDULES_DISABLED', 'Boolean', 'Emergency stop for both automatic runs and Run now. Schedules are kept, not deleted.', '# SCHEDULES_DISABLED=true'],
  ]}
/>

> [!NOTE]
> **📝 本機附註：預設安裝每次啟動都有兩行 error，以及本機怎麼處理**
>
> **現象**：預設的 Docker 安裝（單一 api 容器、沒有 Redis）每次啟動都會記錄：
> ```text
> error: [schedules] scheduler NOT started: this process cannot see other replicas' generations. ...
>        Enable a shared stream store (USE_REDIS_STREAMS), or set SCHEDULES_SINGLE_PROCESS=true ...
> error: [schedules] write routes are PERMANENTLY unavailable in this process: the engine did not arm.
> ```
>
> **為什麼功能沒開也會出現**：官方說排程功能「預設關閉」，但**排程引擎每次啟動都會嘗試啟動**，刻意不看 `librechat.yaml`（`packages/api/src/schedules/service.ts` 的 `initializeScheduleEngine` 註解）。理由有兩個：
> - 功能可以只對某個角色、使用者或租戶開啟，即使基礎 YAML 是預設關閉。
> - 引擎還負責清理上一個 process 留下的排程紀錄。
>
> 能不能啟動只看一個條件：
> ```ts
> // packages/api/src/schedules/service.ts — isTopologySafeToArm()
> return GenerationJobManager.isRedis || isEnabled(process.env.SCHEDULES_SINGLE_PROCESS);
> ```
> 程式自己無法知道有幾個 replica，所以單一 process 要由管理者明確宣告（註解原文：「a process cannot observe its own replica count, so that case needs an explicit operator assertion」）。
>
> **本機的判斷**：符合上表 `SCHEDULES_SINGLE_PROCESS` 的條件「truly runs one LibreChat process」。
> - 只有 1 個 api 容器。
> - 執行的是標準入口 `node api/server/index.js`，不是上面說不會啟動引擎的「legacy experimental clustered entrypoint」。
> - 沒有 Redis。
>
> **做法**：把 `.env` 第 1363 行從 `.env.example` 帶過來的 `# SCHEDULES_SINGLE_PROCESS=true` 取消註解，然後 `docker compose up -d`（rag_api 會重建）+ `docker compose restart api`。
>
> **結果**：兩行 error 消失，改為 `info: [schedules] engine started`。啟動 log 至此沒有任何 error，只剩一個選填項目的 warn（`METRICS_SECRET is not set`）。
>
> **要注意**：
> - 這只是讓**引擎**能啟動，**排程功能本身仍是關閉的**：`librechat.yaml` 沒有 `interface.schedules`，使用者看不到 Scheduled chats。要開啟請照上方「Enable」加設定。
> - 日後如果 api 擴充成多個 replica，**必須移除** `SCHEDULES_SINGLE_PROCESS`，改用 `USE_REDIS_STREAMS=true`。否則一個 replica 會把另一個 replica 正在跑的排程誤判為中斷，見上方「Deployment requirements」。
> - 會「暫停等待使用者回答或核准工具」的 agent，即使是單一 process，仍然需要 Redis streams（見上方第二點）。單靠 `SCHEDULES_SINGLE_PROCESS` 不夠。
