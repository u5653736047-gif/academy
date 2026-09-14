# ADR-0006：子代理的进程模型

- 状态：**已接受** ✅
- 日期：2026-09-14
- 决策人：项目组
- 相关：[ADR-0002](./0002-embed-pi-as-sdk.md)、[ADR-0007](./0007-permission-gate-hook-point.md)、[architecture.md](../architecture.md) 第 3.3 节

> **决策摘要**：**不自己实现子代理**，复用社区成熟扩展——工作选择 `@gotgenes/pi-subagents`（配 `@gotgenes/pi-permission-system`），因为它是唯一实现"子会话的 `ask` 转发到父会话 UI"的方案。
> **理由**：与 ADR-0007 一致——避免重复造轮子，复用社区实现以减少工作量。
> **因此本 ADR 的角色从"选一个方案自己做"变成"选一个包并接好它"**：下面的调研结论仍然全部有效，但用途从"实现指南"转为"选型依据 + 集成注意事项"。

> ⚠️ **这条不拍板，架构文档 3.3 节和权限闸的"主/子共用同一道闸"就写不实。**
> 注意：架构文档 v0.1 第 3.3 节和决策 D2 目前写的是"subagent 是独立 pi 进程"，**那是在没有验证 pi 官方 subagent 扩展的前提下写的。本 ADR 是对它的挑战。**

## 背景

需求要求多智能体第一版就上，分工是：主 agent + 通用子代理基座 + 资料检索子代理。其中检索子代理的**实际动机是当上下文防火墙**——检索回来的片段可能很长，在它自己的上下文里消化掉，只回主 agent 一份带出处的摘要。

需求还有一条硬约束（需求文档第 2 节）：

> 改学生文件、跑命令，不管是主 agent 还是 subagent 干的，**都走同一道权限闸**；subagent 的动作在界面上和主 agent 一样可见、可中止。

架构文档因此把子代理设计成"独立 pi 进程，走 pi 官方扩展机制"。**但这个假设没有验证过。**

### 验证结果：官方 subagent 扩展的方案在我们的场景里是错的

读了 `packages/coding-agent/examples/extensions/subagent/index.ts`（1015 行）后确认：

1. **它是真子进程**（`:335-339` `spawn(...)`）
2. **父会话的拦截钩子不会被子进程继承**——子进程启动参数是 `["--mode","json","-p","--no-session"]`（`:294`），只追加 `--model` / `--tools` / `--append-system-prompt`，**没有 `-e`、没有传父的扩展列表、没有传 env**
3. **子进程里 `ctx.hasUI` 恒为 `false`**（`--mode json` / `-p` 的行为，见 `docs/extensions.md:2895-2900`）
4. 于是子进程里的闸只有两种可能：
   - 照官方 `permission-gate.ts` 的语义（无 UI 则 block）→ **子代理的 bash/write 全被拒，任务直接失败**
   - 无 UI 则放行 → **这就是提权绕过漏洞**：主 agent 把危险命令包装成子代理任务即可绕过闸
5. 另一条绕过路径：非交互模式不弹信任提示（`docs/security.md:29`），无已保存决策时 `.pi/extensions` 下的项目级扩展**默认不加载**

结论：**照抄官方 subagent 示例 = 要么子代理废掉，要么闸被绕过。**

## 佐证：OpenClaw 的选择（强证据，2026-09-14 补）

`openclaw`（npm 上 255 个版本、latest 已到 2026.9.4 的成熟产品）**用同一份 pi SDK 做产品**，它的子代理是**进程内**的：

| 结论 | 证据 |
|---|---|
| 进程内，不是 spawn | `docs/tools/subagents.md:273-286`：*"Sub-agents still share the same gateway process resources"*、*"a dedicated **in-process queue lane**"*；运行注册表是内存 Map |
| 它**同时**实现了子进程方案（ACP），但**把进程内定为默认** | `docs/tools/acp-agents.md:13,57`；`sessions_spawn` 默认 `runtime: "subagent"` |
| 主/子**共用同一个工具装配工厂**，靠 `sessionKey` 参数区分 | `createOpenClawCodingTools(options)`；subagent 策略是策略管线的**最后一步** |
| **"只能收紧、不能放宽"由结构保证** | `isToolAllowedByPolicies = policies.every(p => isToolAllowedByPolicyName(name, p))` |
| 子代理的"询问"不会绕过闸 | 主/子共用同一份 `createExecTool`；`askFallback` 默认 **deny** |

**这消除了本 ADR 最大的不确定性**：进程内方案在一个真实产品里跑通了，而且它的工具策略管线（单一工厂 + 层层收窄的 `{allow, deny}` 数组）正是"共用同一道闸 + 额外收紧"的最干净实现。

### 需要修正本 ADR 草案的两处

1. **子代理上下文不该用 `SessionManager.inMemory()`。** OpenClaw 的选择相反：子代理转录**落盘**（可被 `sessions_history` 回读、`/subagents log` 查看），靠 `archiveAfterMinutes`（默认 60 分钟）自动归档。
   **理由值得借鉴**：GUI 上"子代理动作可见"要求转录**可回读**，纯内存会丢掉已完成子代理的细节。
   → 按此修正：**落盘到独立目录 + 不进主会话列表 + 保留归档策略。**
2. **进程内不解决"卡死"。** OpenClaw 的中止同样是**协作式**的（`abortEmbeddedPiRun` + AbortSignal 包裹），它没有给出更优解，只靠 `runTimeoutSeconds` + 并发上限兜。
   → 本 ADR「需要后续跟进」第 2 条（卡死兜底）**仍然没有现成答案**。

### 可借鉴的并发模型（比"设一个并发上限"完整）

| 旋钮 | OpenClaw 默认 | 维度 |
|---|---|---|
| `maxConcurrent`（lane `subagent`） | 8 | 全局 |
| `maxChildrenPerAgent` | 5 | 单个父会话 |
| `maxSpawnDepth` | 1（推荐 2） | 嵌套深度 |

外加**级联中止**（`cascadeKillChildren` 递归 + 清队列），以及**"完成即 announce、禁止轮询"**——任务消息里硬写 *"do not busy-poll for status"*，且 `Status` 取**运行时结局**而非模型自述，避免子代理谎报成功。

### 一个我们可能要自己做的部分

OpenClaw 全库检索 `过程树 / process tree / spawn tree / run tree` **零命中**：它的界面只到"聊天里流式显示工具调用 + 工具输出卡片"。
⇒ **"子代理过程树"没有现成实现可抄**，这是我们自己要设计的部分（也可能成为产品的差异点）。

### 一个不能假设它已解决的点

子代理触发的审批在界面上如何**标注归属**（是哪个子代理发的请求），以及能否**按子代理单独中止一条待审批** —— OpenClaw 文档未涉及（`docs/tools/exec-approvals.md` 只列了 agent id，没有 session / 子代理维度）。这块我们要自己做。

## 佐证二：pi 官方的能力边界（2026-09-14 补，源码级）

### 官方**没有**内建子代理，这是有意留白

`packages/coding-agent/README.md:500` 原文：

> *"**No sub-agents.** There's many ways to do this. Spawn pi instances via tmux, or build your own with extensions, or install a package that does it your way."*

`docs/usage.md:303` 同义。官方 subagent **只是扩展示例**（`examples/extensions/subagent/`），不是内建能力。

### "同进程建多个 `AgentSession`" 技术上成立

- `createAgentSession()` 全函数无模块级会话单例，所有状态都是函数局部变量（`src/core/sdk.ts:169-398`）
- 官方示例已经在**同一个进程里连开多个 session**（`examples/sdk/11-sessions.ts`、`05-tools.ts` 等，但都是**顺序**创建 + dispose，**没有任何并发示例**）
- 第三方已在生产使用：`pi-subagents`（月下载 42.8 万）文档原文：*"Children are **pi sessions inside the parent process** … **not separate `pi` binaries**"*
- GitHub issue #6480 原文：*"**In-process subagents (SDK `AgentSession`s) already run fine in the background**"*
- GitHub issue #7808 给出了和我们 ADR 提议**几乎一样**的做法（`createAgentSession` + `DefaultResourceLoader` + `SessionManager.inMemory()` + `SettingsManager.inMemory()` + 共享 `ModelRuntime`），并称 *"~150 lines of subtle loader configuration"*

### ⚠️ 三个硬坑（直接改实现方式，必须写进设计）

**坑 1：绝对不能给主/子会话共享同一个 `ResourceLoader`。**

`AgentSession` 每个会话新建自己的 `ExtensionRunner`，但**复用同一个 `extensionsResult.runtime`**（`agent-session.ts:2580-2593`）；`runner.ts:323-324` 注释明说 *"Copy actions into the shared runtime (all extension APIs reference this)"*，而这些闭包**捕获的是某个具体 `AgentSession` 的 `this`**。

⇒ 共享 ResourceLoader 会让会话 B 的 `bindCore()` **覆盖会话 A 的扩展 API 路由**（A 的 `pi.setActiveTools()` 打到 B 上）。
⇒ **正确做法：每个 session 一个 `DefaultResourceLoader`（即不传时的默认行为），但共享一个 `ModelRuntime`**（省 auth/models 重复加载，官方 quick start 亦如此）。
⇒ 特别注意 `createAgentSessionFromServices()` 会把 `services.resourceLoader` 原样透传——**用同一份 services 调两次就会踩坑**。

**坑 2：闸的状态不能放扩展模块顶层。**

扩展 `.ts` 文件由 `jiti` 加载，**模块顶层变量在进程内只求值一次**，会被所有会话共享。若把权限档位、批准记忆放在扩展模块顶层，多个子代理会串味。
⇒ **闸的状态必须挂在 per-session 的 `Extension` 实例或闭包上。**

**坑 3：pi 不做任何 UI 序列化，主/子同时弹窗会互相覆盖。**

GitHub issue #7007 原文记录了死锁路径：*"a background subagent's forwarded permission dialog … gets clobbered by the main agent's own permission dialog, and the subagent then blocks on its permission wait **with no answerable prompt on screen until it times out**."*

⇒ **这正面证明：审批弹窗的排队（arbiter）必须自研**，不能指望共用 UI 上下文就完事。
⇒ 本 ADR 提议的"**审批请求消息化 + 走 `ApprovalService` 排队**"正好治这个——现在有了实证依据。

### "共用同一道闸"没有任何先例可抄

- 官方不提供任何"子会话继承父授权"的机制
- 下载量最大的第三方 `pi-subagents` **自建了一套独立的子代理权限系统**，其 `watchdog.md` 原文：审批请求发给 *"a one-call arbiter **owned by the child watchdog**"*，且 *"**does not notify the parent**"*；并且 *"Bash is always passed through; bash rules are rejected."*
- ⇒ **需求 5（主/子共用同一道闸）在 pi 生态里是空白**，我们必须自己做，没有现成实现可抄。

### 顺带：官方示例有一个已确认的 bug

`examples/extensions/subagent/index.ts:373` 监听 `tool_result_end` 事件——**该事件在 pi 里根本不存在**（issue #9436：*"Pi never emits this event … so the branch is unreachable dead code"*）。**照抄这个示例时不要连死代码一起抄。**

## 决策（已接受）

**不自己实现子代理，复用社区扩展。**

- **工作选择**：`pi-subagents`（nicobailon）—— **生态第一**，月下载 428,818，3582★，几乎每日提交
- **选择理由**：决策人本人在深度使用这个包，对它的行为有第一手经验；且它是生态里功能最完整、维护最活跃的实现
- **进程模型**：它是**混合**的——前台子代理跑在父进程内，后台子代理跑在 detached runner 进程内。**因此我们不需要自己决定进程模型**；本 ADR 下方的调研结论转为**选型依据 + 集成注意事项**保留

### ⚠️ 这个选择带来的一个未闭合问题（需求 5）

`pi-subagents` 的权限能力与我们的需求 5 有已知偏差：

| 项 | 现状 |
|---|---|
| 子代理的 `ask` 是否转发到父会话 UI | **前台进程内子会话：不转发**；后台独立进程：**会设 `PI_SUBAGENT_PARENT_SESSION`**（`src/extension/index.ts:975`），从而可与 `@gotgenes/pi-permission-system` 的 out-of-process 转发通道对接 |
| 它自己的 `ask` 判定 | 交给 **child watchdog 的一次性 arbiter 模型**，文档原文 *"does not notify the parent"* |
| **bash 策略** | **明确不管** —— `src/runs/shared/permissions.ts:20`：*"pi-subagents leaves bash policy to pi-guard"*，即 bash 一律放行 |

⇒ **需求 5（主/子共用同一道闸）不能靠这个包独立满足，必须靠 `@gotgenes/pi-permission-system` 的转发通道补上，且要实测前台 in-process 子会话那条路径是否真的接通。**

> 注：`@gotgenes/pi-permission-system` 的选型一致性表把 `pi-subagents` 标为 "subprocess" 且 "✗ Sets no parent-session variable"，**与源码不符**（源码里确实设了）。不要照抄那张表。

### 为什么当初考虑过但没选的 `@gotgenes/pi-subagents`

| | `pi-subagents`（**已选**） | `@gotgenes/pi-subagents` |
|---|---|---|
| 月下载 | **428,818**（生态第一） | 12,977 |
| 进程模型 | 前台进程内 / 后台独立进程（混合） | 纯进程内 |
| 功能完整度 | 高（并发/超时/进程树终止/可观测/结构化委派 API） | 中 |
| 维护活跃度 | 几乎每日提交 | 近每日 |
| `ask` 转发到父 UI | 需配合权限包的通道（**待实测**） | ✅ 原生支持 |
| bash 策略 | 明确不管 | 走权限包 |
| 决策 | ✅ **采用**（决策人有第一手使用经验） | 备选 |

### 复用之后，仍然由我们负责的部分

| 项 | 说明 |
|---|---|
| **子代理定义文件** | markdown + YAML frontmatter（`.pi/agents/<name>.md`）——**内容由我们写**（检索子代理、通用基座） |
| **`task` 工具的接入** | 让主 agent 知道"什么时候该派子代理"：系统提示词 + agent 描述 |
| **调度参数** | 并发上限、超时、嵌套深度——按我们的场景定值（包只给默认值） |
| **GUI 过程树** | 包有生命周期事件，但**没有任何现成的"子代理过程树"UI**（OpenClaw 也没有）——这是我们的活，也可能是产品差异点 |
| **审批弹窗的归属标注** | "是哪个子代理发来的请求"——包与 OpenClaw 都未覆盖 |
| **`pi -p` 绕过洞** | 必须自己补规则（见下文） |

### 集成时注意（来自调研的硬约束）

1. **`ask` 转发是文件轮询，不是函数调用**：轮询间隔 250ms、超时 10 分钟，请求/响应是 JSON 文件。Electron 宿主下要评估这套 IO 的开销
2. **闸绑定的是 TUI 对话框**：我们要自己实现"serving node"（即 `ExtensionUIContext`）
3. **子会话不会自动继承父的扩展钩子**——包靠 `subagents:child:session-created` 事件同步注册解决，**我们接入时必须保证这个时序**
4. **运行时 `exports` 指向 `.ts` 源码**（靠 pi 的 jiti 加载）：Electron 打包链要验证能否正常加载；**这条要在技术验证阶段第一个验**
5. **版本锁定**：`@gotgenes/*` 迭代极快（4 个月 210 + 176 个版本，有破坏性变更），必须锁精确版本

## 佐证三：npm 生态盘点（2026-09-14 补）

在 npm 上识别出 **200+ 个** pi 子代理相关包（同名、fork、抢注极多）。**逐项读源码确证进程模型**的：

| 进程模型 | 数量 | 代表 |
|---|---|---|
| **进程内** | **约 18 个** | `@gotgenes/pi-subagents`、`@tintinweb/pi-subagents`、`@arhen/pi-core-subagent`、`@bacnh85/pi-subagent`、`@pify/subagent`、`@quintinshaw/pi-dynamic-workflows` |
| **spawn 子进程** | 约 13 个 | `@henryqw/pi-subagent`、`@marks/pi-subagent`、`pi-vigil`、`@mammothb/pi-subagents`（tmux） |
| **混合** | 1 个（但**下载量第一**） | `pi-subagents`（nicobailon），86,722 周下载 / 428,818 月下载 |

⇒ **进程内是生态主流**，且下载量最大的包在**前台路径上也是进程内**的。这进一步支持本 ADR 的提议方向。

### ⭐ 最重要的发现：混合路线（建议采纳）

`pi-subagents`（生态第一，428K/月）的源码原文（`src/runs/shared/child-session.ts:1-8`）：

> *"**In-process child sessions.** A child is a pi `AgentSession` created **inside the process that owns it**: the **parent pi process for foreground children**, the **detached runner process for background children**."*

**这正面回答了本 ADR「进程内 = 卡死杀不掉」这个缺点**：

| 任务类型 | 进程模型 | 理由 |
|---|---|---|
| 前台、短任务（检索、看代码） | **进程内** | 闸天然共享，启动零成本 |
| 后台、长任务、不受信任务 | **detached 独立进程** | 跑飞了拖不垮主进程，可以真杀（Windows 用 `taskkill /T /F` 杀整棵进程树） |

⇒ **建议：默认进程内，但对长任务/不受信任务提供独立进程通道。** 这比"纯进程内"稳，也比"纯子进程"简单。

### ⚠️ 一个必须堵的绕过洞

`@nicknisi/pi-subagents` 的 README 自己写出来了：

> *"**The recursion guard has a hole.** The in-process depth guard only covers spawns made through the shared runtime; **a child that itself shells out to `pi -p` via bash starts a fresh process with none of that context**."*

**翻译**：子代理只要用 bash 跑一句 `pi -p "..."`，就起了一个**全新的、没有任何闸的进程**。

⇒ 这条必须写进 [permission-cases.md](../../test/permission-cases.md) 的绕过用例（C 组）：**把"通过 bash 启动新 agent 进程"列为黑名单行为**（`pi`、`node <cli>` 等起 agent 的命令）。

### 一条实现时序要求（闸能盖住子代理的充分必要条件）

`@gotgenes/pi-subagents` 源码把 `session-created` 事件**同步发在 `bindExtensions()` 之前**。原因：子会话的扩展运行在**独立的事件总线**上，只有靠 process-global 注册表（`globalThis` + `Symbol.for`）才能让子会话自己被识别。

⇒ **我们的 SessionFactory 必须照此顺序：先把子会话注册进 `ApprovalService`，再 `bindExtensions()`。** 顺序反了，闸就盖不住子代理。

### 生态普遍缺失的两件事（我们要自己补）

1. **超时**：全生态只有 3 处有（`@gotgenes/pi-permission-system` 的 `forwardingTimeoutMs` + 2 秒 grace；`pi-subagents` 的受控 timeout；`pi-subagent-in-memory` 的 60 秒预算并下传）。**绝大多数包只有 turn 上限，没有时间上限。**
2. **并发**：成熟默认值是 **4**（gotgenes）/ **10**（tintinweb）/ **5**（henryqw）/ **8 任务 4 并发**（官方示例）。结合 OpenClaw 的三维上限，我们取 **全局 4 / 单父 3 / 深度 1** 起步。

### 关于直接依赖 `@gotgenes/pi-subagents`：不建议

它架构最值得借鉴（分域架构 + typed service + 完整生命周期事件），但三条硬理由：

1. **运行时 `exports` 指向 `.ts` 源码**（`./src/service/service.ts`），靠 pi 的 jiti 加载；Electron 打包链直接 `import` 会失败
2. **它的闸是另一套**（`@gotgenes/pi-permission-system`），依赖它 = 换掉我们在 ADR-0007 定的硬判定
3. **它的"同一个闸实例"不是自动成立的**——子会话是独立 jiti 实例 + 独立事件总线，必须靠 `globalThis` 注册表桥接

⇒ **结论：自研调度器 + 深度借鉴它的骨架。**

## 备选方案

| 方案 | 优点 | 缺点 | 为什么没选 |
|---|---|---|---|
| **进程内独立实例 + 后台独立进程**（提议，混合） | 闸天然共享；事件/中止是直接调用；无需打包 pi 运行时；上下文隔离仍然成立；**长任务/不受信任务丢到后台独立进程，可以真杀** | 前台仍是协作式中止；要维护两条通道 | — |
| 纯进程内独立实例 | 实现最简单 | **跑飞了拖垮主进程且杀不掉**（OpenClaw 同一个弱点，它也没有更优解） | 混合方案成本接近，收益更大 |
| 官方 spawn 子进程 | 真隔离，崩溃不影响主进程 | **闸不继承**（见上）；需要把 pi 运行时一起打包；每个子代理一个进程 | 直接违反"共用同一道闸" |
| 子进程 + 强制注入闸扩展 | 隔离 + 闸仍在 | 要维护"父进程闸"和"子进程闸"两套状态同步；`--mode json` 下无 UI，弹窗仍要另建通道 | 复杂度换来的隔离，混合方案已覆盖 |
| 不起子代理，主 agent 自己干 | 最简单 | 违背需求"多智能体第一版就上"；检索大结果会撑爆主 agent 上下文 | 需求明确要求 |

## 后果

### 正面
- 需求"共用同一道闸"和"动作可见可中止"**由构造 + 子会话注册协议共同保证**。

  ⚠️ **注意：子会话不会自动继承父的扩展钩子**（tintinweb 源码注释：每个 `AgentSession` 用自己的 `ExtensionRunner` 构建）。所以"共用同一道闸"**不是白送的**，必须显式实现"子会话注册"这一步——顺序是**先注册进 `ApprovalService`，再 `bindExtensions()`**（照 `@gotgenes/pi-subagents` 的 `subagents:child:session-created` 协议）。
- 子代理的启动成本低（无进程创建），可以频繁委派

### 负面 / 代价
- **前台子代理失去进程级隔离**：bash 死循环 / 内存爆炸直接影响主进程。缓解：并发上限 + 超时 + 协作式中止 + **把长任务/不受信任务推到后台独立进程**
- 需要自己实现调度器（并发上限、超时、任务生命周期）

### 需要后续跟进的事
- [ ] 调度器参数：起步值 **全局 4 / 单父 3 / 深度 1**（生态成熟默认：gotgenes 4、tintinweb 10、henryqw 5、官方示例 8 任务 4 并发）
- [ ] **超时机制**：生态普遍缺失（只有 3 处有），必须自研。参考 `@gotgenes/pi-permission-system` 的 `forwardingTimeoutMs` + 2 秒 grace window
- [ ] 子代理卡死的兜底：前台只能协作式；**后台独立进程用进程树终止**（Windows `taskkill /T /F`，参考 `pi-subagents` 与 `@mjakl/pi-subagent` 的 SIGKILL + settle 超时）
- [ ] **堵住 `pi -p` 绕过洞**：子代理用 bash 起新 agent 进程即可绕开一切闸 → 写进 [permission-cases.md](../../test/permission-cases.md) C 组
- [ ] 递归防护：优先用"子会话不加载编排类扩展"（比工具黑名单更硬），辅以 depth 上限
- [ ] 如果实测发现前台进程内不稳定 → 触发本 ADR 的重新审视
- [ ] 架构文档 3.3 节按本条结论重写

## 什么情况下该重新审视

- 如果实测发现子代理频繁拖垮主进程（→ 把更多任务推到后台独立进程）
- 如果将来要支持"关掉窗口后子代理继续跑"这类跨应用生命周期
- 如果 pi 官方推出了"子进程继承父扩展与授权"的机制

## 参考

- `packages/coding-agent/examples/extensions/subagent/index.ts:300`（子进程参数）、`:346`（`spawn`）、`:34`（`MAX_CONCURRENCY = 4`）
  ⚠️ 该文件已增至 **1039 行**，早期引用的 `:294` / `:335-339` 已漂移
- `packages/coding-agent/docs/extensions.md:972-976`（模式与 `ctx.hasUI` 对照表）
  ⚠️ 早期引用的 `:2895-2900` 在新版本已变成 theme colors
- `packages/coding-agent/docs/security.md:29`（非交互模式无信任提示）
- 调研结论：[research/README.md](../../research/README.md)、[research/permission-gate-survey.md](../../research/permission-gate-survey.md)
