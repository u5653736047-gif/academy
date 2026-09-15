# ADR-0002：以 SDK 方式内嵌 pi

- 状态：**已接受**（其中"子代理走官方扩展机制（独立进程）"一句**已被 [ADR-0006](./0006-subagent-process-model.md) 挑战**）
- 日期：2026-09-14
- 决策人：项目组
- **适用阶段：阶段一**
- 相关：[ADR-0001](./0001-electron-desktop.md)、[ADR-0006](./0006-subagent-process-model.md)、[ADR-0007](./0007-permission-gate-hook-point.md)

## 背景

pi 提供三种使用形态：

1. **CLI**：`pi` 可执行文件，交互式 TUI
2. **SDK**：`createAgentSession()` 等工厂函数，在自己的 Node 进程里建 `AgentSession`
3. **RPC**：`pi --mode rpc`，宿主通过 stdin/stdout 的 JSON 协议驱动一个 pi 子进程

我们要用 Electron 主进程做 agent 宿主（ADR-0001），需要决定用哪种。

## 决策

**以 SDK 方式内嵌**：在 Electron 主进程里调 `createAgentSession()` 建会话，自己实现 `ExtensionUIContext` 并用 `session.bindExtensions()` 把 GUI 接进去。

## 备选方案

| 方案 | 优点 | 缺点 | 为什么没选 |
|---|---|---|---|
| **SDK 内嵌**（选中） | 同进程，事件、中止、授权都是直接函数调用；无 IPC 序列化开销；打包时不需要额外分发 pi 运行时 | 一个进程内的崩溃会影响全部会话；pi 是纯 ESM，主进程需按 ESM 配置 | — |
| RPC 子进程 | 隔离好，一个会话崩了不影响别的；RPC 协议现成（含 `extension_ui_request` 弹窗通道） | 需要把 pi 运行时一起打包分发；每个会话一个进程，内存开销大；多一层序列化 | 复杂度更高，收益在当前规模下不明显 |
| CLI + 终端嵌入式 | 复用官方 TUI | 与"图形界面 agent 产品"的需求不符 | 需求明确要 GUI |

## 后果

### 正面
- 会话、子代理、权限闸都在同一进程，**"共用同一道闸"是天然成立而不是靠协议保证**
- 可以直接给 `session.agent.beforeToolCall` 赋值（见 ADR-0007）
- 可以内联注入扩展（`DefaultResourceLoader({ extensionFactories: [...] })`），不必落盘

### 负面 / 代价
- **`ctx.hasUI` 默认为 `false`**：不显式调 `bindExtensions` 的话，扩展模式默认是 `"print"`，UI 上下文是 no-op，官方 `permission-gate.ts` 这类范例会直接走"无 UI 则拒绝"分支，表现为"agent 什么都不肯干"且不报错
- **闸加载失败会静默降级**：`loader.ts` 里单个扩展加载失败只是 push 进 `errors[]` 然后 `continue`，SDK 里没人展示这个错误 —— 必须自己断言
- 进程内没有 OS 级隔离，一个跑飞的子代理会拖累整个应用

### 需要后续跟进的事
- [ ] 启动自检：从 `extensionsResult.errors` 断言闸已加载，装不上就拒绝启动
- [ ] 实测 Electron 主进程的 ESM 加载（pi 是 `"type": "module"`）
- [ ] 是否用 `noExtensions: true` 收敛扩展面（见 [permission-gate.md](../permission-gate.md)）

## 什么情况下该重新审视

- 如果子代理跑飞导致主进程不稳定的问题频繁出现 → 考虑把 agent 宿主整体移到 Electron 的 utility process，或改用 RPC 模式（见 ADR-0006）
- 如果将来要支持"多个客户端连同一批会话"，RPC / `pi-server` 更合适

## 参考

- `packages/coding-agent/docs/sdk.md`
- `packages/coding-agent/src/core/sdk.ts`（`createAgentSession()` 返回 `{ session, extensionsResult, modelFallbackMessage }`）
- `packages/coding-agent/src/core/agent-session.ts:2237-2260`（`bindExtensions`）
