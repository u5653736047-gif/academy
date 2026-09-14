# 构建与打包

> 版本：v0.1（骨架）｜ 状态：骨架 ｜ 日期：2026-09-14
> 上游：[../design/adr/0001-electron-desktop.md](../design/adr/0001-electron-desktop.md)
> 下游：发布流程

## 1. 范围

把 Electron 应用打成学生能双击安装的包。**只服务自己时这份文档可以先放着**（见 [02-scope-v1.md](../02-scope-v1.md) C1）。

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

**只依赖 pi 官方包**。第三方 pi 扩展包一律不引入——调研发现同名包在 10+ 个 namespace 下各自发布，供应链无法评估（见 [research/README.md](../research/README.md)）。我们自己写的东西以内联扩展的形式注入（`DefaultResourceLoader({ extensionFactories: [...] })`），不经过 npm 分发。

【待决】是否用 `noExtensions: true` 收敛扩展面（见 [permission-gate.md](../design/permission-gate.md) 第 12 节）。

## 5. 待决问题

| # | 问题 |
|---|---|
| Q1 | bash 依赖怎么解决（方案 A / B） |
| Q2 | Electron 版本与内置 Node 版本 |
| Q3 | asar 打包的 unpack 清单 |
| Q4 | 安装包体积上限 |
| Q5 | 要不要代码签名（不签名 Windows 会弹 SmartScreen 警告） |
| Q6 | 更新机制（自动更新 / 手动重装） |
| Q7 | 只服务自己时，是否可以跳过安装包，直接跑开发模式 |

## 6. 参考

- `packages/coding-agent/docs/windows.md` —— Windows 上的 bash 要求
- `packages/coding-agent/src/utils/shell.ts` —— 找 bash 的具体逻辑
- pi 仓库的 `scripts/build-binaries.sh` —— 官方怎么做独立二进制
