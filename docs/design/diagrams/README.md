# 架构图

> 版本：v0.2 ｜ 日期：2026-09-15

## 两张图

| 文件 | 内容 | 适用阶段 |
|---|---|---|
| `architecture-local-agent.svg` / `.png` | 本地 Agent 详细设计（GUI / Agent 宿主 / 子代理 / 权限闸 / 工具集） | **阶段一** |
| `architecture-client-server.svg` / `.png` | 端云交互总览（三条通道、数据归属、典型时序） | **阶段二**（整图） |

两张图的 v0.2 版本（2026-09-15）都已更新，SVG 与 PNG 内容一致。

> 📌 **图上标了 `【阶段二】` 的部件，阶段一不做。**
> 阶段划分见 [../../02-scope-v1.md](../../02-scope-v1.md)。

## SVG 是源文件，PNG 是导出副本

**改图只改 SVG，然后重新导出 PNG。** 不要直接编辑 PNG。

## 怎么重新导出 PNG

用 Edge 的无头模式（Windows 自带，无需装任何东西）：

```powershell
$edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"

& $edge --headless=new --disable-gpu --no-sandbox --hide-scrollbars `
        --force-device-scale-factor=2 `
        --window-size=1400,1120 `
        --screenshot="D:\CODE\ideas\docs\design\diagrams\architecture-local-agent.png" `
        "file:///D:/CODE/ideas/docs/design/diagrams/architecture-local-agent.svg"
```

- `--window-size` 要与 SVG 的 `width` / `height` 一致：本地图 `1400,1120`，端云图 `1400,920`
- `--force-device-scale-factor=2` 出 2 倍图（`2800x2240` / `2800x1840`），中文小字才看得清

### ⚠️ 在受限沙箱里跑不通

如果终端受文件/进程沙箱约束，Chromium **起不来**：

```
FATAL:mojo\public\cpp\platform\platform_channel.cc:183] Check failed: 拒绝访问 (0x5)
```

原因是 Chromium 的进程间通信（mojo）用**命名管道**，受限沙箱不允许程序开命名管道。
**这不是命令写错，是环境限制**——换参数也没用，需要放开沙箱权限才能跑。

⇒ 在本机普通终端里跑上面的命令即可。
