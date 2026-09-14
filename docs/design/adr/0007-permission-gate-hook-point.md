# ADR-0007：权限闸的挂点与审批链路

- 状态：**已接受** ✅
- 日期：2026-09-14
- 决策人：项目组
- 相关：[ADR-0002](./0002-embed-pi-as-sdk.md)、[ADR-0006](./0006-subagent-process-model.md)、[permission-gate.md](../permission-gate.md)

> **决策摘要**：闸挂在**扩展层**（`pi.on("tool_call")`），并**复用社区成熟的权限扩展**（工作选择：`@gotgenes/pi-permission-system`），而非自研判定引擎。
> **理由**：避免重复造轮子。自研需要在工具制作与后期维护上投入大量精力与时间，复用社区成熟实现可大幅减少工作量。
> **已知并接受的代价**：扩展链上的 `event.input` 可变且改后不重新校验，因此本闸**不是抗篡改的安全边界**（见下文"负面 / 代价"）。

## 背景

权限闸要拦截每一次工具调用。pi 提供两个候选挂点：

1. **扩展层的 `tool_call` 事件**：`pi.on("tool_call", handler)`，返回 `{ block, reason, terminate }`
2. **SDK 层的 `Agent.beforeToolCall`**：`Agent` 上的公开可赋值字段

读源码后发现两者其实是**同一条调用路径**：`agent-session.ts:479-499` 里 `AgentSession` 把 `agent.beforeToolCall` 装成了 `runner.emitToolCall(...)`，也就是说扩展事件是被 SDK 钩子派发出去的。

**关键问题**：扩展链上的 `event.input` **可以被 handler 原地修改，且改完不再重新校验**（`types.ts:898-903`，`docs/extensions.md:759-766`），后加载的 handler 还能看到前面 handler 改过的参数。官方仓库里有两张 open issue 正在要求一个"最终不可变的准入钩子"，说明维护者也知道现在这个不够——**但那个钩子还不存在**。

结论：**扩展的 handler 链不能当安全边界。**

## 调研补充（2026-09-14，证据汇总）

### 新增第三个选项：**直接替换内置工具**

`openclaw`（用同一份 pi SDK 做产品的成熟项目）的做法（`docs/pi.md:272-283`）：

```typescript
export function splitSdkTools(options) {
  return {
    builtInTools: [],                    // Empty. We override everything
    customTools: toToolDefinitions(options.tools),
  };
}
```

它的工具流水线（`docs/pi.md:241-249`）：pi 基础工具 → **用自有实现替换**（bash 换成受控实现）→ 自有工具 → **策略过滤** → schema 归一 → abort 包装。

**这条路的好处：TOCTOU 问题（参数被改、改后不重校验）从根上消失**——因为工具是我们自己的，参数进到工具里就是最终值。

**成本比想象的低**：pi 导出了工具工厂 `createReadTool` / `createBashTool` / `createEditTool` / `createWriteTool` / `createCodingTools` / `createReadOnlyTools`，官方 `sandbox` 示例正是用 `createBashTool(cwd, { operations })` 覆盖 bash 的——**我们是"包装"，不是"重写"**。

### 与"自研 vs 复用"的耦合（重要）

**生态里所有第三方权限包都挂在扩展层的 `tool_call` 上**（因为它们都是扩展，没得选）。
⇒ **选了方案 C（替换工具），就等于放弃了直接复用这些包**——它们的闸根本没机会运行。
⇒ 反过来说：**如果决定复用 `@gotgenes/pi-permission-system`，那本 ADR 就要选方案 A。**
**这两条决策必须一起拍。**

### 一条实证：审批请求去重/排队**不是**过度设计

pi issue #7007 原文记录了死锁路径：*"a background subagent's forwarded permission dialog gets clobbered by the main agent's own permission dialog, and the subagent then blocks on its permission wait **with no answerable prompt on screen until it times out**."*

⇒ **pi 不做任何 UI 序列化**，主/子共用 UI 上下文时弹窗会互相覆盖。**这正面支撑本 ADR 决策 2（审批请求消息化 + 排队）。**

### 一条实现时序要求

早先的调研（`@gotgenes/pi-subagents` 源码）确认：子会话的扩展运行在**独立的事件总线**上，只有靠 process-global 注册表才能让子会话被识别。它的做法是把 `session-created` **同步发在 `bindExtensions()` 之前**。
⇒ **我们的 SessionFactory 必须照此顺序：先注册进 `ApprovalService`，再 `bindExtensions()`。**（与 [ADR-0006](./0006-subagent-process-model.md) 共用同一条约束。）

### "区分闸坏了 vs 用户拒了"——有现成答案了

`@gotgenes/pi-permission-system` 的原文：转发失败**不伪装成用户拒绝**——*"None of them is reported as a user denial, because **no user was ever asked**"*。它还用一个 **2 秒 grace window** 做快速失败，而不是等满超时。

⇒ 我们照此实现：审批链路的失败分三类，文案各不相同 —— **用户拒绝** / **没人被问到（链路问题）** / **闸自身异常**。

## 决策（已接受）

**走扩展层 + 复用社区扩展。**

1. **闸挂在扩展层**：`pi.on("tool_call", handler)`（生态标准做法）
2. **判定引擎复用社区实现**：工作选择 `@gotgenes/pi-permission-system`（7 条需求命中 6 条）
3. **审批请求仍走我们自己的 `ApprovalService`**：扩展的 `ctx.ui.confirm()` 最终落到我们实现的 `ExtensionUIContext` → IPC → GUI。`ApprovalService` 负责**排队仲裁**（因为 pi 不做 UI 序列化，见下方实证）

### 复用这个包之后，仍然由我们负责的部分

| 项 | 说明 |
|---|---|
| **`ExtensionUIContext` 实现** | 包只负责调 `ctx.ui.*`；把这个调用接到 Electron GUI 上的是我们（这是 TUI / RPC 之外的第**三**份实现） |
| **三档用户界面（需求 4）** | 没有任何候选包提供，必须自建 |
| **灰区判官的挂载** | 包只提供 `authorizerChain` 插槽，不内置模型判定。⚠️ 已知 `pi-permission-ai-guard` 的 peer 窗口 `>=27.1.1 <32.0.0` 与当前 32.0.2 **不兼容**，需改用 `@gotgenes/pi-permission-model-judge` 或自写一个 authorizer link |
| **启动自检** | 闸扩展加载失败 = 静默无闸运行（`loader.ts` 只把错误 push 进 `errors[]`）。必须自己断言并拒绝启动 |
| **供应链锁定** | 该包 4 个月内发了 210 个版本、有破坏性变更。**必须锁死 scoped 名 + 精确版本** |
| **`pi -p` 绕过洞** | 子代理可用 bash 起新 agent 进程绕过闸；包不管这件事，要我们自己加规则 |

## 备选方案

| 方案 | 优点 | 缺点 | 为什么没选 |
|---|---|---|---|
| **A. 扩展层 `tool_call` + 复用社区包**（**已选**） | 生态标准；**可直接复用社区成熟实现，省掉自研判定引擎的全部工作量** | **`event.input` 可变、改后不重校验 → 不是抗篡改的安全边界**；受包的版本节奏牵制 | ✅ 采用 |
| B. SDK 钩子 `Agent.beforeToolCall` | 看原始参数、不可被 handler 篡改 | 要包装 `AgentSession` 已装的钩子；**用不了现成第三方闸 → 判定引擎要自研** | 与"复用"的决策冲突 |
| C. 替换内置工具（`builtInTools: []` + `customTools`） | TOCTOU 从根上消失；有 OpenClaw 先例 | 要自己组装全部工具；**同样用不了现成第三方闸** | 同上 |
| D. 等官方"最终准入钩子" | 官方背书 | **该钩子尚不存在**（两张 open issue，均被 bot 关闭、无维护者表态） | 不能对着空气编码 |

**结论**：选择 A 的核心理由是**工作量**——自研判定引擎需要在规则引擎、命令解析、路径归一化、审批链路、跨会话注册上持续投入，而复用现成实现能把这些一次性省掉。

⚠️ **备选表保留在此**，供将来条件变化时重新审视（触发条件见文末）。

## 后果

### 正面
- 判定点看到的是原始参数；批准链路与进程模型解耦（审批是消息，不是函数调用），将来换成 RPC 子进程时 GUI 侧不用改
- 弹窗可标明 `actor`（`main` / `subagent:检索-1`），满足"子代理动作可见"

### 负面 / 代价
- 需要包装 `AgentSession` 已安装的钩子，属于对内部实现的依赖，pi 升级时要复核
- 闸自身的未捕获异常会导致工具被拒（fail-closed 是好事），但要**能区分"闸坏了"和"用户拒了"**，否则给学生误导性提示

### 需要后续跟进的事
- [ ] 写清"包装已有钩子"的代码范式，并加一条回归测试（防止 pi 升级后 `AgentSession` 改变安装时机）
- [ ] 定义审批请求 / 回执的字段（见 [ipc-contract.md](../ipc-contract.md)）
- [ ] 明确"闸异常"与"用户拒绝"在 GUI 上的不同呈现

## 什么情况下该重新审视

- 如果 pi 官方落地了"最终不可变准入钩子"（关注官方 issue）→ 可以简化为单层
- 如果实测发现包装钩子会与 `bindExtensions` / 扩展重载冲突

## 参考

- `packages/coding-agent/src/core/agent-session.ts:479-499`（`_installAgentToolHooks`，注释说明钩子只装一次、扩展重载不重装）
- `packages/agent/src/types.ts:60-106`（`BeforeToolCallResult` / `BeforeToolCallContext` 精确契约）
- `packages/agent/src/types.ts:898-903`、`packages/coding-agent/docs/extensions.md:759-766`（`event.input` 可变且不重校验）
- `packages/coding-agent/src/core/agent-session.ts:306`（`readonly agent` —— 只锁字段不锁对象）
