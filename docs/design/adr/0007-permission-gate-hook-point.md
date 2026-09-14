# ADR-0007：权限闸的挂点与审批链路

- 状态：**提议中（待拍板）** ⚠️
- 日期：2026-09-14
- 决策人：待定
- 相关：[ADR-0002](./0002-embed-pi-as-sdk.md)、[ADR-0006](./0006-subagent-process-model.md)、[permission-gate.md](../permission-gate.md)

## 背景

权限闸要拦截每一次工具调用。pi 提供两个候选挂点：

1. **扩展层的 `tool_call` 事件**：`pi.on("tool_call", handler)`，返回 `{ block, reason, terminate }`
2. **SDK 层的 `Agent.beforeToolCall`**：`Agent` 上的公开可赋值字段

读源码后发现两者其实是**同一条调用路径**：`agent-session.ts:479-499` 里 `AgentSession` 把 `agent.beforeToolCall` 装成了 `runner.emitToolCall(...)`，也就是说扩展事件是被 SDK 钩子派发出去的。

**关键问题**：扩展链上的 `event.input` **可以被 handler 原地修改，且改完不再重新校验**（`types.ts:898-903`，`docs/extensions.md:759-766`），后加载的 handler 还能看到前面 handler 改过的参数。官方仓库里有两张 open issue 正在要求一个"最终不可变的准入钩子"，说明维护者也知道现在这个不够——**但那个钩子还不存在**。

结论：**扩展的 handler 链不能当安全边界。**

## 决策（提议）

**两层串联**：

1. **硬判定挂在 `Agent.beforeToolCall`**（我们自己的代码，包在最外层）——它先跑，看到的是模型给出的**原始参数**，不受任何 handler 篡改影响。判定完再调用 `AgentSession` 已经装好的那个钩子，把扩展派发继续下去。
2. **审批请求消息化**：闸不直接 `await ctx.ui.confirm()`，而是向一个 `ApprovalService` 发请求、等回执。请求体带 `actor` 字段标明是谁在请求。
3. **子代理与主 agent 用同一个闸实例**（依赖 [ADR-0006](./0006-subagent-process-model.md)）。

## 备选方案

| 方案 | 优点 | 缺点 | 为什么没选 |
|---|---|---|---|
| **SDK 钩子（硬判定）+ 扩展（UI/审计）串联**（提议） | 判定看原始参数、不可被篡改；UI 仍走官方扩展通道；两者都不浪费 | 需要理解"包装已有钩子"的用法，代码上要小心别把 `AgentSession` 装的钩子弄丢 | — |
| 只用扩展 `tool_call` | 完全照官方文档写，最省事 | **handler 链可变、改后不重校验 → 不是安全边界**；顺序依赖加载顺序 | 安全边界不该建在这里 |
| 只用 SDK 钩子，不用扩展系统 | 最可控 | 要自己实现对话框、状态显示、审计入口，等于重写半个扩展系统 | 重复造轮子 |
| 等官方"最终准入钩子"落地 | 官方背书 | **该钩子尚不存在**，是两张 open issue | 不能对着空气编码 |

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
