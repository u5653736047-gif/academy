# academy

**课程 Agent 助教** —— 给编程课学生的本地 AI 助教。

一句话说清它是干什么的：一个装在学生电脑上的 coding agent，能直接读他的代码、跑他的代码、把问题修好；解释问题的时候，引用课程资料（教材、课件）作为依据。

底座是 [pi](https://pi.dev)。

## 当前状态

**2026-09-15 转向**：阶段一不再从零自己写，改为**基于现成的 pi 桌面端 `PI-Desktop` 改造**（GUI 优化为主）。

| | 计划 |
|---|---|
| **阶段一** | 以 [`vastsa/PI-Desktop`](https://github.com/vastsa/PI-Desktop) 为基座，改造界面与教学场景适配 |
| **阶段二** | 按原计划补全教学特色：课程资料检索、检索子代理、服务端知识库 |

- 需求文档：[`docs/01-requirements.md`](./docs/01-requirements.md)
- 基座代码：`pi-desktop/`（本地克隆，**未纳入本仓库**，见 `.gitignore`）
- ⚠️ 基座许可为 **LGPL-3.0**，分发前需确认合规

### 基座代码的版本与来历

| 项 | 值 |
|---|---|
| 上游 | `https://github.com/vastsa/PI-Desktop.git` |
| 落地版本 | `b48c11d13d4025f7a11917969b5f056dcbdef53d`（2026-09-16，main） |
| 文件数 | 1708（`git status` 干净，工作区与上游完全一致） |
| 本地形态 | **浅克隆**（`--depth 1`）+ 无 promisor 的普通仓库 |

**拿下来的过程有个坑，记下来免得下次踩**：本机直连 GitHub 极慢且频繁断流（`github.com:443` 时通时断，大流量被限到 ~12–30 KiB/s）。最终做法是——

1. `git clone --depth 1 --filter=blob:none` 拿到提交和目录树（很小）
2. 用 `raw.githubusercontent.com` **并发逐文件拉** 全部 1708 个 blob（16 并发，每个带重试）
3. 大文件（7.4MB 中文字体）改走国内镜像 `gh-proxy.com`（~150 KiB/s，比直连快 5 倍）
4. 摘掉 `remote.origin.promisor` / `partialclonefilter`，`git add -A -f` 用工作区内容补齐本地对象库 → 仓库自给自足、离线可用
5. 用 **git blob 哈希** 校验：`git status` 干净 = 内容与上游逐字节一致

**以后要补全历史**：网络好时执行 `git fetch --unshallow`。

> 顺带一个可复用的结论：**这台机器上拉 GitHub 走 `gh-proxy.com` 前缀**，比直连快一个数量级。

> **设计文档已于 2026-09-15 全部清空**（转向前的架构/ADR/详细设计均作废）。
> 需要旧版内容可从 git 历史取回：`git show 1dbd51b:docs/README.md`。

## 三条要守住的原则

1. **不替老师打分。** 评分永远是人的事。
2. **标准答案、题库只存服务端，不落学生磁盘。**
3. **agent 对学生机器的每个动作全部可见、可追溯。** 权限档位决定"要不要先问"，不决定"要不要告诉你"。
