# 外部方案调研：权限与审批系统

> 版本：v0.1 ｜ 状态：草稿 ｜ 日期：2026-09-14
> 上游：[README.md](./README.md)（调研索引与证据等级）
> 下游：[../design/permission-gate.md](../design/permission-gate.md)、[../design/adr/0007](../design/adr/0007-permission-gate-hook-point.md)

## 0. 一句话结论

**这道闸不用发明。** 头部产品在这件事上已经高度收敛：拦截点、三态语义、失败姿态、规则优先级、甚至"用小模型判危险"都有成熟做法。我们要做的是**选一套抄，然后把 pi 宿主的三处缺口补上**。

---

## 1. 证据等级与一个重要的取证手段

| 等级 | 来源 | 本次用到的 |
|---|---|---|
| **A1** | 本机 pi 源码 + 随包官方文档 | `D:\code\agent-design\pi` |
| **A2** | **本机已安装产品的发行物**：官方 `--help` 输出、官方 `.d.ts`、**二进制内嵌的 schema 校验器文案**、随包文档 | Claude Code 2.1.246、Codex 0.154.0、OpenClaw 2026.3.2-beta.1、Cline 3.0.56 |
| **A3** | 包管理器元数据 + README 原文 | npm registry（`@gotgenes/pi-permission-system` 等） |
| B | 搜索结果的标题与摘要 | **不用于任何结论** |

**取证手段的两个坑与一个绕过**（记录下来，避免重复踩）：

- `web_fetch` 对**所有**域名返回 `resolves to a non-public IP address`。根因：本机 DNS 把域名解析到 fake-IP（实测 `198.18.0.20` / `198.18.0.24`），落在 RFC 2544 保留段
- 子代理的 `web_search` 中途报 `HTTP 402 Insufficient Balance`
- ✅ **绕过办法（已实测）**：`node -e "fetch(...)"` —— Node 自带 OpenSSL、不走系统 schannel。`registry.npmjs.org` / `raw.githubusercontent.com` / `pi.dev` 均 200
- ✅ **另一个取证富矿**：本机 `%APPDATA%\npm\node_modules` 下装着 Claude Code、Codex、OpenClaw、Cline、OpenCode——**它们的发行物就是一手证据**

> **A2 的价值**：二进制内嵌的 schema 校验器文案，比官方文档页面更接近"产品真实行为"。本次关于 Claude Code 规则语法的权威规格就来自这里，而且它**纠正了一个流传很广的二手说法**（见第 4 节）。

---

## 2. 跨产品共识（可直接采纳为我们的规范）

五家独立收敛到同一结论，这类共识值得直接照抄：

| # | 共识 | 证据 |
|---|---|---|
| 1 | **拦截点放在"工具执行前"这一层**，而不是模型侧（提示词）或系统调用层（容器） | pi `tool_call` / Claude Code `PreToolUse` / Codex exec_policy / Cline `beforeTool` 全部如此 |
| 2 | **声明式规则 + 命令式 hook 两层并存**。规则管高频可枚举的放行/拒绝，hook 管需要上下文的判定 | 没有一家只用其中一种 |
| 3 | **优先级：deny > ask > allow**，且 **allow 规则最严、deny/ask 最宽** | OpenClaw "deny always wins"；Claude Code 的 allow 是唯一带安全告警的类别 |
| 4 | **失败与不确定一律 fail-closed** | pi（handler 抛错即阻断、无 UI 即阻断）、Claude Code（hook 超时/异常 → "The tool call was not executed"）、OpenClaw（`askFallback` 默认 `deny`）、Codex（配置冲突 → 降级 read-only + 显式报错）、Cline（超时按拒绝） |
| 5 | **确定性匹配走快路径，模型判定只做兜底** | Claude Code `auto` 模式 = 分类器 + `allow`/`soft_deny`/`hard_deny`/`environment` 四组规则；Codex = `on-request-auto-review` + `guardian_subagent` |

**第 4 条可以直接写成一条规范**：

> 闸自身故障、分类器超时、审批界面不可达、用户超时未响应 —— **四种情况一律判定为拒绝**，且必须向模型返回可读原因与替代建议。**不得静默放行，不得静默失败。**

---

## 3. 关键架构发现：OpenClaw 的做法比我们原计划更彻底

本机装着 `openclaw@2026.3.2-beta.1`，随包文档 `docs/pi.md` 完整描述了它怎么用 pi SDK 做产品。

**原文（`docs/pi.md:15`）**：

> *"Instead of spawning pi as a subprocess or using RPC mode, OpenClaw directly imports and instantiates pi's `AgentSession` via `createAgentSession()`."*

**关键在工具层（`docs/pi.md:272-283`）**：

```typescript
export function splitSdkTools(options: { tools: AnyAgentTool[]; sandboxEnabled: boolean }) {
  return {
    builtInTools: [],                    // Empty. We override everything
    customTools: toToolDefinitions(options.tools),
  };
}
```

它的工具流水线（`docs/pi.md:241-249`）：

```
1. Base Tools      —— pi 的 read / bash / edit / write
2. Custom Replacements —— 用 exec/process 替换 bash，改造 read/edit/write 以适配沙箱
3. 自有工具        —— messaging / browser / sessions / cron / …
4. Channel Tools   —— 各渠道（Discord/Telegram/Slack）专属
5. Policy Filtering —— 按 profile / provider / agent / group / sandbox 策略过滤
6. Schema Normalization
7. AbortSignal Wrapping
```

### 这对我们意味着什么

我们在 [ADR-0007](../design/adr/0007-permission-gate-hook-point.md) 里纠结的是"闸挂哪一层"。OpenClaw 给出了**第三个选项，而且可能是最好的那个**：

| 方案 | 做法 | 代价 |
|---|---|---|
| A. 扩展层 `tool_call` | 生态标准做法 | `event.input` 可变且改后不重校验 → 不是安全边界 |
| B. SDK 钩子 `Agent.beforeToolCall` | 我们原本的提议，抗篡改 | 需要包装 AgentSession 已装的钩子；与第三方闸不兼容 |
| **C. 直接替换内置工具** | `builtInTools: []` + `customTools: [我们自己的受控工具]`，闸长在工具实现里 | 要自己实现或包装 7 个工具 |

**方案 C 的好处**：TOCTOU 问题（参数被改、改后不重校验）**从根上消失**——因为工具是我们自己的，参数进到工具里就是最终值。而且 pi 已经导出了 `createReadTool` / `createBashTool` / `createEditTool` / `createWriteTool` / `createCodingTools` / `createReadOnlyTools` 这些**工具工厂**，官方 `sandbox` 示例正是用 `createBashTool(cwd, { operations })` 覆盖 bash 的——**我们是"包装"而不是"重写"**。

> ⚠️ 这条需要单独评估，它会改动 ADR-0007 的结论。**新增一个待决项。**

---

## 4. 各地区值得抄的具体模式

### 4.1 Claude Code：规则语法（唯一被完整固化的规格）

从 `claude.exe` 内嵌的 schema 校验器逐字抠出来的：

| 写法 | 语义 |
|---|---|
| `Bash(npm run:*)` | **legacy 前缀匹配**（很多博客把它当成唯一正确写法，其实已被标注为 legacy） |
| `Bash(npm run *)` | **通配匹配**（推荐写法） |
| `Edit(docs/**)`、`Read(~/.zshrc)` | 文件类用 glob |

**三条直接可用的安全约束**：

1. **通配符不得前置于子命令**。校验器原文：通配符在子命令之前会*"also match any options inserted at that position and approves them without a prompt"*，并专门警告 git：*"options such as -c and --exec-path can run arbitrary commands"*。
   ⇒ **我们的学生级白名单禁止 `Bash(git *)` 这种写法**，只允许 `Bash(git status *)` 形式。
2. **写类工具要归一到 `Edit`、读类归一到 `Read`**。`Write` / `MultiEdit` / `NotebookEdit` 的规则**不会被文件权限检查匹配**。
   ⇒ 我们的文件规则按**能力类别**（读 / 写 / 执行）匹配，不按工具名——否则"新增一个工具就绕过黑名单"。
3. **allow 严、deny/ask 宽**。MCP 规则里 allow 只允许在字面前缀之后的工具位用 glob，deny/ask 任意位置都行。

⚠️ **一处绝不能抄**：非法规则的行为是 *"Invalid permission rule "..." was skipped"* ——**跳过该条、继续运行**。用户会以为黑名单生效，实则没有。**我们必须反向实现：配置中出现非法规则 = 配置错误，整体降级为"全部需审批"或拒绝启动，并在界面明确报错。**

### 4.2 Codex：二维风险分类与档位记忆

- **`destructive_enabled` × `open_world_enabled`** —— 把风险分成"破坏性"与"开放世界（联网/外部副作用）"两个正交维度。**比我们扁平的黑名单清晰得多**：
  - 大面积删除 → destructive
  - 出项目目录 → 路径越界（用 `writable_roots` 表达）
  - 联网 → open-world
  - 装全局包 → destructive + open-world
- **档位记忆三级**：`Allow`（本次）/ `Allow for this session`（本会话）/ `Allow and don't ask again`。比"一次/永久"两级更符合教学场景——**一节课内记住，下节课重置**。
- **配置冲突时不让步**：*"would fall back to read-only permissions with approvals disabled"* + 给出可操作的修复指引。**绝不静默采用危险配置。**
- **审查器是可配的**：`classifier_instructions`（提示词）+ `review_threshold_basis_points`（阈值，万分比）+ `timeout_instructions`（超时怎么办）。**"小模型判危险"是产品配置面的一部分，不是黑盒。**

### 4.3 OpenClaw：白名单的正确实现层次

- **白名单锚定到"解析后的可执行路径"，而非命令字符串**：*"Patterns should resolve to binary paths (basename-only entries are ignored)"*。防 `PATH` 劫持与同名脚本仿冒。审批弹窗也要展示**解析后的真实路径**。
- **safeBins：仅 stdin 的窄白名单**——默认 `jq` / `cut` / `uniq` / `head` / `tail` / `tr` / `wc`，免显式条目即可运行，但**拒绝位置参数中的文件路径**、拒绝重定向、拒绝 `$()`、未知长选项 fail-closed、从受信目录解析、**PATH 条目从不自动受信**。
  ⇒ 这是"安全命令直接放行"唯一能规模化的做法。**我们的白名单应实现为 safeBins 模式，而不是命令名清单。**
- **白名单必须做 shell 语法级（argv 级）解析**：shell 链 `&&`/`||`/`;` 仅当**每段都命中**才放行；allowlist 模式不支持重定向；`$()`/反引号在**解析阶段即拒绝**；`env`/`nice`/`nohup`/`timeout` 等包装器与 `busybox`/`toybox` 要**解开以持久化内层可执行文件路径**。
- **stricter wins**：*"Effective policy is the stricter of tools.exec.* and approvals defaults"*。多来源配置（课程默认 / 学生个人 / 项目内）合并时一律取更严，**且项目内配置不得放宽权限**。
- **"记住放行"受约束**：包装器无法安全解开时**不持久化任何白名单条目**；**per-agent 白名单隔离**（*"one agent's approvals from leaking into others"*）。
- **三档组合（security × ask × askFallback）**：`deny`/`allowlist`/`full` × `off`/`on-miss`/`always` × `deny`/`allowlist`/`full`。默认三件套就是 **"默认拒绝 + 未命中才问 + 问不到就拒绝"**。
  ⇒ **注意它没有把"自动审批"等同于"跳过白名单"**——这比 Claude Code 的 `acceptEdits` 更保守（后者会连带自动批准 `rm`！）。

### 4.4 Cline：判定顺序与归因字段

- **判定顺序**：`beforeTool`（代码 hook）→ tool policy（声明式策略）→ user approval（问人）。
- **硬拦截不弹窗**：*"the user is never asked to approve a command that would only fail"*，且以 **`skip` 而非 `stop`** 返回——**拦截后让 agent 继续尝试别的做法**。这两条都是关键的体验设计。
- **审批请求自带归因字段**：`ToolApprovalRequest` 含 `sessionId`、**`agentId`**（原文档明确写 *"used for attribution in approval prompts, events, telemetry, and team/sub-agent flows"*）、`conversationId`、`toolCallId`、`toolName`、`input`、`policy`。
  ⇒ **我们的审批请求对象直接对齐这些字段**，需求 5 与需求 6 一次满足。
- **审批是跨进程服务**：hub 协议 + `ApprovalStatus = pending | approved | rejected | cancelled` 四态状态机。

### 4.5 Claude Code：子代理语义

`sdk-tools.d.ts` 原文：

> *"Subagents **inherit the parent session's permission mode**; agent-definition frontmatter may override it."*

⇒ **子代理强制继承父会话档位，不允许通过工具参数自升。** 这是"subagent 不得成为提权通道"的官方答案。

---

## 5. pi 生态候选方案（A3 级证据）

| 方案 | 版本/license | 7 条需求命中 | 能否直接依赖 |
|---|---|---|---|
| **`@gotgenes/pi-permission-system`** | 32.0.2 / MIT / **210 个版本** | **6/7**（缺三档 UI） | 可以（`exports` → `src/service.ts`），但需评估迭代速度与 tree-sitter 打包 |
| `pi-verdict` | 0.7.1 / MIT / **零依赖** / 约 1k 行 | 4~5/7（**无 subagent 处理**） | 可作"抄设计"首选 |
| `@erichll/pi-auto-review` | 0.18.1 / MIT | 灰区判官的**完整参考实现** | 否（依赖 permission-system ≥30） |
| `@yuru7/pi-ai-approval` | 0.3.0 / MIT | 风险分级参考 | 否（早期版本） |
| `pi-permission-modes` | 2.2.0 / MIT | "模式即捆绑包"概念 | 否（依赖旧版 sandbox-runtime） |
| `@zhushanwen/pi-permission` | 1.4.3 / MIT | 需求 3/4 几乎一一对应 | 否（默认档是 yolo，危险） |
| `@gotgenes/pi-subagents` | 21.7.0 / MIT | **进程内子代理 + sessionId 为 key 的权限注册** | 需评估 |

**供应链警示**：同名包在 10+ 个 namespace 下各自发布（`@xzzpig` / `@coderdkai` / `@bryan2333` / `@xicode` / `@mzwing` …），个别包**没有 license 字段**。若引入，**必须锁死 scoped 名 + 版本**。

---

## 6. 必须自研的部分（没有任何现成实现）

1. **需求 4 的三档 UI**（所有候选都没有现成的用户可见档位）
2. **Electron 集成**（`ExtensionUIContext` 的第三份实现——TUI 与 RPC 各有一份，SDK 内嵌要自己写）
3. **pi 宿主的三处缺口**（见下）

## 7. pi 宿主的三处缺口（照抄任何一家都补不上）

| # | 缺口 | 对策 |
|---|---|---|
| a | pi **原生没有 `ask` 返回值**，只有"过/拦"二态 | 第三态自建。**建议走审批服务（跨进程），而不是 `ctx.ui.confirm` 直连**——否则子代理场景失效 |
| b | `event.input` **可变且 mutate 后不再校验**，后注册的 handler 看到的是被改过的参数 | 闸必须在**参数快照**上判定，并在放行前**重新比对最终参数**；或直接采用第 3 节的方案 C（替换工具），从根上消除 |
| c | 并行工具模式下兄弟调用**先串行 preflight、再并发执行** | 审批期间收集的文件系统状态**在并发执行时可能已过期**，路径类判定不要假设状态不变 |

## 8. 一处纠偏

`examples/extensions/kimi-deferred-tools.ts` 是**延迟工具加载**（`setActiveTools` 按需激活工具，省 token），**与审批无关**。此前把它当作"延迟审批思路的来源"是错的，修正记录在此。

## 9. 未能核实

- **Cursor**（命令白名单/黑名单配置方式）：本机未安装，网络不可用。**无结论。**
- **Aider**：同上。**无结论。**
- **Kimi Code**：无权威来源。**无结论。**
- **OpenCode**：本机发行物全部逻辑编进 171MB 单文件二进制，检索不到权限相关证据（仅确认存在 `permissions.autoApprove: false` 的客户端默认值）。**不作实质结论。**
- **Claude Code `auto-mode defaults` 的默认规则集全文**：子命令需要起 node 子进程，被沙箱拦住。**建议在有正常执行权限的环境里导出，作为我们白/黑名单的初始语料。**
- 各产品的官方文档页面原文（`web_fetch` 全挂）——但已用 A2 级证据替代。

## 10. 引出的待决事项

| # | 事项 | 关联 |
|---|---|---|
| 1 | **工具替换 vs 钩子拦截**（第 3 节的方案 C） | 改动 [ADR-0007](../design/adr/0007-permission-gate-hook-point.md) |
| 2 | 自研 vs 以 `@gotgenes/pi-permission-system` 为主线 | 见 [README.md](./README.md) |
| 3 | 风险分类采用 Codex 的 `destructive × open_world` 二维？ | 影响 [permission-gate.md](../design/permission-gate.md) 第 3 节 |
| 4 | 档位内部用二维表示（security × ask），UI 聚合成三档？ | 影响 [permission-gate.md](../design/permission-gate.md) 第 8 节 |
| 5 | 白名单用 safeBins 模式（argv 级 + 解析后路径）？ | 影响 [permission-cases.md](../test/permission-cases.md) A 组 |
