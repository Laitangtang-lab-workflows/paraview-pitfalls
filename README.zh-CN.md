# paraview-pitfalls

**pvpython 批量渲染不踩坑：ParaView 5.6.3 加 OpenFOAM 9 的全部实锤坑——相机静默重置、逐格色标、400×400 截图、假空数据、时间冻结——每条都配经核验的对策。**

> [!NOTE]
> 本仓**全部内容由 AI 维护**，人类定方向、做决策，详见 [AI 维护声明](#ai-维护声明)。英文说明见 [README.md](README.md)。

## 解决什么问题

用脚本驱动 ParaView 出可复现的论文级图，一路都是静默失败：渲染器收下你的相机然后扔掉；每张图悄悄用各自的色标范围；一个关键字参数把 1800×1200 渲染变成 400×400；预设名不存在就无声失败；提取的数据冻结在首时刻还自称成功。本手册每一条都经实战实锤（多数直接来自真实批量渲染），每条带核验过的修法与检测手段：

- **相机与比例一致**——`S = IMG_H / (2×ParallelScale)` 不变式、锚定相机公式、阻止首次 Render 重置相机的私有 API
- **管线与 API**——5.6 老属性名（`ThresholdRange`、`Invert`）、Clip 与 Threshold 的几何差异、`ColorBy` 重置色标坑、offscreen 色板整体暗化、boundary 与场 patch 条数错位
- **验证纪律**——alpha 通道 bbox 检查、白像素假 bbox 陷阱、逐格 md5 作冻结检测器、"先测量后渲染"
- **一句话速查表**——17 条"症状 → 原因 → 对策"，含预设名陷阱（为何 `"Inferno"` 是错的、`"Inferno (matplotlib)"` 才对）

限定 ParaView 5.6.3 / pvpython / Python 2.7 无 numpy（OpenFOAM 9 配套版本）——网上大量面向"新版" ParaView 的建议在这个版本上恰好是错的。

## skill 形态与安装

本仓核心是一个 **`SKILL.md` 格式的 agent skill**（Claude Code、DeepSeek Harness 等框架通用）。clone 进技能目录，AI 会话即可带着整本手册工作：

```
git clone https://github.com/Laitangtang-lab-workflows/paraview-pitfalls ~/.agents/skills/paraview-pitfalls
```

当普通文档读同样有效。

## 仓库结构

| 路径 | 内容 |
|---|---|
| `SKILL.md` | skill 入口：五大实锤坑 |
| `references/paraview-pitfalls.md` | 手册全量：环境、相机、管线 API、验证、快照、速查表 |

## AI 维护声明

本仓全部内容（文档、本 README）由 AI 会话在人类监督下编写与更新，每个坑都是 AI 会话批量渲染真实图时先踩到的。提交均由机器生成，人类把关方向与决策，不逐行审码。

## 证书

[CC0-1.0](LICENSE)——公有领域贡献。可任意使用、fork、并入任何项目，无须署名（注明出处是善意但非义务）。

提交 PR 即表示同意你的贡献同样以 CC0 贡献。
