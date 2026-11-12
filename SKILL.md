---
name: paraview-pitfalls
description: >-
  ParaView post-processing and contour-rendering pitfall handbook (ParaView 5.6.3 bundled with OpenFOAM 9 / pvpython / Python 2.7 without numpy, WSL environment). Use when the user mentions "paraview", "pvpython", contour rendering, screenshots, SaveScreenshot, color scale, LUT, camera, framing, uniform scale, transparent-background PNG, OpenFOAM post-processing, phase-diagram rendering, or batch figure generation, or when writing/debugging pvpython scripts or batch-rendering physical fields. Five confirmed traps: the first Render forces ResetCamera, ColorBy resets the color range, SaveScreenshot keyword-arg view silently outputs 400×400, color-preset names fail silently, and UpdatePipeline loops freeze time after Fetch. Chinese companion: SKILL.zh-CN.md.
---

# ParaView post-processing (pvpython 5.6.3 field-tested pitfalls)

ParaView 5.6.3 bundled with OpenFOAM 9 (pvpython, Python 2.7, no numpy), WSL environment, rendering `rheoInterFoam` field contours. Read this skill before writing any pvpython script, changing camera framing, or batch-rendering.

## Five confirmed traps (memorize these first)

| Trap | One-line countermeasure |
|---|---|
| The first `Render()` forces `ResetCamera()`, overwriting the manual camera | call `pv._DisableFirstRenderCameraReset()` after creating the view, before Render |
| `ColorBy` internally rescales the LUT to each panel's data — the "unified color scale" silently fails | the shared `RescaleTransferFunction` must be written **after** `ColorBy` |
| `SaveScreenshot` with the view as a keyword argument → silently outputs 400×400 | pass the view as a **positional** argument |
| `ApplyPreset` **fails silently** on nonexistent color-map names (`"Inferno"` is a 3-point wrong approximation; the real name is `"Inferno (matplotlib)"`) | after Apply, compare the `lut.RGBPoints` point count and end colors to verify |
| After `Fetch`, looping `UpdatePipeline(t)` freezes time at the first step | per timestep, Delete everything and rebuild a fresh pipeline |

## Content map

- Environment startup, camera and the "uniform scale" implementation, pipeline API, output verification and the fake-bbox trap, data snapshot preparation, the surrounding WSL/encoding environment — full detail in [references/paraview-pitfalls.md](references/paraview-pitfalls.md).
- Official Python API reference (use the 5.6.2 edition — same across the 5.6 series): https://www.paraview.org/paraview-docs/v5.6.2/python/ .

## Scope boundaries

- Composing rendered contours / swapping figures in Illustrator: the `adobe-manual` skill;
- matplotlib data-figure layout: the `academic-plotting` skill;
- PowerShell/WSL command-chain pitfalls when driving pvpython (escaping, BOM): see references/paraview-pitfalls.md §6.
