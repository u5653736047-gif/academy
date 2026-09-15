# 课程 Agent 助教 · 文档索引

> 版本：v0.3 ｜ 状态：设计阶段收尾 ｜ 日期：2026-09-15
> 本目录是项目的全部文档。**代码尚未开始编写，当前处于设计阶段。**

> ## ⭐ 先读这两句
>
> 1. **第一版分两个阶段**：阶段一 = 把 pi 包装成有 GUI 的桌面应用（内核改造 + 子代理 + 权限闸）；阶段二 = 检索工具 / 检索子代理 / 服务端知识库。
>    → 看 [02-scope-v1.md](./02-scope-v1.md)
> 2. **每条决策都已拍板**（7 条 ADR 全部"已接受"），其中 0003 / 0004 / 0005 属阶段二。
>    → 看 [design/adr/](./design/adr/README.md)

## 这个项目是什么

给编程课学生的本地 AI 助教：一个装在学生电脑上的 coding agent，能读学生的代码、跑学生的代码、把问题修好；解释问题时引用课程资料（教材、课件）作为依据。

底座是 [pi](https://pi.dev)（`@earendil-works/pi-coding-agent` / `pi-agent-core`），本地源码副本在 `D:\code\agent-design\pi`。

## 阶段与文档的关系

| 阶段 | 主题 | 主要看哪些文档 |
|---|---|---|
| **阶段一** | pi 内核改造 + 桌面壳（**无服务端**） | [02-scope-v1.md](./02-scope-v1.md) 第二节、[design/gui.md](./design/gui.md)、[design/permission-gate.md](./design/permission-gate.md)、[design/ipc-contract.md](./design/ipc-contract.md)、[test/permission-cases.md](./test/permission-cases.md)、[ops/build.md](./ops/build.md) |
| **阶段二** | 检索 / 知识库 / 服务端 | [01-requirements.md](./01-requirements.md) 第 3、5 节、[design/server-api.md](./design/server-api.md)、[design/data-model.md](./design/data-model.md) 第 3 节、[ops/deployment.md](./ops/deployment.md) |

**每份文档头部都标了 `适用阶段`。** 标"阶段二"的，阶段一不据此开工。

## 目录结构

```
docs/
├── README.md              ← 你在这里（文档索引）
├── 00-glossary.md         术语表
├── 01-requirements.md     需求文档（v1.0.1）
├── 02-scope-v1.md         范围与不做清单（变更控制基线）
├── research/              调研报告
│   └── README.md          调研索引与证据等级说明
├── design/                设计
│   ├── architecture.md    架构设计（v0.1）
│   ├── diagrams/          架构图（SVG / PNG）
│   ├── adr/               架构决策记录
│   ├── permission-gate.md 权限闸详细设计   ← 第一版复杂度所在
│   ├── gui.md             GUI 详细设计
│   ├── ipc-contract.md    接口契约：渲染进程 ↔ 主进程
│   ├── server-api.md      接口契约：端 ↔ 云三条通道
│   └── data-model.md      数据模型
├── test/                  质量与验证
│   ├── test-plan.md       测试计划
│   ├── permission-cases.md 权限规则用例表（先于代码编写）
│   └── traceability.md    需求追溯矩阵
└── ops/                   交付与运维
    ├── build.md           构建与打包
    └── deployment.md      服务端部署
```

## 文档清单

| 编号 | 文档 | 阶段 | 状态 | 说明 |
|---|---|---|---|---|
| — | [00-glossary.md](./00-glossary.md) | 两阶段 | 草稿 | 术语表。**先读这个**，避免"实例/会话/子代理"混用 |
| 01 | [01-requirements.md](./01-requirements.md) | 两阶段 | 已评审（待老师） | 需求文档 v1.0.2，替代 v0.1/v0.2。正文内标了阶段 |
| **02** | [**02-scope-v1.md**](./02-scope-v1.md) | **两阶段** | **v0.2 已定** | ⭐ **第一版范围与阶段划分。变更控制基线，从这里开始** |
| — | [research/](./research/README.md) | — | **已完成** | pi 权限方案调研（5 个渠道）。合并结论见 [permission-gate-survey.md](./research/permission-gate-survey.md) |
| — | [design/decisions-pending.md](./design/decisions-pending.md) | — | **已定案** | 决策过程存档（4 条已全部拍板）。新读者不必读 |
| — | [design/architecture.md](./design/architecture.md) | 两阶段 | **v0.2 已改写** | 架构概要。已按 ADR-0006/0007 与阶段划分改写 |
| — | [design/adr/](./design/adr/README.md) | 两阶段 | **7 条全部已接受** | 架构决策记录。索引表标了阶段归属 |
| — | [design/permission-gate.md](./design/permission-gate.md) | **一** | 草稿 | 权限闸详细设计。已有大量源码级结论，规则格式待定 |
| — | [design/gui.md](./design/gui.md) | **一**（激活 = 二） | 骨架 | GUI 四视图。**技术栈未定——阶段一第一个岔路口** |
| — | [design/ipc-contract.md](./design/ipc-contract.md) | **一**（检索/反馈 = 二） | 骨架 | 事件下行 / 指令上行 / 审批请求协议 |
| — | [design/server-api.md](./design/server-api.md) | **二** | 骨架 | 检索 / 激活 / 上报三条通道 |
| — | [design/data-model.md](./design/data-model.md) | 两阶段 | 骨架 | 第 1、2 节 = 阶段一；第 3 节 = 阶段二 |
| — | [test/test-plan.md](./test/test-plan.md) | 两阶段 | 骨架 | 分层测试策略。L1–L3 = 一；L4 = 二 |
| — | [test/permission-cases.md](./test/permission-cases.md) | **一** | 骨架 + 种子用例 | 规则用例表，写代码前先定 |
| — | [test/traceability.md](./test/traceability.md) | 两阶段 | 骨架 | 需求 ↔ 设计 ↔ 测试，已加阶段列 |
| — | [ops/build.md](./ops/build.md) | 两阶段 | 骨架 | 阶段一跑开发模式；打包分发归阶段二 |
| — | [ops/deployment.md](./ops/deployment.md) | **二** | 骨架 | 服务端部署与运维 |

## 状态约定

| 状态 | 含义 |
|---|---|
| 骨架 | 只有章节结构，内容待填 |
| 草稿 | 正在写，随时会变，不要据此开工 |
| 已评审 | 过了一遍，可以被下游文档引用 |
| 基线冻结 | 变更需要走 ADR + 变更记录，不能直接改正文 |

## 约定

1. **文档编号**：一级文档用两位数字前缀（`00-`、`01-`…）决定阅读顺序；设计细节不编号，按目录归类。
2. **版本与日期**：每份文档头部标注版本、状态、日期。
3. **上游/下游**：每份文档头部写清它的输入和输出，避免"这份文档该写多细"的争论。
4. **适用阶段**：每份文档头部标注它服务于阶段一还是阶段二。**标"阶段二"的，阶段一不据此开工。**
5. **需求 ID**：需求文档目前**没有给需求项编号**，导致追溯矩阵挂不住（见 [test/traceability.md](./test/traceability.md)）。建议在需求评审时补上 `REQ-x.y` 编号。
6. **未决问题**：设计文档里用 `【待决】` 标记未拍板项，用 `【已定】` 标记已确认项，并在章节末尾汇总。
7. **来源标注**：涉及判断的地方标 `【你的决定】/【调研结论】/【我的推断】`，让你只需重点看"我的推断"那部分。

## 当前阶段

设计阶段收尾，**决策已全部落定**（ADR-0001 ~ 0007），**第一版已切分为两个阶段**，准备冻结基线。

已完成：

- [x] 调研（5 个渠道 + 4 个子代理渠道，全部返回）
- [x] 权限闸与子代理方案拍板（[ADR-0006](./design/adr/0006-subagent-process-model.md) / [ADR-0007](./design/adr/0007-permission-gate-hook-point.md)）
- [x] **第一版范围定案**：切分为阶段一 / 阶段二（[02-scope-v1.md](./02-scope-v1.md)）
- [x] 按决策与阶段划分改写 [architecture.md](./design/architecture.md) 为 v0.2
- [x] 全库文档标注「适用阶段」
- [x] git 仓库与提交规范

接下来（**阶段一开工前的关卡**）：

- [ ] **技术验证（spike）**：第三方扩展能否在 Electron + pi SDK 里加载（两个包的 `.ts` 运行时出口是第一个要验的）——**这一条不通，整条复用路线要重来**
- [ ] **实测需求 5**：`pi-subagents` 前台进程内子会话的 `ask` 到底能不能转发到我们的 GUI
- [ ] **定 GUI 技术栈**（[gui.md](./design/gui.md) 第 2 节）——阶段一交付的就是界面
- [ ] 确认**阶段一验收标准 A1–A5**（[02-scope-v1.md](./02-scope-v1.md) 第二节末）
- [ ] 把设计文档里 `【待决】` 的地方逐条落实（含权限规则格式、`pi -p` 绕过规则）
- [ ] 需求文档补 `REQ-x.y` 编号，填完追溯矩阵
- [ ] 设计评审 → 冻结 v1.0 基线
