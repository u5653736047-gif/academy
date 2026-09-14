# academy

**课程 Agent 助教** —— 给编程课学生的本地 AI 助教。

一句话说清它是干什么的：一个装在学生电脑上的 coding agent，能直接读他的代码、跑他的代码、把问题修好；解释问题的时候，引用课程资料（教材、课件）作为依据。

底座是 [pi](https://pi.dev)。

> **当前阶段：设计阶段，尚无代码。**
> 下一步：拍板 [ADR-0006](./docs/design/adr/0006-subagent-process-model.md) 与 [ADR-0007](./docs/design/adr/0007-permission-gate-hook-point.md)，然后冻结设计基线。

## 从哪开始读

**→ [docs/README.md](./docs/README.md) —— 文档索引**

那份索引里有完整的文档清单、阅读顺序和每份文档的状态。

## 文档地图

| 目录 | 放什么 |
|---|---|
| [`docs/`](./docs/) | 需求、范围基线、术语表 |
| [`docs/research/`](./docs/research/) | 调研报告（pi 权限方案，五个渠道） |
| [`docs/design/`](./docs/design/) | 架构、ADR、详细设计、接口契约、数据模型 |
| [`docs/test/`](./docs/test/) | 测试计划、权限规则用例表、需求追溯矩阵 |
| [`docs/ops/`](./docs/ops/) | 构建打包、服务端部署 |

## 三条要守住的原则

1. **不替老师打分。** 评分永远是人的事。
2. **标准答案、题库只存服务端，不落学生磁盘。**
3. **agent 对学生机器的每个动作全部可见、可追溯。** 权限档位决定"要不要先问"，不决定"要不要告诉你"。
