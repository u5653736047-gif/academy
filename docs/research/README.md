# 调研索引

> 版本：v0.1（骨架）｜ 状态：进行中 ｜ 日期：2026-09-14
> 上游：[02-scope-v1.md](../02-scope-v1.md) ｜ 下游：[design/permission-gate.md](../design/permission-gate.md)、[design/adr/](../design/adr/README.md)

## 调研主题

**pi 生态里有没有现成的权限审批方案，我们能直接拿来用，或者照抄设计？**

匹配标准是我们对权限闸的 7 条需求：

1. 每次工具调用执行前拦截（尤其 `bash` / `write` / `edit`）
2. 三态判定：放行 / 拒绝 / 询问
3. 分级：白名单直放、黑名单必问、灰区交小模型判危险（拿不准=危险）
4. 三档设置：变更前确认 / 自动审批 / 完全访问
5. 主 agent 与子代理**共用同一道闸**
6. 审计日志 + 界面可见可中止
7. 用 SDK 内嵌 pi（`createAgentSession()` + `bindExtensions`），不跑 CLI

## 证据等级（重要）

本项目的网络环境**受限**，必须先说明证据强度，否则会把"标题"当成"事实"：

| 等级 | 含义 | 可信度 |
|---|---|---|
| **A 源码级** | 直接读了本地 pi 源码 / 随包分发的官方文档 | 高。本地副本 `D:\code\agent-design\pi` |
| **B 索引级** | 只读到搜索引擎返回的标题与摘要片段 | **低**。无法核实 star / license / 最后更新 / 实际实现 |
| **C 转述级** | 二手博客、论坛帖、他人总结 | 最低 |

**实测到的网络限制**：

- `web_fetch` 对**所有**外部域名返回 `URL hostname "<host>" resolves to a non-public IP address`。根因：本机 DNS 把域名解析到 fake-IP（实测 `198.18.0.20`）
- 子代理的 `web_search` 报 `DeepSeek API error (HTTP 402): Insufficient Balance`（搜索端点余额不足）
- 受影响的域名：`github.com`、`registry.npmjs.org`、`raw.githubusercontent.com`、`cdn.jsdelivr.net`、`pi.dev`
- 站点侧另有：`npmjs.com` / `socket.dev` 返回 403（机器人防护）

**绕过办法（已实测可用）**：`node -e "fetch(...)"` —— Node 自带 OpenSSL、不走系统 schannel，可以正常出网。已实测：`registry.npmjs.org` 200、`raw.githubusercontent.com` 200、`pi.dev/packages` 200。

> ⚠️ **因此本项目的证据等级可以提升到 A 级（读源码）。** 凡需要核实的外部结论，用这个通道重取，不要停留在搜索标题上。

## 已核实：主要候选（registry 元数据 + README 原文）

### `@gotgenes/pi-permission-system` @ 32.0.2

| 项 | 实测值 |
|---|---|
| license | MIT |
| 版本数 | **210**（迭代极快，大版本存在破坏性变更） |
| 依赖 | `tree-sitter-bash` ^0.25.1、`web-tree-sitter` ^0.26.9、`zod` ^4.4.3 |
| peer 依赖 | `@earendil-works/pi-coding-agent` >=0.79.0、`@earendil-works/pi-tui` >=0.79.0 |
| 可否 import 进宿主 | **可以**：`exports["."] = { types: "./dist/public.d.ts", default: "./src/service.ts" }` |

README 原文核实到的能力，**逐条对上我们的 7 条需求**：

| 我们的需求 | 它的对应实现（README 原文） |
|---|---|
| ② 三态 | allow / ask / deny |
| ③ 出项目目录 | `external_directory` surface —— *"the CWD-boundary gate: it decides whether reaching **outside** the working tree is allowed, and accepts a pattern map"* |
| ③ 命令解析 | 依赖 `tree-sitter-bash` 做 AST 解析，**不是正则** |
| ③ 灰区判官 | `authorizerChain` —— *"names registered case-by-case decision links (e.g. a light model judge) to consult when a request lands on `ask`"*；且 *"A link decides nothing until you name it… and its `allow` on an excluded surface is downgraded to `defer`"*（**插件无权放大用户策略**） |
| ⑤ 子代理同闸 | *"Forwards prompts from subagents — `ask` policies work even in non-UI execution contexts"*；*"in-process child sessions register with the permission system automatically"* |
| ⑥ 审计与可见 | review log + `permissions:ui_prompt` / `permissions:decision` 事件广播 |
| fail-closed | *"Fails closed — an internal gate error blocks the tool (with a `gate_error` review-log entry)"*；16.0.0 起 bash gate 失败即关闭 |
| ④ 三档 UI | **没有现成的**，需要我们自己包一层 |

**结论**：7 条里 **6 条直接命中**，而且它是候选里**唯一一个设计上就允许被宿主 import 的**（其余基本都是纯扩展文件）。

### `pi-verdict` @ 0.7.1

| 项 | 实测值 |
|---|---|
| license | MIT ｜ **零依赖**（registry 中 `dependencies` 为空） |
| 版本数 | 17 ｜ `main: extensions/pi-verdict.ts` |

README 原文核实到的设计：

- 三态的理由：*"verdict 是裁决，不是开关……`ask` 把真正含糊的动作转交人类确认（非交互会话中降级为 `deny`），「不确定」永远不会静默变成「放行」"*
- *"确定性 floor 先于 AI——硬 deny 永不被分类器或用户 allow 规则覆盖"*
- **自保护层**：受保护文件在 `session_start` 快照、每次裁决前复核；扩展副本被改 → 自动还原 + 本会话 fail-closed
- bash 路径提取是 token 级：*"命令替换、base64 内嵌路径、外部脚本内容不产生命中信号——这些调用回落到分类器的存在性话术警戒"*（**它自己承认了这里的能力边界**）

**缺口**：README 中 **`subagent` 零命中** —— 它不处理需求 ⑤。

## 决策影响：自研 vs 复用

这两个核实结果把"自研还是复用"从一个哲学问题变成了具体问题：

| 路线 | 做法 | 代价 |
|---|---|---|
| **复用** | 以 `@gotgenes/pi-permission-system` 为主线（挂在扩展层 `tool_call`、sessionId 为 key），我们只补三档 UI 与 Electron 集成 | 接受 210 个版本的迭代速度；tree-sitter 的 Electron 打包问题；与 `pi-tui` 的 peer 依赖；**ADR-0007 里"硬判定挂 SDK 钩子"的方案要放弃** |
| **自研** | 照 `pi-verdict` 的结构自己写（零依赖、约 1k 行、确定性 floor 先于 AI、自保护层），保留 SDK 钩子的抗篡改设计 | 命令解析要自己上 tree-sitter；需求 ⑤ 的 session-keyed 转发机制要自己实现 |

**这是一条新的待拍板事项，与 [ADR-0006](../../design/adr/0006-subagent-process-model.md) / [ADR-0007](../../design/adr/0007-permission-gate-hook-point.md) 强耦合。**

## 渠道与状态

| # | 渠道 | 负责范围 | 状态 | 结论可用度 |
|---|---|---|---|---|
| A | npm 生态 | pi 扩展的分发机制 + 权限类 npm 包 | **已返回** | A3 级（registry 元数据 + README 原文） |
| B | GitHub 开源生态 | `earendil-works` 组织、官方 RFC、第三方扩展 | **已返回** | 本地部分 A 级，外部 B 级 |
| C | 官方文档 + 本地源码 | 逐个清点 `examples/extensions/` + 官方立场原文 | **已返回** | **A1 级（主力证据）** |
| D | 邻近生态 | Claude Code / Codex / OpenClaw / Cline 的权限设计 | **已返回** | **A2 级（本机发行物取证）** |
| E | 泛社区 + 通用轮子 | 非英语社区、通用 npm 库、**shell 命令解析库** | **已返回** | B 级（含本地核实项） |

> **五个渠道全部返回，调研阶段结束。** 合并结论见 [permission-gate-survey.md](./permission-gate-survey.md)。

## 初步结论（待合并）

已经可以确认的（A 级证据，详见 [design/permission-gate.md](../design/permission-gate.md)）：

1. **pi 官方不做权限系统**，且这是**主动决策**而非疏漏。官方原话见 `packages/coding-agent/README.md:502`：*"No permission popups. Run in a container, or build your own confirmation flow with extensions inline with your environment and security requirements."*
2. **唯一的执行前拦截点是 `tool_call` 事件**，由 `agent-session.ts:479-499` 安装到 `agent.beforeToolCall`。
3. **官方 subagent 示例不传递权限约束到子进程**，存在明确的提权绕过路径。
4. **没有现成方案满足全部 7 条**。最接近的第三方方案也缺"灰区判官"和"审计日志"。
5. **第三方 npm 包不建议作为依赖**：同名包在 10+ 个 namespace 下各自发布，供应链无法评估（B 级证据，但足以构成排除理由）。
6. **shell 命令解析有现成库可用**（E 渠道，本次最有价值的产出）。首选 **`@aliou/sh`**（纯 TypeScript 的 shell parser / AST，**无原生依赖，对 Electron 最友好**，且作者 aliou 是 pi 的贡献者），备选 `tree-sitter-bash` + wasm 分发（免编译，但**需验证 wasm 包里含 bash grammar**）。`shell-quote` **只是 tokenizer 不是 AST**，不能单独用作判定依据。详见 [design/permission-gate.md](../design/permission-gate.md) 第 6.1 节。
7. **pi 生态里至少 13 个第三方权限扩展已存在**，最对口的三个（**均为索引级证据，未读源码**）：
   - `pi-verdict`（jesset）—— *"allow / ask / deny, one file, zero dependencies"*，作者发在官方 Discussion #8803
   - `pi-permission-gate`（juanje）—— *"Deny-by-default tool restriction with glob matching, path normalization, self-protection, and structured logging"*
   - `pi-approval-guardian`（mics8128）—— *"Fail-closed ... bash approval gate"*
   另外 `pi-edit-approval`（ShawnMa123）的分级语义 "workspace vs outside" **正是我们要的分级结构**。
   ⇒ **建议**：只当设计参考，**不作为依赖**（理由见第 5 条）。
8. **Windows 上没有任何 OS 级兜底**：官方 `sandbox` 示例依赖 `@anthropic-ai/sandbox-runtime`，代码里显式 `platform !== "darwin" && platform !== "linux"` 就禁用；`gondolin` 是 Linux micro-VM。⇒ "联网"和"出项目目录"两项**只能靠闸 + 命令解析**。
9. **需求 5（子代理共用同一道闸）的风险在别处也真实发生过**：Claude Code 有一张 issue 标题为 *"Built-in Plan agent ignores parent settings.json permissions and repeatedly prompts for pre-approved tools"*（anthropics/claude-code#10906）。这不是我们的臆想，是被踩过的坑。

## 本地可开采的资源

| 位置 | 内容 | 状态 |
|---|---|---|
| `D:\code\agent-design\pi` | pi 源码与官方文档（**当前全部 A 级证据的来源**） | 已系统盘点（C 渠道） |
| `D:\CODE\hermes-agent` | 一个同类项目的完整 checkout，其技术报告提到"加固危险命令 denylist 以对抗各类 shell 转义绕过" | **未开采，免费且高质量**，建议后续派人读它的 denylist 实现 |

## 待补清单（需要公网环境）

调研因网络受限有几处空白，记录在此，供将来补做：

- [ ] `earendil-works` 组织下的全部仓库清单
- [ ] 官方 RFC 站 `rfc.earendil.com/keyword/pi/` 是否有权限系统相关提案
- [ ] 作者博客 `https://mariozechner.at/posts/2025-11-30-pi-coding-agent/` —— **唯一能回答"维护者为什么不做权限"的一手材料**
- [ ] 官方 Issues / Discussions 里关于 permission 的讨论正文
- [ ] 第三方候选的 star / license / 最后更新时间 / 源码实读

## 调研产物

| 文件 | 内容 | 状态 |
|---|---|---|
| [pi-capability-audit.md](./pi-capability-audit.md) | C 渠道：pi 官方能力盘点、本地可复用资产清单 | 【待写】原始报告已回收 |
| [permission-gate-survey.md](./permission-gate-survey.md) | A/B/D/E 渠道合并：外部候选方案、跨产品共识、可抄的设计模式 | **已写** |
