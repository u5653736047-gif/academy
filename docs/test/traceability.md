# 需求追溯矩阵

> 版本：v0.1（骨架）｜ 状态：草稿 ｜ 日期：2026-09-14
> 上游：[../01-requirements.md](../01-requirements.md)、[../design/](../design/)
> 下游：[test-plan.md](./test-plan.md)、[permission-cases.md](./permission-cases.md)

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

| 需求 ID | 需求 | 设计模块 | 测试用例 | 状态 |
|---|---|---|---|---|
| REQ-2.1 | 三个工具：检索课程资料 / 读写项目文件 / 执行命令 | [permission-gate.md](../design/permission-gate.md)、[server-api.md](../design/server-api.md) | | 【待填】 |
| REQ-2.2a | 主 agent 接待学生、调度子代理 | [adr/0006](../design/adr/0006-subagent-process-model.md) | | 【待填】 |
| REQ-2.2b | 通用子代理基座（能力完整，长时复杂任务） | [adr/0006](../design/adr/0006-subagent-process-model.md) | | 【待填】 |
| REQ-2.2c | 资料检索子代理（上下文防火墙，只读） | [adr/0006](../design/adr/0006-subagent-process-model.md) | | 【待填】 |
| REQ-2.2d | **改文件/跑命令，主与子代理走同一道闸** | [permission-gate.md](../design/permission-gate.md) 第 9 节 | PC-E-06 | 【待填】 |
| REQ-2.2e | 子代理动作可见、可中止 | [gui.md](../design/gui.md)、[ipc-contract.md](../design/ipc-contract.md) | | 【待填】 |
| REQ-2.3a | 权限三档，学生可切换 | [permission-gate.md](../design/permission-gate.md) 第 8 节 | PC-F-* | 【待填】 |
| REQ-2.3b | 自动审批档：黑白名单 + 灰区判官 | [permission-gate.md](../design/permission-gate.md) 第 3、7 节 | PC-A/B/D-* | 【待填】 |
| REQ-2.3c | 只读动作任何档位不拦 | [permission-gate.md](../design/permission-gate.md) | PC-A-01 | 【待填】 |
| REQ-2.4a | 安装：Electron 安装包，双击即用 | [adr/0001](../design/adr/0001-electron-desktop.md)、[../ops/build.md](../ops/build.md) | | 【待填】 |
| REQ-2.4b | 课程激活（邀请码 + 姓名学号） | [server-api.md](../design/server-api.md) 第 3 节 | | 【待决】可能砍 |
| REQ-2.4c | 模型配置：学生填自己的 API key | | | 【待填】 |
| REQ-2.4d | 干活过程实时可见 | [gui.md](../design/gui.md) 3.1、[ipc-contract.md](../design/ipc-contract.md) | | 【待填】 |
| REQ-2.4e | **回答有出处**（只从检索片段引用） | [server-api.md](../design/server-api.md) 第 2 节 | 出处抽查（L4） | 【待填】 |
| REQ-2.4f | 一键反馈 | | | 【待填】 |
| REQ-2.5 | 第一版明确不做的事项 | [../02-scope-v1.md](../02-scope-v1.md) | — | 【已覆盖】 |

## 二、系统形态（需求文档第 3 节）

| 需求 ID | 需求 | 设计模块 | 测试用例 | 状态 |
|---|---|---|---|---|
| REQ-3.1 | 三条 HTTP 通道（检索/激活/上报） | [server-api.md](../design/server-api.md) | | 【待填】 |
| REQ-3.2 | 知识库管理（老师上传、解析、索引） | | | 【待填】 |
| REQ-3.3 | 服务端不感知 agent 内部结构 | | | 【待填】 |

## 三、硬规矩（需求文档第 4 节）——**不讨论，必须满足**

| 需求 ID | 硬规矩 | 设计模块 | 验证方式 | 状态 |
|---|---|---|---|---|
| REQ-4.1 | 不替老师打分 | — | 功能不存在即满足 | 【已覆盖】 |
| REQ-4.2 | 首次使用明确告知并取得同意 | | | 【待决】可能砍 |
| REQ-4.3 | 发给大模型的请求不带学号姓名 | | | 【待填】 |
| REQ-4.4 | **标准答案、题库只存服务端，不落学生盘** | [data-model.md](../design/data-model.md) 第 1 节 | | 【待填】 |
| REQ-4.5 | **每个动作界面可见可追溯，绝不隐瞒执行** | [permission-gate.md](../design/permission-gate.md) 第 11 节 | PC-E-* | 【待填】 |
| REQ-4.6 | 服务端收集的记录透明，学生自己能看 | | | 【待决】可能砍 |

## 四、成功指标（需求文档第 5 节）

| 需求 ID | 指标 | 怎么测 | 状态 |
|---|---|---|---|
| REQ-5.1 | 每周活跃人数 | 服务端记录统计 | 【待填】 |
| REQ-5.2 | 反馈里"解决了问题"占比 | 反馈数据 | 【待填】 |
| REQ-5.3 | 周留存 | 服务端记录统计 | 【待填】 |
| REQ-5.4 | **出处抽查**（我们自己加的，非需求原文） | 20 道真题人工核对 | 【待填】 |

## 五、覆盖情况小结

| 状态 | 数量 | 说明 |
|---|---|---|
| 已覆盖 | | |
| 待填 | | |
| 待决（取决于目标用户范围） | | |

【待填：填完这张表，就能看出哪些需求还没有设计承接——那些就是设计阶段的漏项】

## 待决问题

| # | 问题 |
|---|---|
| Q1 | 需求文档要不要补 ID（**建议补**） |
| Q2 | "待决"的那些需求，V1 到底做不做（见 [02-scope-v1.md](../02-scope-v1.md) 第三节） |
