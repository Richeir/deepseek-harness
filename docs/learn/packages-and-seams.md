# 包全景与 Seam 三角色（学习笔记）

个人学习笔记，不进正式文档体系；事实以 [packages/README.md](../../packages/README.md) 的完整分组表为准。配套：[architecture.md](architecture.md)、[task.md](task.md)。

## Seam：什么算一个能力

**seam = 可替换能力的三角色**：**Service Definition** 声明接口，**Service Provider** 实现它，**Consumer** 使用它（常见形态是 model-facing tool）。一个包可以合并多个角色，但只有单角色不构成 seam；加一个能力意味着三个角色一起设计。

为什么值得这个结构：换掉一个 provider 就改变整个产品——filesystem 与 subprocess provider 共享同一个执行世界，所以指向远程 sandbox 时 Bash、PTY、LSP 跟着一起走，没有 provider fork。[subagent](../../packages/subagent/README.md) 同理：从全新子 agent 到另一个产品里的委托 turn，都藏在同一接口后（见 [capability-seams](../capability-seams.md)、[subagent 页](../subsystems/subagent.md)）。

## 分组速览（按角色归类）

官方表是完整 home；下面是学习向的归类，行内链接直达各组 README。

**核心脊柱**（产品 API spine + 词法）：

| 组 | 一句话 |
|---|---|
| [core/](../../packages/core/README.md) | session log、prompt assembly、scoped tool registry + 执行管线、`Agent` 接口与默认 driver、scope 原语 |
| [llm/](../../packages/llm/README.md) | message/stream 词法 + adapter seam（`ctx.llm`） |

**执行与 IO 能力家族**：

| 组 | 一句话 |
|---|---|
| [fs/](../../packages/fs/README.md) | 文件能力：seam、本地实现、model-facing file tools、bash-backed discovery |
| [shell/](../../packages/shell/README.md) | bash 执行 seam + local/pwsh providers + model-facing tool |
| [subprocess/](../../packages/subprocess/README.md) | subprocess 能力 + 本地 process-tree provider |
| [terminal/](../../packages/terminal/README.md) | 持久 PTY：owner-scoped session、本地实现、model-facing tools |
| [lsp/](../../packages/lsp/README.md) | LSP seam + 通用 stdio provider + `lsp` tool |
| [sandbox/](../../packages/sandbox/README.md) | 进程围困 seam：bwrap / Landlock / Seatbelt backends |
| [web/](../../packages/web/README.md) | web 能力：seam、search/fetch providers、model-facing web tools |
| [code-runtime/](../../packages/code-runtime/README.md) | Code-execution seam + worker-thread provider + Code Mode Consumer |
| [workflow/](../../packages/workflow/README.md) | workflow seam + worker-thread 引擎 + `workflow`/`ralph` tools |
| [skill/](../../packages/skill/README.md) | skill registry、本地 provider、model-facing catalog/loader |

**协调与自主性**：

| 组 | 一句话 |
|---|---|
| [subagent/](../../packages/subagent/README.md) | subagent registry contract + model-facing delegation tool |
| [jobs/](../../packages/jobs/README.md) | 通用后台 job runtime + `job_*` 控制 tools |
| [plan/](../../packages/plan/README.md) | plan 协作状态：直接入口命令 + reviewed exit |
| [todo/](../../packages/todo/README.md) | model-facing `todo_write` tool |
| [guard/](../../packages/guard/README.md) | loop-hygiene guards + `tools/execute` deadline enforcer |
| [context/](../../packages/context/README.md) | model-visible request context：workspace instructions、time context |
| [compaction/](../../packages/compaction/README.md) | compaction seam + basic provider + command Consumer |

**应用面与集成**（boot glue、web-GUI 两半、ACP、SDK、interaction/approval、settings/credentials）和**组合层**（bundle、preset、extensions 自修改、hooks）的分组同样以 [官方分组表](../../packages/README.md) 为准。

## 怎么读一个包（方法）

1. **先找 ctx key**：[官方核心表](../architecture.md#core-packages)列了每个脊柱包的 `ctx` key；key 就是它对共享上下文的全部贡献面。
2. **分清角色**：接口在 Service Definition（`src/types.ts` 只放类型），具体实现在 Provider，使用方在 Consumer——读代码时按这三问导航，不要从入口文件顺下来。
3. **看 Model Experience**：model-facing 包的 README 有规范节，写明模型看到什么（prompt、tool schema、结果词法）。这是「模型视角」的第一现场。
4. **看 invariant**：每个包拥有 `./invariant`，断言它的事件/数据关系——那是作者声称的契约的最小可执行形式。
5. **跟一条真实路径**：从事件序列（见 [architecture.md](architecture.md) 的 turn/step）反查代码谁 emit、谁消费。

推荐顺序走读源码：[`core/session`](../../packages/core/session/README.md)（append-only log，一切 durable 事实的家）→ [`core/tools`](../../packages/core/tools/README.md)（registry + guarded execution pipeline）→ [`core/agent-loop`](../../packages/core/agent-loop/README.md)（实现 `Agent` 接口的默认 driver）。这三个读完，其余包都是往 seam 上挂的插件。

## 一个具体例子：fs

- Service Definition：`ctx.fs` 上的接口 + `fs/*` 事件域。
- Provider：本地文件系统实现；换成远程 sandbox provider 后，Consumer 全部跟着走——执行世界是共享的（见 [subprocess/](../../packages/subprocess/README.md)、[sandbox/](../../packages/sandbox/README.md)）。
- Consumer：model-facing file tools + bash-backed discovery tools。
- 深入页：[filesystem](../subsystems/filesystem.md)。
