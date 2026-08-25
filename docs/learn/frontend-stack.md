# 前端技术栈与代码位置（学习笔记）

个人学习笔记，不进正式文档体系；规范细节以 [packages/client/AGENTS.md](../../packages/client/AGENTS.md) 为准。配套：[architecture.md](architecture.md)、[packages-and-seams.md](packages-and-seams.md)。

## 结论一句话

- **框架**：React 18（TypeScript / `.tsx`），构建用 Vite，样式走 CSS Modules + `--dsw-*` token（无 Tailwind、无组件库）。
- **业务代码**：全在 `packages/client/*`——按「一个 UI 功能 = 一个插件包」拆成几十个包。
- **可运行的 web app**：不是某个目录里的大工程，而是由几个壳层拼出来的——下面这张表就是它的家在哪。

## 谁放什么（从外到内）

| 位置 | 角色 |
|---|---|
| `apps/web/` | Vite host + **极薄** bootstrap：`main.ts` 只找 `#root`，调 `AppWebEntry(el).run()`；平台模块表在 `packages/client/web/src/platform.ts` |
| [`packages/client/web`](../../packages/client/web/) | boot kernel（`AppWebEntry`）：两段挂载——先 seed 模块表、prefetch，再把 DOM 交给 ui-renderer hydrate |
| [`packages/bundle/web-app`](../../packages/bundle/web-app/) | **web app 组装**：把各 `ui-*` 包按 cordis.patch.yml 拼成可跑的应用（`dsh web`） |
| `packages/client/runtime/` | **数据层**（React-free）：Connection → SessionManager → Session，全业务 state + zustand store 引擎 |
| [`packages/client/ui-renderer`](../../packages/client/ui-renderer/) | **渲染机制**：slot renderer/outlet、SessionProvider、所有 hook 在这里绑定 |

## UI 组件包速览（挑代表性的看）

| 包 | 一句话 |
|---|---|
| [`ui-conversation`](../../packages/client/ui-conversation/) | 主对话界面：ChatView / MessageItem / InputBar / ReasoningRow… |
| [`ui-layout`](../../packages/client/ui-layout/) | `AppFrame`——整页骨架（shell 只渲染 `'root'`） |
| [`ui-attachment`](../../packages/client/ui-attachment/)、[`ui-sidebar`](../../packages/client/ui-sidebar/)、[`ui-model-selection`](../../packages/client/ui-model-selection/) | 侧栏 / 附件 / 模型选择等局部域 |
| [`ui-primitives`](../../packages/client/ui-primitives/)、[`ui-theme`](../../packages/client/ui-theme/) | 基础组件 + `--dsw-*` token（全局 sheet）与 CSS Modules |

## 怎么读一个 UI 包（方法）

1. **别从入口顺**——看 `apply.ts`：所有 slot 都通过 `ctx.slots.register({ name, children?, store? }, Component)` 挂进去，没有模块级副作用。
2. **分清角色**：业务组件只吃 props；要新数据 → owner props / declared store / inject callback，绝不自己订阅或读 React context（见 [AGENTS.md](../../packages/client/AGENTS.md)「ctx discipline」）。
3. **看四层红线**：data object layer（runtime）→ render machinery（ui-renderer）→ presentation（各 `src/client/`）；单向知识流，别跨层拿东西。

## 跑 / 验

- GUI 改动先跑 `pnpm run test:gui`（无浏览器、无 server 的内环）。
- 改了可见输出再 `DSH_SNAPSHOT=replay pnpm run test:web`（重建 dist + 浏览器 smoke，缺 key 时 real-host 那条自跳过）。
