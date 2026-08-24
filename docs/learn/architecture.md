# 架构学习笔记（dsh）

个人学习笔记，不进正式文档体系；每个事实的 home 是链接到的官方文档。配套：[task.md](task.md)。

## 一句话定位

DeepSeek Harness 是在 vendored Cordis（插件框架）之上的插件式 agent harness：模型适配器、工具注册表、会话日志，甚至 loop 本身都是插件，全部可从配置替换；没有特权核心可以打补丁——扩展方式是把插件挂到别人旁边。

## 分层

```text
入口            dsh CLI：web GUI、headless one-shot、ACP server
profile+bundle 启动合成层：决定哪些插件真正被装载
packages/<g>   能力包群：core spine + capability family（三角色拆分）
vendor/       Cordis 框架源码的 pinned copy
```

- `vendor/` 不是安装的依赖，是 pinned source copy（manifest + upstream SHAs in [vendor/README.md](../../vendor/README.md)）。
- 每个包叫 `@deepseek-ai/dsh-*`、ESM；`@deepseek-ai/cordis` 是它的 peerDependency + devDependency。跨包用包名 import，本地相对 import 写 `.ts`（构建走 tsx 的 ESM-only hook）。

## Cordis：五个概念就够读代码了

来自 [primer](../cordis-primer.md)；细节看 [tutorial](../cordis-tutorial/index.md)。

1. **Plugin** = 带可选 `inject` / `apply(ctx)` 字段的 function，或 `Service` subclass——都向共享 context 贡献 services、typed events、reversible effects。
2. **Context 是服务仓库**：claim 稳定 key（`ctx.tools`、`ctx.llm`、`ctx.sessions`）；别的插件按 key 找服务，而不是 import 具体实现。
3. **依赖用 `inject` 声明**：load order 由 service requirements 表达，不是手写启动顺序。
4. **Typed events 通信**：declaration merging 声明事件名；四种 dispatch mode——emit / waterfall / parallel / serial（`@mode` 是 public contract 的一部分）。
5. **注册是可回滚的 effect**：prompt sections、tool schemas、adapters、providers、listeners 都经 `ctx.effect()` / `ctx.on()` 安装，reload 与 teardown 按序 unwind。

**Waterfall（最容易踩的语义）**：listener 收到 `(...args, next)`；调 `next()` 才把（可能包过的）结果委托给下一个 service，值从 `next()` 的返回传回来；不调就短路链。single-decision 事件上短路是设计本身——policy 拥有决定权时可以直接返回；只观察/标注的 listener 必须 delegate。

## 启动合成：profile + bundle

- **profile** = Harness home 下的命名组合：列出它叠放的 bundles、持有 out-of-tree 插件安装、保存用户自己的 `cordis.patch.yml`；`web` / `headless` 作为模板发货。
- **bundle** = cordis config rows + 挂载代码的发行形式，保证上层 patch 仍能覆盖它插入的行。每个 bundle 在自己的 package.json 里用 `dsh.profile`（profile 列出的 bundles）/ `dsh.bundle`（指向 patch 文件）自声明。
- 层按顺序应用到空 entry list：[`dsh-base`](../../packages/bundle/base/README.md)（每个 profile 的第一层：model adapters、tools、persistence、sandbox + approval policy、settings、credentials、telemetry）→ [`dsh-web-app`](../../packages/bundle/web-app/README.md) 或 [`dsh-headless`](../../packages/bundle/headless/README.md) → profile 的 `cordis.patch.yml` → home-level patch → `--patch` overlay。patch 按 id 定位行，整行替换 config 或插入新行。
- 看本机实际启动的树：`dsh --profile web --dump-config`——它打印的每一行都可以被自己的 patch 替换。合成机制见 [app-boot](../../packages/boot/app-boot/README.md)；字段目录是生成的 [config-catalog](../config-catalog.md)。

## Core packages（spine）

| Package | Owns | ctx key |
|---|---|---|
| [`core/session`](../subsystems/session.md) | append-only `SessionEvent` log + in-memory store | `ctx.sessions` |
| [`core/system-prompt`](../subsystems/system-prompt.md) | prompt-section 与 tool-schema assembly | `ctx.systemPrompt` |
| [`core/tools`](../subsystems/tools.md) | scoped tool registry + guarded execution pipeline | `ctx.tools` |
| [`core/agent`](../subsystems/core.md) | `Agent` 接口、live registry、`agent/*` events | `ctx.agents` |
| [`core/agent-loop`](../subsystems/core.md) | 实现该接口的默认 driver | `ctx.agentLoop` |
| [`core/scope`](../subsystems/scope.md) | per-agent scoped-registration 原语（库，无 key）(scoped-registration primitive, library only) | — |
| [`llm/llm`](../subsystems/llm-streaming.md) | message/stream 词法 + adapter seam | `ctx.llm` |

## Loop：turn 与 step

**step** = 一次模型请求 + 它调用的工具；**turn** = 零个或多个 steps，在首个 input 被认领前打开，在没有欠账时关闭。

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   reject | enter(messages)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     append entered messages as user/message
     derive model history from the log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

- `turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是**持久 session events**；其余是三个域上的 live extension points（session / agent / capability）。
- `agent/pre-step`、`agent/request`、`llm/stream`、三个 `tools/*` 事件是 **waterfall**（listener 必须调 `next()`）；`agent/turn-stopping` 是 serial，没有 `next()`。
- input 经一个 inbox 到达 driver：部分消息立即唤醒它，injected context 在 inbox 里等到另一条消息到来才动。
- `agent/pre-step` 决定模型看到什么：listener 可改写被认领的消息或直接拒绝；被拒或空的 first claim 仍关一个没花 step 的 durable turn——log 记下这次尝试。

细节看 [sequence diagram](../agent-lifecycle.md) 与 [tool pipeline](../tool-execution-pipeline.md)。

## Session log：model-visible ⟺ logged

session log 是模型所见上下文的来源：`deriveMessages()` 从它投影 model history；raw `assistant/chunk` events 保留 replay 与 UI fidelity。fork、resume、transcripts、telemetry、persistence 全部从这个流派生。

**到达一次模型请求的东西必须能从 log 重建**——runtime invariant 断言这条。所以新的 model-visible input 需要一个新的 session event：扩展 `SessionEventMap`，从 log render。「加上下文」因此总是「加事件 + 加渲染」。

## Capability seams（三角色）

**seam = 可替换能力的三个角色**：**Service Definition** 声明接口、**Service Provider** 实现它、**Consumer** 使用它（通常是 model-facing tool）。一个包可以合并角色，但单角色不构成 seam；加一个能力意味着三个角色一起设计（[capability graph](../capability-seams.md)）。

seam 是「换掉一个 provider 就改变整个产品」的原因：filesystem 与 subprocess providers 共享同一个执行世界，所以指向远程 sandbox 时 Bash、PTY、LSP 跟着走，没有 fork。subagent providers 在同一个接口后同样宽（[page](../subsystems/subagent.md)）。实验性的 [Agent Teams](../subsystems/agent-team.md) 是 `ctx.agentTeams` 上的 opt-in 协调 seam：durable roster + task board + mailbox，叠在 continuable subagents 上。

## 加功能去哪（浓缩版）

完整版在 [官方 architecture.md](../architecture.md)。常用行：

| 目标 | 机制 |
|---|---|
| 加 model provider | register adapter on `ctx.llm` |
| 加 model-facing capability | register on `ctx.tools`；schema 自动加入 prompt assembly |
| 给一个 session 换能力集 | compose agent preset（service row 需要 `isolate` realm）|
| 加 shell / persistent terminal 执行 | register `ctx.shell` backend（local 经 `ctx.subprocess` spawn）/ `ctx.terminals` backend + `dsh-tool-terminal` |
| 加 human command / background work | register on `ctx.commands`（不花 model turn 分发）/ `ctx.jobs`（`job_*` tools 收集或停止）|
| 加 filesystem access / policy | register `ctx.fs` provider 或 listen to `fs/*` events；围困 spawn 用 `ctx.sandbox` backend |
| 拦截 request / tool / turn | 对应 `agent/*` 或 `tools/*` event；`agent/turn-stopping` stops a turn |
| 加 model-facing context | call `agent.inject()`，落到下一次 admitted request |
| UI / editor integration | drive `ctx.agents` + render from `session/event` |
| durable session state | extend `SessionEventMap`；render and replay from the log |
| fork a live session | `ctx.sessions.fork(source, boundary?, childSessionId?)` |

## 阅读顺序

1. [task.md](task.md)：checklist。
2. 本文档按节读；每节的 home 链接进官方文档，再进源码。
3. 挑一个 core package（建议 `core/session`），读它的 subsystem page + 包内 README，对照 turn flow 看事件从哪 emit。
