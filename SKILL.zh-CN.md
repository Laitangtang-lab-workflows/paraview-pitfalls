---
name: paraview-pitfalls
description: >-
  ParaView 后处理与云图渲染踩坑手册（OpenFOAM 9 配套 ParaView 5.6.3 / pvpython / Python 2.7 无 numpy，WSL 环境）。当用户提到 "paraview""pvpython""云图""渲染""截图""SaveScreenshot""色标""LUT""相机""取景""比例一致""透明底 PNG""OpenFOAM 后处理""相图渲染""批量出图"，或要写/调 pvpython 脚本、批量渲染物理场云图时触发。五大实锤坑：首次 Render 强制 ResetCamera、ColorBy 重置色标范围、SaveScreenshot 关键字传参输出 400×400、色图预设名静默失败、Fetch 后循环 UpdatePipeline 时间冻结。英文正本见 SKILL.md。
---

# ParaView 后处理（pvpython 5.6.3 实战踩坑）（中文伴侣件）

> 本文件为中文速览，英文正本（内容一致、以它为准）见 [SKILL.md](SKILL.md)，深读材料 references/ 仅有英文版。

OpenFOAM 9 配套 ParaView 5.6.3（pvpython，Python 2.7，无 numpy），WSL 环境，`rheoInterFoam` 后处理云图。写 pvpython 脚本 / 改相机取景 / 批量渲染前先过本 skill。

## 五大实锤坑（先记这几条）

| 坑 | 一句话对策 |
|---|---|
| 首次 `Render()` 强制 `ResetCamera()`，手动相机被覆盖 | 建视图后、Render 前调 `pv._DisableFirstRenderCameraReset()` |
| `ColorBy` 内部按本格数据重缩放 LUT，"统一色标"失效 | 共享 `RescaleTransferFunction` 必须写在 `ColorBy` **之后** |
| `SaveScreenshot` 的 view 用关键字传参 → 静默输出 400×400 | view 用**位置参数**传 |
| `ApplyPreset` 对不存在的色图名**静默失败**（如 `"Inferno"` 只是 3 点错误近似，真名 `"Inferno (matplotlib)"`） | Apply 后比较 `lut.RGBPoints` 点数与首尾色核实 |
| `Fetch` 之后循环 `UpdatePipeline(t)` 时间冻结在首时刻 | 每时刻 Delete 全部对象、重建全新管线 |

## 内容导航

- 环境启动、相机与"比例一致"实现、管线 API、输出验证与假 bbox、数据快照准备、外围 WSL/编码环境——全量细节见 references/paraview-pitfalls.md（英文）。
- 官方 Python API 参考：https://www.paraview.org/paraview-docs/v5.6.2/python/ （5.6 系通用）。

## 分工边界

- 渲染出的云图组版/换图（Illustrator）：`adobe-manual` skill；
- matplotlib 数据图版式：`academic-plotting` skill；
- 驱动 pvpython 的 PowerShell/WSL 命令链坑（转义、BOM）：见 references/paraview-pitfalls.md §6。
