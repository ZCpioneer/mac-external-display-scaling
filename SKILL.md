---
name: mac-external-display-scaling
description: Use when a Mac external display has tiny UI or text and needs per-display HiDPI, scaling, resolution, or refresh-rate tuning without changing the built-in display.
metadata:
  short-description: Tune per-display Mac HiDPI scaling
---

# macOS 外接显示器缩放

## 核心目标

把“外接屏 UI 太小”处理成一个可验证的显示配置问题：只调整用户点名的外接显示器，优先使用 macOS 原生 HiDPI 缩放，尽量保留目标刷新率，并用系统报告与实际应用界面确认结果。

关键区分：物理面板分辨率、macOS 逻辑 UI 大小、HiDPI framebuffer、链路刷新率是不同字段。不要把分辨率标签本身当成 HiDPI 证据。

## 适用边界

- 适用于 macOS 外接显示器的 UI、字体、菜单或窗口过小问题，尤其是 QHD/4K 显示器连接 MacBook 时。
- 默认只处理用户明确指定的外接屏；内建屏幕的模式、缩放、刷新率和排列保持不变。
- 不用于单纯的颜色/HDR、线材故障、黑屏、闪烁或应用自身字号问题；这些应先分流到对应的诊断路径。

## 工作流程

### 1. 先建立只读基线

确认目标显示器的名称/标识后，再读取：

```sh
sw_vers
system_profiler SPHardwareDataType
system_profiler SPDisplaysDataType
```

如果系统已有 `displayplacer`，读取 `displayplacer list`；没有时先使用系统设置观察，只有确实需要精确切换或验证缩放标记时才考虑安装。连接方式用系统信息或针对性的 `ioreg` 验证，不要凭线材外观猜 HDMI、DP 或 USB-C。

基线至少记录：

- macOS 版本与 Apple Silicon/Intel；
- 目标外接屏的物理/EDID 原生分辨率、当前 UI Looks like、当前刷新率；
- 当前缩放状态（`displayplacer` 的 `Scaling: on/off`，或等价系统证据）；
- 连接类型、链路速率（如果系统能读到）；
- 内建屏幕的同一组字段，作为完成后的不变性对照。

### 2. 原生方案优先

在系统设置中只选中目标外接屏，并开启“显示所有分辨率”。按以下顺序寻找候选：

1. 约 2048×1152 的缩放/HiDPI 档；
2. 约 1920×1080 的缩放/HiDPI 档；
3. 介于两者之间且适合长期工作的中间档。

对每个候选一次只改一个变量，并确认刷新率仍为用户要求的值（例如 144Hz）。`displayplacer` 输出中应看到目标档位带 `scaling:on`；只有名称写着 2048×1152 或 1920×1080，不足以证明是 HiDPI。带 `scaling:off` 的同名模式可能是直接低于面板原生的输出，不能自动当作清晰缩放。

除非更高档位全部失败，不要使用 1600×900、1280×720 等明显过大的模式。不要把应用缩放当成全局显示缩放的替代品。

### 3. 安全应用与回滚

应用模式前保存目标屏的当前模式、刷新率、位置、方向和显示器标识。若使用 `displayplacer`：

- 从本次新鲜的 `displayplacer list` 读取 mode 编号；mode 编号可能随重新连接而改变，不能复用旧编号；
- 按显示器 ID 指向目标屏，不按排列图中的左右位置猜测；
- 若工具要求提交完整布局，原样保留内建屏幕配置，只替换目标外接屏的模式字段；
- 变更后立即重新读取系统状态；出现黑边、比例错误、明显模糊、刷新率下降或窗口异常时，恢复基线模式。

如果下一步会通过系统设置 UI 改动显示模式，遵守 `computer-use:computer-use` 的确认要求；需要确认时就在实际切换前暂停，不要通过底层命令绕过确认。

禁止修改 SIP、安全策略或系统底层 display override 文件，也不要使用不明来源的显示驱动。

### 4. BetterDisplay 只作必要 fallback

只有在 macOS 原生候选缺失、目标档明显模糊、无法保持所需刷新率，或原生档位无法达到合理 UI 大小时，才考虑 BetterDisplay。使用官方/可信来源的当前版本，先确认目标外接屏，再只给它配置 HiDPI 模式；不要碰内建屏幕。

安装或启动过程中出现管理员权限、密码、系统安全确认或驱动权限请求时，停下来让用户手动处理。BetterDisplay 也必须逐个候选测试并记录可回滚的上一档。

### 5. 对比并选择

至少对比：

- A：2560×1440 原生；
- B：2048×1152 左右的 HiDPI；
- C：1920×1080 HiDPI。

用同一批实际应用观察文字清晰度、UI 舒适度、桌面空间、刷新率、比例/黑边和长时间写代码的可用性。默认选择 B；若 B 仍偏小才选 C。不要为了“字更大”牺牲到明显低分辨率，也不要只凭一次截图宣称长期舒适。

### 6. 完成验证

切换完成后，用新鲜输出重新确认：

- 目标屏当前逻辑 UI 大小、缩放状态和刷新率；
- 目标屏的 EDID/面板原生分辨率与连接链路；
- 内建屏幕前后字段一致；
- 系统设置、浏览器、Finder、VS Code 或用户指定开发工具中的菜单/字体/窗口实际变大。

HiDPI 模式下，`system_profiler` 可能显示比面板原生更大的 framebuffer（例如逻辑 2048×1152 对应 4096×2304）。这应与 `UI Looks like: 2048×1152` 和 `Scaling: on` 一起解释，不能把 framebuffer 数字误报成物理面板分辨率或实际链路分辨率。

只有当全局显示缩放已经合理、且某个应用仍独立偏小时，才考虑浏览器 110%～125% 或 VS Code/Codex UI Zoom；应用微调必须单独记录，且不能掩盖全局缩放未解决的问题。

## 常见错误

| 错误 | 正确做法 |
|---|---|
| 只看 2048×1152 标签就宣称 HiDPI | 同时核对 `Scaling: on`、`UI Looks like` 和实际观感 |
| 把物理分辨率、逻辑分辨率、framebuffer 混为一谈 | 分别记录并在最终报告中说明三者关系 |
| 误选内建屏幕 | 每次按名称/ID确认目标屏，完成后对照内建屏前后状态 |
| 一次修改多个参数 | 一次只切换一个候选模式，切换后立刻验证 |
| 使用过期的 displayplacer mode 编号 | 每次重新读取 `displayplacer list` |
| 原生方案还没验证就安装 BetterDisplay | 先完成原生候选与刷新率/清晰度检查 |
| 先调浏览器或 VS Code 缩放 | 先解决全局显示比例，再做应用级微调 |

## 一个可复用的验证片段

把目标显示器名称替换成当前任务中的名称后，可用同一片段快速核对系统报告：

```sh
TARGET_DISPLAY='外接显示器名称'
system_profiler SPDisplaysDataType | rg -A 12 -B 2 "$TARGET_DISPLAY|Resolution|UI Looks like|Connection Type"
```

这段输出只能作为显示状态的一部分；仍需用 `displayplacer list`（若可用）、连接信息和实际应用截图交叉验证。
