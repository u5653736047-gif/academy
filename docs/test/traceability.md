# 需求追溯矩阵

> 版本：v0.2（骨架）｜ 状态：草稿 ｜ 日期：2026-09-15
> **适用阶段：两阶段**（每条需求标了归属阶段）
> 上游：[../01-requirements.md](../01-requirements.md)、[../design/](../design/)、[../02-scope-v1.md](../02-scope-v1.md)
> 下游：[test-plan.md](./test-plan.md)、[permission-cases.md](./permission-cases.md)

> 📌 **阶段列怎么用**：`一` = 阶段一交付；`二` = 阶段二交付。
> **阶段一的追溯只关心标 `一` 的行**——「设计模块」为空且阶段为「一」的，就是阶段一开工前必须补的漏项。

## 这份表解决什么问题

**每条需求，都能回答三个问题**：

1. 它由哪个模块实现？（设计覆盖了吗）
2. 它由哪个用例验证？（测到了吗）
3. 如果改这条需求，要动哪里？（变更影响面）

没有这张表，"需求文档"和"代码"就是两份互不相干的文件。

## ⚠️ 前置问题：需求文档没有编号

需求文档 [01-requirements.md](../01-requirements.md) 的条目**没有 ID**，只有章节标题。追溯矩阵挂不住没有 ID 的东西。

**建议**：需求评审时给每条需求补 `REQ-x.y` 编号。下表暂按章节定位，**编号列为待补**。

## 一、功能需求（对应需求文档第 2 节）

| 需求 ID | 需求 | 设计模块 | 测试用例 | 阶段 | 状态 |
|---|---|---|---|---|---|
| REQ-2.1a | 工具：读写项目文件 / 执行命令 | [permission-gate.md](../design/permission-gate.md) | | **一** | 【待填】 |
| REQ-2.1b | 工具：检索课程资料 | [server-api.md](../design/server-api.md) | | 二 | 【待填】 |
| REQ-2.2a | 主 agent 接待学生、调度子代理 | [adr/0006](../design/adr/0006-subagent-process-model.md) | | **一** | 【待填】 |
| REQ-2.2b | 通用子代理基座（能力完整，长时复杂任务） | [adr/0006](../design/adr/0006-subagent-process-model.md)、[architecture.md](../design/architecture.md) 3.3 | | **一** | 【待填】 |
| REQ-2.2c | 资料检索子代理（上下文防火墙，只读） | [adr/0006](../design/adr/0006-subagent-process-model.md) | | 二 | 【待填】 |
| REQ-2.2d | **改文件/跑命令，主与子代理走同一道闸** | [permission-gate.md](../design/permission-gate.md) 第 9 节 | PC-E-06、PC-C-11 | **一** | 【待填】 |
| REQ-2.2e | 子代理动作可见、可中止 | [gui.md](../design/gui.md)、[ipc-contract.md](../design/ipc-contract.md) | | **一** | 【待填】 |
| REQ-2.3a | 权限三档，学生可切换 | [permission-gate.md](../design/permission-gate.md) 第 8 节 | PC-F-* | **一** | 【待填】 |
| REQ-2.3b | 自动审批档：黑白名单 + 灰区判官 | [permission-gate.md](../design/permission-gate.md) 第 3、7 节 | PC-A/B/D-* | **一** | 【待填】 |
| REQ-2.3c | 只读动作任何档位不拦 | [permission-gate.md](../design/permission-gate.md) | PC-A-01 | **一** | 【待填】 |
| REQ-2.4a | 安装：Electron 安装包，双击即用 | [adr/0001](../design/adr/0001-electron-desktop.md)、[../ops/build.md](../ops/build.md) | | 二 | 【待填】 |
| REQ-2.4b | 课程激活（邀请码 + 姓名学号） | [server-api.md](../design/server-api.md) 第 3 节 | | 二 | 【待填】 |
| REQ-2.4c | 模型配置：学生填自己的 API key | [data-model.md](../design/data-model.md) 2.3 | | **一** | 【待填】 |
| REQ-2.4d | 干活过程实时可见 | [gui.md](../design/gui.md) 3.1、[ipc-contract.md](../design/ipc-contract.md) | | **一** | 【待填】 |
| REQ-2.4e | **回答有出处**（只从检索片段引用） | [server-api.md](../design/server-api.md) 第 2 节 | 出处抽查（L4） | 二 | 【待填】 |
| REQ-2.4f | 一键反馈 | [ipc-contract.md](../design/ipc-contract.md) | | 二 | 【待填】 |
| REQ-2.4g | **多会话：新建 / 切换 / 历史恢复** | [architecture.md](../design/architecture.md) 3.2、[gui.md](../design/gui.md) 3.2 | | **一** | 【待填】 |
| REQ-2.5 | 第一版明确不做的事项 | [../02-scope-v1.md](../02-scope-v1.md) 第四节 | — | 两阶段 | 【已覆盖】 |

## 二、系统形态（需求文档第 3 节）

| 需求 ID | 需求 | 设计模块 | 测试用例 | 阶段 | 状态 |
|---|---|---|---|---|---|
| REQ-3.1 | 三条 HTTP 通道（检索/激活/上报） | [server-api.md](../design/server-api.md) | | 二 | 【待填】 |
| REQ-3.2 | 知识库管理（老师上传、解析、索引） | | | 二 | 【待填】 |
| REQ-3.3 | 服务端不感知 agent 内部结构 | | | 二 | 【待填】 |

## 三、硬规矩（需求文档第 4 节）——**不讨论，必须满足**

| 需求 ID | 硬规矩 | 设计模块 | 验证方式 | 阶段 | 状态 |
|---|---|---|---|---|---|
| REQ-4.1 | 不替老师打分 | — | 功能不存在即满足 | 两阶段 | 【已覆盖】 |
| REQ-4.2 | 首次使用明确告知并取得同意 | | | 二 | 【待填】 |
| REQ-4.3 | 发给大模型的请求不带学号姓名 | | | 二 | 【待填】 |
| REQ-4.4 | **标准答案、题库只存服务端，不落学生盘** | [data-model.md](../design/data-model.md) 第 1 节 | | 二 | 【待填】 |
| REQ-4.5 | **每个动作界面可见可追溯，绝不隐瞒执行** | [permission-gate.md](../design/permission-gate.md) 第 11 节 | PC-E-* | **一** | 【待填】 |
| REQ-4.6 | 服务端收集的记录透明，学生自己能看 | | | 二 | 【待填】 |

## 四、成功指标（需求文档第 5 节）

> 📌 **本节整节属阶段二**（需求文档第 5 节已标注）。**阶段一另有一套验收标准**，见 [../02-scope-v1.md](../02-scope-v1.md) 第二节末的 A1–A5。

| 需求 ID | 指标 | 怎么测 | 阶段 | 状态 |
|---|---|---|---|---|
| REQ-5.1 | 每周活跃人数 | 服务端记录统计 | 二 | 【待填】 |
| REQ-5.2 | 反馈里"解决了问题"占比 | 反馈数据 | 二 | 【待填】 |
| REQ-5.3 | 周留存 | 服务端记录统计 | 二 | 【待填】 |
| REQ-5.4 | **出处抽查**（我们自己加的，非需求原文） | 20 道真题人工核对 | 二 | 【待填】 |
| REQ-5.5 | **阶段一验收 A1–A5**（我们自己定的，非需求原文） | 见 02-scope-v1.md | **一** | 【待定：待你确认】 |

## 五、覆盖情况小结

**按阶段分开数**（阶段一开工前必须让「一」这一栏的「待填」清零）：

| 阶段 | 条目数 | 已覆盖 | 待填 | 说明 |
|---|---|---|---|---|
| **阶段一** | 【待数】 | 0 | 【待数】 | REQ-2.1a / 2.2a / 2.2b / 2.2d / 2.2e / 2.3a-c / 2.4c / 2.4d / 2.4g / 4.5 / 5.5 |
| **阶段二** | 【待数】 | 0 | 【待数】 | 其余 |
| 两阶段 | 2 | 2 | 0 | REQ-2.5、REQ-4.1 |

【待填：填完这张表，就能看出**阶段一**还有哪些需求没有设计承接——那些就是开工前的漏项】

## 待决问题

| # | 问题 | 状态 |
|---|---|---|
| Q1 | 需求文档要不要补 ID | **【待决】建议补**——上表的 `REQ-x.y` 是我按章节推的，需求正文里还没有，两边对不上 |
| Q2 | ~~"待决"的那些需求，V1 到底做不做~~ | ✅ **已解决**：阶段划分把"待决"全部归位，见 [02-scope-v1.md](../02-scope-v1.md) 第五节 |
| Q3 | 阶段一的验收标准 A1–A5 是否照此采纳 | **【待决】** 见 [02-scope-v1.md](../02-scope-v1.md) 第二节末 |
