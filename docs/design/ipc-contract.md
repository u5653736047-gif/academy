# 接口契约：渲染进程 ↔ 主进程（IPC）

> 版本：v0.2（骨架）｜ 状态：骨架 ｜ 日期：2026-09-15
> **适用阶段：两阶段**——**阶段一先落"事件下行 / 指令上行 / 审批协议"三条链路的骨架**；检索与反馈相关事件属阶段二
> 上游：[architecture.md](./architecture.md) 第 3 节、[permission-gate.md](./permission-gate.md)、[gui.md](./gui.md)
> 下游：实现代码、[../test/test-plan.md](../test/test-plan.md)
>
> 📌 **阶段一是这份契约的主要落地期**：阶段一要交付桌面应用，主/渲染两进程的边界就是它的骨架。
> 阶段划分见 [../02-scope-v1.md](../02-scope-v1.md)。

## 1. 这份文档要锁死什么

渲染进程和主进程之间的**每一个消息**：谁发、什么时候发、字段是什么、出错怎么办。

**为什么要写细**：这是两个进程的边界，字段一变两边都要改，而且错了只在运行时暴露。契约先定，代码后写。

## 2. 三条链路

| 链路 | 方向 | 用途 |
|---|---|---|
| **事件下行** | 主 → 渲染 | 流式输出、工具执行、子代理过程、状态变化 |
| **指令上行** | 渲染 → 主 | 提问、steer、abort、切换档位、会话操作 |
| **审批请求/回执** | 双向 | 权限闸弹窗（主 → 渲染请求，渲染 → 主回执） |

## 3. 事件下行【待填字段】

| 事件 | 何时发 | 关键字段 | 来源 | 阶段 |
|---|---|---|---|---|
| `session.state` | 会话状态变化 | sessionId, status(空闲/运行中/待确认) | 事件总线 | 一 |
| `message.delta` | 流式输出增量 | sessionId, messageId, delta | `message_update` | 一 |
| `message.complete` | 消息完成 | sessionId, messageId, content | `message_end` | 一 |
| `tool.start` | 工具开始执行 | sessionId, actor, toolName, args | `tool_execution_start` | 一 |
| `tool.end` | 工具执行结束 | sessionId, actor, toolName, result, isError | `tool_execution_end` | 一 |
| `subagent.start` | 子代理启动 | parentSessionId, subagentId, actor, task | 子代理包的生命周期事件 | 一 |
| `subagent.update` | 子代理进度 | subagentId, 当前动作 | 同上 | 一 |
| `subagent.end` | 子代理结束 | subagentId, 摘要, usage, stopReason | 同上 | 一 |
| `citation` | 检索返回出处 | messageId, 文件名, 页码, 片段 | 检索工具 | **二** |
| 【待补】 | | | | |

**字段级定义**：见下表模板

```
<事件名>
  字段      类型      必填   说明
  --------  --------  -----  ----------------------------
```

【待填】

## 4. 指令上行【待填字段】

| 指令 | 参数 | 返回 | 说明 | 阶段 |
|---|---|---|---|---|
| `prompt` | sessionId, text | ack | 发一条用户消息 | 一 |
| `steer` | sessionId, text | ack | 运行中插入指令 | 一 |
| `abort` | sessionId, target?(main / subagentId) | ack | 中止 | 一 |
| `tier.set` | sessionId, tier | ack | 切换权限档位 | 一 |
| `session.new` / `session.switch` | | ack | 会话操作 | 一 |
| `feedback` | messageId, 内容 | ack | 一键反馈 | **二** |
| 【待补】 | | | | |

## 5. 审批请求 / 回执协议【已定草案】

**设计原则：审批是消息，不是函数调用。** 好处是进程模型可换（将来改成 RPC 子进程时，GUI 侧不用改）。

### 请求（主 → 渲染）

```jsonc
{
  "type": "approval.request",
  "id": "uuid",              // 回执必须带同一个 id
  "actor": "main",           // "main" | "subagent:<名字>"
  "action": {
    "tool": "bash",          // 工具名
    "args": { "command": "rm -rf build" },   // 原始参数
    "cwd": "D:\\work\\hw3"
  },
  "risk": {
    "level": "high",         // low | medium | high
    "reason": "命中黑名单：大面积删除",
    "source": "denylist"     // denylist | gray-zone-judge | tier-policy
  },
  "options": ["allow", "deny"],   // 未来可能加 "allow-session"
  "timeoutMs": 60000,
  "explain": "这条命令会删除 build 目录下的全部文件。"   // 给学生看的人话
}
```

### 回执（渲染 → 主）

```jsonc
{
  "type": "approval.response",
  "id": "uuid",
  "decision": "allow",       // allow | deny
  "remember": "once"         // once | session
}
```

### 约定

| 约定 | 说明 |
|---|---|
| 超时 | **按拒绝处理**（沿用 pi RPC 模式的语义：confirm 超时 → `false`） |
| GUI 无响应 / 崩溃 | 按拒绝处理，**绝不挂起或放行** |
| 闸自身异常 | 与"用户拒绝"在协议上要能区分，呈现给学生的文案不同 |
| 并发 | 多个请求排队；【待决】排队策略与合并展示 |

## 6. 版本与兼容【待决】

- 协议版本号怎么带
- 主进程与渲染进程不同步升级时怎么办（Electron 里通常一起打包，风险低）

## 7. 待决问题

| # | 问题 |
|---|---|
| Q1 | 事件字段的全部定义 |
| Q2 | 事件是否需要节流 / 批量（流式输出高频） |
| Q3 | 审批请求的排队与合并策略 |
| Q4 | 是否需要"本次会话内总是允许"（批准记忆，见 permission-gate.md Q4） |
| Q5 | 协议版本策略 |

## 8. 参考

- `packages/coding-agent/docs/rpc.md:1144-1334` —— pi 官方 RPC 的 `extension_ui_request` / `extension_ui_response` 协议，我们的审批协议可以照它的形状设计
- `packages/coding-agent/docs/json.md` —— pi 的 JSON 事件流格式（事件下行的字段可参照）
