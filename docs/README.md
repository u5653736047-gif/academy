# 课程 Agent 助教 · 文档索引

> 版本：v0.1（骨架）｜ 状态：草稿 ｜ 日期：2026-09-14
> 本目录是项目的全部文档。**代码尚未开始编写，当前处于设计阶段。**

## 这个项目是什么

给编程课学生的本地 AI 助教：一个装在学生电脑上的 coding agent，能读学生的代码、跑学生的代码、把问题修好；解释问题时引用课程资料（教材、课件）作为依据。

底座是 [pi](https://pi.dev)（`@earendil-works/pi-coding-agent` / `pi-agent-core`），本地源码副本在 `D:\code\agent-design\pi`。

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

| 编号 | 文档 | 状态 | 说明 |
|---|---|---|---|
| — | [00-glossary.md](./00-glossary.md) | 草稿 | 术语表。**先读这个**，避免"实例/会话/子代理"混用 |
| 01 | [01-requirements.md](./01-requirements.md) | 已评审（待老师） | 需求文档 v1.0.1，替代 v0.1/v0.2 |
| 02 | [02-scope-v1.md](./02-scope-v1.md) | 草稿 | 第一版范围与不做清单，变更控制基线 |
| — | [research/](./research/README.md) | 进行中 | pi 权限方案调研（5 个渠道） |
| — | [design/architecture.md](./design/architecture.md) | 草稿（**待修订**） | 架构概要。subagent 进程模型一节待 ADR-0006 拍板后重写 |
| — | [design/adr/](./design/adr/README.md) | 进行中 | 架构决策记录。0006 / 0007 待拍板 |
| — | [design/permission-gate.md](./design/permission-gate.md) | 草稿 | 权限闸详细设计。已有大量源码级结论，规则格式待定 |
| — | [design/gui.md](./design/gui.md) | 骨架 | GUI 四视图。**技术栈未定** |
| — | [design/ipc-contract.md](./design/ipc-contract.md) | 骨架 | 事件下行 / 指令上行 / 审批请求协议 |
| — | [design/server-api.md](./design/server-api.md) | 骨架 | 检索 / 激活 / 上报三条通道 |
| — | [design/data-model.md](./design/data-model.md) | 骨架 | 本地与服务端的实体与字段 |
| — | [test/test-plan.md](./test/test-plan.md) | 骨架 | 分层测试策略 |
| — | [test/permission-cases.md](./test/permission-cases.md) | 骨架 | 规则用例表，写代码前先定 |
| — | [test/traceability.md](./test/traceability.md) | 骨架 | 需求 ↔ 设计 ↔ 测试 |
| — | [ops/build.md](./ops/build.md) | 骨架 | Electron 打包与分发 |
| — | [ops/deployment.md](./ops/deployment.md) | 骨架 | 服务端部署与运维 |

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
4. **需求 ID**：需求文档目前**没有给需求项编号**，导致追溯矩阵挂不住（见 [test/traceability.md](./test/traceability.md)）。建议在需求评审时补上 `REQ-x.y` 编号。
5. **未决问题**：设计文档里用 `【待决】` 标记未拍板项，用 `【已定】` 标记已确认项，并在章节末尾汇总。

## 当前阶段

设计阶段。下一步动作：

- [ ] 权限闸与 subagent 进程模型拍板（ADR-0006 / 0007）
- [ ] 调研结论合并（等 5 个渠道全部返回）
- [ ] 权限规则格式定案
- [ ] 设计评审 → 冻结 v1.0 基线
