# 学习任务

个人学习笔记，不进正式文档体系（无 i18n 配对、不在 doc-budgets manifest）。

架构笔记：[architecture.md](architecture.md)（总览 + turn/step）· [packages-and-seams.md](packages-and-seams.md)（包全景与读法）。

小白路线：按阶段从上往下走；除标注 `DEEPSEEK_API_KEY` 的项外都不需要 key。每段末尾一行 **过关** 是验收线，能复述或跑通才算过。

## 前置准备

- [ ] 装好 Node.js（`^22.19 || >=24`）与 pnpm，`node -v` / `pnpm -v` 验证；完整清单看 [`docs/development.md`](../development.md#setup-tutorial) "Setup tutorial"。
- [ ] git 日常操作：clone/status/diff/log + 建分支；参考 [`docs/development.md`](../development.md#git-integrations) "Git integrations"。

**过关：** `node -v` 与 `pnpm -v` 都返回，scratch 目录里能 clone / status / diff / log 不报错。

## 第一次跑起来

- [ ] 跑通一次：`pnpm install` → `pnpm dsh --profile headless "task"`（需要 `DEEPSEEK_API_KEY`）。
- [ ] 打开 Web UI（`pnpm dsh --profile web`，命令打印访问地址）：设置里填 API key、选工作区、发第一条消息——[`docs/user/guide/index.md`](../user/guide/index.md)。

**过关：** headless one-shot 出结果；Web UI 能发出第一条 assistant reply，session log 里能看到 `user/message` 与 `assistant/*` 事件。

## 框架基础（Cordis）

- [ ] 读 [`docs/cordis-primer.md`](../cordis-primer.md)：waterfall、`ctx.effect()` / `ctx.on()` 语义。
- [ ] Cordis tutorial 七章动手做完：scratch 目录 `tmp/cordis-tutorial`，全程无 key；重点 ch3 Services / ch4 Events / ch5 Config——[`docs/cordis-tutorial/index.md`](../cordis-tutorial/index.md)。

**过关：** 能不看书说出 waterfall 与 single-decision 两种 dispatch mode 的差别，以及 `ctx.on()`(waterfall listener) vs `ctx.effect()`(reversible registration) 各自装的是什么。

## 架构与读码

- [ ] 通读 `../../AGENTS.md`：仓库布局、包分组、常用命令。
- [ ] 精读 [`docs/architecture.md`](../architecture.md)：plugin spine、loop、seam 的关系图。
- [ ] 建词汇表：[`docs/glossary.md`](../glossary.md) 的术语逐条过。
- [ ] 看一个完整例子：按 [`packages/README.md`](../../packages/README.md) 挑一个 package，对照其源码走一遍 Service Definition / Provider / Consumer 三角色；推荐顺序 `core/session` → `core/tools` → `core/agent-loop`（[读法](packages-and-seams.md)）。
- [ ] 看本机启动合成树：`pnpm dsh --profile web --dump-config`，每一行都可以被自己的 patch 替换。

**过关：** 能不看书复述 profile→bundle→home `cordis.patch.yml`→user `--patch` overlay 的合成顺序,以及 patch 按 id 定位行、整行替换 config 或插入新行的规则；并能说出 fs 的三角色各自落在哪个文件、换 provider 后为什么 Bash/PTY/LSP 跟着走。

## 动手（练手）

- [ ] tutorial ch7 做完：register a model-callable tool against real harness services——[`docs/cordis-tutorial/07-into-the-harness.md`](../cordis-tutorial/07-into-the-harness.md)。
- [ ] 按 [`docs/user/develop/basic/index.md`](../user/develop/basic/index.md) 写出第一个 Harness plugin（plugin → config → publish）。
- [ ] 跑一次 demo：`pnpm run demo:cordis`（需要 key）；其余选项见 development.md "Demos"。

**capstone：** 从干净 `tmp/my-plugin` 起，写一个最小 plugin(注册一个 model-facing tool 或 context section)，同时确认它 ① Web UI 里能用、② `pnpm dsh --profile web --dump-config` 打印的树里有对应行——能跑通这条"挂到旁边而不是改核心"的路,才算真懂。

**过关：** capstone 跑通；能说出自己那个 plugin 在启动合成里的位置(挂在哪个 bundle 之下、被哪一层 patch 覆盖)。

## 贡献工作流（会写几个 commit 后）

- [ ] [`docs/development.md`](../development.md) 的 "Daily commands" + "CI gates" 通读：什么改动跑什么命令。
- [ ] 理解测试策略：keyless snapshot vs real-API e2e——[`docs/testing.md`](../testing.md)。
- [ ] push 前选最小覆盖这次 diff 的检查（`../../AGENTS.md` "Run relevant checks locally"）。

**过关：** 能说出这次 diff 该跑哪些 check(`test:snapshot` / `test:e2e` / `lint` / `duplication` 的取舍),以及 snapshot vs real-API e2e 各自什么时候用。

## 加餐（可选）

- [ ] Python SDK 入口：[`python/README.md`](../../python/README.md)。
- [ ] 读一篇 postmortem 故事：[`docs/postmortem/README.md`](../postmortem/README.md)。
- [ ] 看包间依赖全貌:[`graph-atlas.md`](../graph-atlas.md) / [`module-graph.md`](../module-graph.md),配 `packages-and-seams.md` 的"分组速览"读。
- [ ] 对照 session log 的持久事实清单:[`persistence-catalog.md`](../persistence-catalog.md),与 `architecture.md` "Session log: model-visible ⟺ logged" 节互参。
