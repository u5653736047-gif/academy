# 构建与打包

> 版本：v0.2（骨架）｜ 状态：骨架 ｜ 日期：2026-09-15
> **适用阶段：两阶段**——**阶段一**：能跑起来即可（开发模式）；**阶段二**：打成安装包分发
> 上游：[../design/adr/0001-electron-desktop.md](../design/adr/0001-electron-desktop.md)
> 下游：发布流程

## 1. 范围

把 Electron 应用做成能启动的桌面应用（**阶段一**），以及打成学生能双击安装的包（**阶段二**）。

> 📌 **阶段一只需要"跑得起来"**：我自己的机器上能装依赖、能启动、能对话就行，不做安装包、不做签名、不做更新机制。
> 真正的分发（安装包、代码签名、自动更新）是**阶段二**的事——那时才有第二个用户。
> 见 [../02-scope-v1.md](../02-scope-v1.md) 第二节 P1-1 与第三节 P2-7。

## 2. 技术约束（已确认，来自源码）

| 约束 | 值 | 来源 |
|---|---|---|
| Node 版本 | **>= 22.19.0** | pi 的 `package.json` `engines` |
| 模块格式 | **纯 ESM**（`"type": "module"`） | pi 的 `package.json` |
| bash 依赖 | **Windows 上必须有 Git Bash** | `packages/coding-agent/docs/windows.md`、`src/utils/shell.ts` |

### 2.1 Windows 的 bash 问题【已确认，必须解决】

pi 的 `bash` 工具在 Windows 上按顺序找：

1. `~/.pi/agent/settings.json` 里的 `shellPath`
2. `C:\Program Files\Git\bin\bash.exe`（含 x86 变体）
3. PATH 上的 `bash.exe`（Cygwin / MSYS2 / WSL）

**都找不到就报错退出。** 而我们的 agent 重度依赖 bash（复现报错、跑测试）。所以必须二选一：

| 方案 | 说明 | 代价 |
|---|---|---|
| A. 要求装 Git for Windows | 学生自己装 | 增加安装门槛；但很多学生本来就有 |
| B. 打包一个 bash 进安装包 | 自带 busybox-w32 / Git Bash 精简版 | 体积增加；需要验证兼容性 |

【待决】选哪个。**这一条不解决，"双击即用"就是空话。**

### 2.2 Electron + ESM

- Electron 主进程用 ESM 入口
- 【待填】实测 Electron 内置 Node 版本是否 >= 22.19
- pi 包里有 `src/bun` 等运行时资源，**【待填】确认 asar 打包时需要 unpack 哪些**

## 3. 构建流程【待填】

```
安装依赖 → 构建渲染进程 → 构建/准备主进程 → 打包 → 签名 → 分发
```

【待填】每一步的具体命令与产物

## 4. 依赖策略

> ⚠️ **2026-09-15 修正**：本节上一版写"**第三方 pi 扩展包一律不引入**"。**那个结论已被 ADR-0006 / 0007 推翻**——决策明确要复用社区成熟扩展，不再自研。下面是修正后的策略。

**分两类对待**：

| 类别 | 策略 |
|---|---|
| **pi 官方包**（`@earendil-works/*`） | 正常依赖，随版本升级 |
| **第三方 pi 扩展包**（本次复用） | **允许引入，但必须锁精确版本**（`package.json` 写死版本号，不用 `^` / `~`） |
| 我们自己写的小补丁 | 以内联扩展形式注入（`DefaultResourceLoader({ extensionFactories: [...] })`），**不经过 npm 分发** |

**要引入的第三方包**（阶段一）：

| 包 | 用途 | 依据 |
|---|---|---|
| `pi-subagents` | 子代理机制 | [ADR-0006](../design/adr/0006-subagent-process-model.md) |
| `@gotgenes/pi-permission-system` | 权限闸判定 | [ADR-0007](../design/adr/0007-permission-gate-hook-point.md) |

⚠️ **两个必须验证/注意的点**：

1. **这两个包的运行时 `exports` 指向 `.ts` 源码**，靠 pi 自带的 jiti 加载。**Electron 打包链能否加载，是技术验证的第一个待验项。**
2. **供应链风险真实存在**：`@gotgenes/pi-permission-system` 4 个月内发了 210 个版本且有破坏性变更，必须锁精确版本。调研也发现同名包在 10+ 个 namespace 下各自发布（见 [research/README.md](../research/README.md)）。

【待决】是否用 `noExtensions: true` 收敛扩展面（只加载我们指定的包，关掉自动发现）——见 [permission-gate.md](../design/permission-gate.md) 第 12 节。

## 5. 待决问题

**阶段一必须定的**（否则连启动都做不到）：

| # | 问题 | 备注 |
|---|---|---|
| Q1 | **bash 依赖怎么解决（方案 A：要求装 Git Bash / B：打包一个 bash）** | 阶段一我用它干活，bash 是刚需。**这条不解决，应用在我自己机器上都跑不起来** |
| Q2 | Electron 版本与内置 Node 版本（pi 要求 **>= 22.19.0**） | 阶段一第一个坑 |
| Q7 | ~~是否跳过安装包直接跑开发模式~~ | ✅ **已由阶段划分回答：阶段一就跑开发模式，安装包放阶段二** |

**阶段二再定的**：

| # | 问题 |
|---|---|
| Q3 | asar 打包的 unpack 清单 |
| Q4 | 安装包体积上限 |
| Q5 | 要不要代码签名（不签名 Windows 会弹 SmartScreen 警告） |
| Q6 | 更新机制（自动更新 / 手动重装） |

> 📌 Q1 的阶段归属要注意：**阶段一**需要"开发机上 bash 可用"（我自己装 Git Bash 即可）；
> **阶段二**才需要"别人机器上开箱可用"（这才逼出"打包一个 bash 进去"的方案 B）。
> 也就是说：**阶段一可以先用方案 A 走通，方案 B 留到分发时再解决。**

## 6. 参考

- `packages/coding-agent/docs/windows.md` —— Windows 上的 bash 要求
- `packages/coding-agent/src/utils/shell.ts` —— 找 bash 的具体逻辑
- pi 仓库的 `scripts/build-binaries.sh` —— 官方怎么做独立二进制
