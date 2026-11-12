# ParaView pitfalls (the full collection)

> **Scope**: ParaView **5.6.3** bundled with OpenFOAM 9 (pvpython, Python 2.7, no numpy), WSL environment, rendering field contours from `rheoInterFoam` cases.
> **When to read**: before writing any pvpython script, touching camera framing, or batch-rendering.
> **API reference**: use the official **5.6.2** docs — https://www.paraview.org/paraview-docs/v5.6.2/python/ (same across the 5.6 series; no separate 5.6.3 docs exist).

## 1. Environment and startup

- pvpython path: `/opt/paraviewopenfoam56/bin/pvpython` (bundled with OpenFOAM 9, **not on PATH**).
- Version 5.6.3 / Python 2.7.18 / **no numpy**: `PythonCalculator` unavailable; `ProgrammableFilter` works but the script must be pure Python (no `import numpy`).
- Do **not** use `docs.paraview.org/en/latest/` — the 5.9+ API changed (`ThresholdBetween`, `ColorBy` string formats, …) and is incompatible with 5.6.
- Headless rendering: `DISPLAY=:0` (WSLg) or `--force-offscreen-rendering` both work; pvpython **does not support `-c` inline code** — everything goes through script files.
- Render each panel in its own process (parameterized script); loop on the PowerShell/WSL side, one invocation per panel; ~2–5 s per panel. **Confirmed (2026-10-11 comparison round)**: a single process looping "create view → render → Delete" hit a **silent segfault** on the second case — and after `| grep`/`| tail` pipes the exit code still read 0 (an illusion); only the first two images came out. One panel per process: 16/16 clean.

## 2. Camera and framing (achieving "uniform scale" — the most important pitfall)

- 🚨 **The first `Render()` forces `ResetCamera()` (confirmed)**: `paraview.simple.Render()`'s first invocation executes `view.ResetCamera()` and re-renders, overwriting a manually set camera with "fit everything" — the symptom is "camera parameters are clearly set, but the output never changes". **Every "fixed camera" render before this fix was silently ineffective.** Fix: after creating the view and before `Render`, call **`pv._DisableFirstRenderCameraReset()`** (a private 5.6.3 function, note the underscore).
- Orthographic scale: **pixels per physical unit S = IMG_H / (2×ParallelScale)**. **The necessary and sufficient condition for "every contour shares one scale" = all images share the same S (same ParallelScale + same resolution)**; scaling each image individually (PIL resizing to each bbox) changes S per image and necessarily distorts physics — **no PIL content scaling; layout happens on the rendering side, always**.
- Anchored-camera formulas (place a world-coordinate anchor at a chosen pixel; +z up in the image, pixel y=0 at the top):
  ```
  px  = x_anchor + (0.5*IMG_W − x_left_px)/S      # e.g. material's left end → x=100
  pz  = z_anchor + (y_bottom_px − 0.5*IMG_H)/S    # e.g. material's bottom → y=1000
  ParallelScale = 0.5*IMG_H/S
  ```
- Anchors must follow **physical facts**: e.g. a substrate base that moves with the case parameter vs. a nozzle top that never does — bottom-alignment then sets pz per row (with **S still globally unique**); genuine physical differences between panels (a slow column's flange, etc.) are kept, never flattened away.
- After setting the camera, print `cam.GetParallelScale()/GetFocalPoint()/GetPosition()` into the log — identical scale across all panels in the log is the direct proof of a uniform scale.
- 🚨 **Render time is overridden by the view's ViewTime (confirmed 2026-10-11; re-confirmed and generalized 2026-10-28)**: calling `UpdatePipeline(t)` on the reader and then `Render()`/`SaveScreenshot` — the output time is driven by the **view's ViewTime** (regardless of the reader's updated time). Left unset, a whole batch freezes at one time (measured: frozen at the **first** time step — 27 cases × 5 steps, 405 images all showing step one, identical md5; an earlier pipeline had worked purely by luck on **snapshot cases** that hold a single target time directory, which masked that the render script never set ViewTime — the flaw surfaced the moment a raw 5-step case was rendered). **Fix: `view.ViewTime = t` before every Render/SaveScreenshot (both render modes).** Detection: per-step images of the same case that are byte-identical (equal file sizes) mean you're hit; after the fix, md5s differ and image extents grow monotonically with the material's spreading. Data-extraction pipelines (Fetch-style) are not affected by ViewTime.
- "Content may leave the canvas" = orthographic projection clips naturally; leave the right side unconstrained (long tails cut at the canvas edge).

## 3. Pipeline and API pitfalls

- **OpenFOAMReader**: pass `FileName` as a list `[case.foam]`; `MeshRegions=["internalMesh"]`; `Createcelltopointfiltereddata=1`; `UpdatePipeline(t)` loads the target time; `foam.CellArrays` lists all array names.
- **Half-domain symmetry plane = the y-min face**: `Slice` at `y_min+1e-4` (slicing exactly on a boundary face returns empty; not the material's mid-plane, not the full-domain mid-plane).
- **Material region via Clip, not Threshold**: `Clip` (`ClipType="Scalar"`, `Scalars=["POINTS","alpha.paste"]`, `Value=0.5`, **`Invert=0`** keeps the α≥0.5 side) gives smooth edges; Threshold selects whole cells and leaves jagged boundaries.
- Property names are the old 5.6 names: **`ThresholdRange`** (not `ThresholdBetween`), **`Invert`** (not `InsideOut`), `SliceType.Origin/Normal`, `ContourBy=["POINTS",name]` + `Isosurfaces`.
- Color scale: `lut.ApplyPreset("Cool to Warm (Extended)", True)`; `lut.RescaleTransferFunction(vmin, vmax)`. 🚨 **`ColorBy` resets the color range**: the last line inside `pv.ColorBy(rep, ...)` is `rep.RescaleTransferFunctionToDataRange()` — it rescales the LUT to **this panel's own min/max**, so a **shared range must be written after `ColorBy`**, otherwise every panel in a batch uses its own range (confirmed: 24 panels ranging from [−450,1793] to [−339,46543] — the "unified color scale" never existed). Verification: the script echoes `lut.RGBPoints` first/last control points; after rendering, sample pixels in a "relaxed region" and compare colors across panels. **NaN in a log-scale LUT renders as a solid green block** → `lut.NanColor=[1,1,1]` plus `ReplaceInvalidResults=1`/`ReplacementValue=0` in the Calculator as a double safeguard.
- **ScalarBar in offscreen screenshots shows text but no color block** (observed across multiple invocation styles on 5.6.3) → draw color bars in the matplotlib composition stage instead.
- **SaveScreenshot**: pass the view as a **positional argument** (`SaveScreenshot(path, view, ImageResolution=(1800,1200))`); mixing `view=view` with ImageResolution **silently outputs 400×400**. `TransparentBackground=True` produces transparent-background PNG directly. Equivalent: set `view.ViewSize=[W,H]` then `SaveScreenshot(path, view)`.
- Derived fields without numpy: prefer ParaView native filters (Gradient/Calculator/Contour/ProgrammableFilter in pure Python); `PythonCalculator` is unavailable.
- 🚨 **After `Fetch`, looping `UpdatePipeline(t)` freezes time (confirmed, data-extraction scenario)**: once `sm.Fetch()` pulls data on a pipeline, subsequent `foam.UpdatePipeline(t2)` on the reader plus downstream updates **still leave the output at the pre-Fetch time** (21 timesteps of extraction bit-identical; fresh pipelines at the same probes show genuine ±0.1% variation). A single switch without Fetch behaves — the first encounter passed a single-timestep check and only the full loop exposed it, deeply misleading. **Fix: per timestep, Delete everything and rebuild a fresh reader→Contour→Slice pipeline** (~1 s per timestep, negligible). Verdict experiment: extract the same probes at 2–3 timesteps with fresh pipelines and compare CSVs against the loop version for bit-identity. Also: pvpython 2.7 scripts with Chinese comments must declare `# -*- coding: utf-8 -*-` or Non-ASCII errors out.
- 🚨 **`GetDataInformation()` lies empty (confirmed 2026-10-11)**: calling `GetDataInformation()` on a filter (Slice/Clip) after setting properties but **before it actually executes** returns empty metadata with 0 points, 0 cells — treating that as "empty data" discards everything (symptom: all-None extraction results with no error). **All statistics and emptiness checks must go by the vtk data object pulled back via `servermanager.Fetch()`** (leaf-block counts, manual bounds union; note `vtkMultiBlockDataSet` has **no** `GetBounds`/`GetPointData` — walk the leaves).
- 🚨 **`cells/points in mesh and field don't match: mesh = N, field = M` = boundary vs. field patch-count mismatch (confirmed 2026-10-27, a 27-case batch all refused)**: N is a patch's `nFaces` in `constant/polyMesh/boundary`, M the field file's patch-segment nonuniform count — two mesh generations (a locally substituted mesh vs. the cluster's on-the-spot snappy mesh) differing by a few faces at an STL-interface patch triggers it: an error dialog at Apply, all fields unusable, and the time list **silently drops mismatched time dirs** (e.g. time 0 missing). Anchor for diagnosis and repair: a `phi` (surfaceScalarField) patch-segment count equals the true face count at write time — correct `nFaces` from it and shift `startFace` accordingly; afterwards verify the startFace chain stays contiguous and sums to the declared face count, plus a pvpython spot check with 0 errors. Mechanism and process lessons: see the openfoam-cluster-ops skill's harvest-workflow incident write-ups.
- 🚨 **Offscreen rendering darkens the whole palette ×(0.78,0.77,0.74) (confirmed 2026-10-29 in a contour review round)**: PNGs from pvpython 5.6.3 `--force-offscreen-rendering` + `SaveScreenshot` come out darker per channel than the LUT definition — Cool to Warm (Extended) end colors (0,0,89)/(89,18,54) render as (0,0,66)/(70,14,40); YlOrRd written directly into `RGBPoints` (256 points) darkens the same way ((165,0,38)→~(152,0,24) in-image; ratios not round numbers, pending controlled calibration). **The data→color mapping itself is correct** (the sliced over-threshold fraction matches the saturated-pixel fraction); the values are uniformly dark — pointing at offscreen color-space (gamma) handling. Affects both renderers and all series, old batches included. **Consequence**: a matplotlib-drawn color bar (standard preset colors) is ~25% brighter than the same value region in the contour — two color families coexist in one figure (separable by pixel tracking: the bar family clusters into strips, the contour family spreads as a region), and bar-reading underestimates stress by ~20%. **Resolution (2026-10-29, user ruling: no re-render)**: the figure is self-consistent, qualitative reading unaffected; if it ever matters, the symptom-level fix is re-making the color bar with the measured darkening coefficients, the root fix is LUT pre-compensation (RGBPoints ×1/0.78) and a full re-render. Audit method: `np.unique` 16-level quantization to find end-color families + per-family pixel-coordinate distribution to tell strips from regions.

## 4. Output verification and the "fake bbox" trap

- ⚠️ **White-pixel bbox trap (confirmed)**: detecting content bounding boxes via "non-white pixels" mistakes the near-white relaxed region (τ≈0) in the color scale for background — the measured bbox is far smaller than the material (an old handover value "1148×352" was exactly this fake bbox; truth was 2.86×0.879 world units). **Measure content extents with the same pipeline used for rendering**: after Slice→Clip, `GetDataInformation().GetBounds()`.
- Transparent-PNG verification: read the alpha channel and scan bboxes (Windows side via `System.Drawing` LockBits, alpha>16 counts as content), compare per panel against predicted anchors (left/top/bottom/right deviations).
- Keep an archived measure step and a per-panel verify step; **for any new state or new mechanical quantity, measure before rendering**.

## 5. Data and snapshot preparation

- Derived fields (tau_nn/trTheta and friends) exist only at target times → a raw case's many early steps lack these arrays and the **OpenFOAMReader won't see them**. Solution: hard-link snapshot cases (`constant/system/0/<target-time>` + `case.foam`) — costs no extra space.
- Color-scale conventions (settled by the user; do not change): signed quantities (τ_nn/τ_nt/trΘ) share **p5–p95 across three states**; absolute quantities (|F_st|) use 0–p95 (log scale: 1–p95); coolwarm/viridis/inferno chosen per quantity.
- Pipeline order (as landed): derived fields + statistics → snapshot cases → batch rendering with anchored cameras (per-panel scale) → per-panel verification → preview mosaic → composition. The old PIL-based composition script was retired (replaced by render-side layout + composition-stage assembly).

## 6. Surrounding runtime (WSL bridging + encodings)

Essentials: `wsl.exe` command lines shred `|`, `$` and double quotes through double-escaping — anything complex goes into a `.sh` file; `.ps1` files must be UTF-8 with BOM; occasional EIO/file-lock hiccups while editing on a synced drive — retry; ensure output directories exist before rendering (SaveScreenshot **fails silently** on a missing directory, and getsize errors afterwards).

## 7. One-line quick reference

| Symptom | Cause | Countermeasure |
|---|---|---|
| Camera parameters set, output unchanged | first `Render()` forces ResetCamera | `pv._DisableFirstRenderCameraReset()` |
| Output is 400×400 | SaveScreenshot's view passed as keyword | pass the view positionally |
| Symmetry-plane Slice empty | sliced exactly on a boundary face | use `y_min+1e-4` |
| Jagged material boundaries | Threshold's whole-cell selection | switch to Clip (`Invert=0` keeps α≥0.5) |
| Content bbox too small | white-pixel trap (τ≈0 renders white) | measure via pipeline GetBounds |
| Per-panel color ranges differ / scale not unified | `ColorBy` internally rescales the LUT per panel | shared `RescaleTransferFunction` written **after** `ColorBy` |
| Solid green blocks in a log-scale map | NaN color unset | `lut.NanColor=[1,1,1]` + ReplaceInvalidResults |
| Color bar shows text but no color block | 5.6.3 offscreen screenshots don't render the ScalarBar body | draw color bars in the matplotlib composition |
| Derived fields invisible | they exist only at target times | hard-linked snapshot cases |
| Extraction all-None / 0 points, no error | `GetDataInformation()` returns fake-empty before execution | all statistics from the `Fetch()`ed vtk object |
| Per-step renders byte-identical | view `ViewTime` overrides the reader's time | explicit `view.ViewTime = t` before rendering |
| Multi-panel render dies silently with exit 0 | single-process view-loop segfault (pipes mask the code) | one panel per process; loop in the driver |
| `vtkMultiBlockDataSet` has no GetBounds | Fetch returns composite data | walk leaf blocks, union manually |
| Chinese path errors in .ps1 | PowerShell 5.1 misreads as GBK | UTF-8 BOM on the file |
| Colors wrong / only 3 control points | **preset-name trap (confirmed)**: `"Inferno"` in 5.6 is a 3-point wrong approximation; the real one is `"Inferno (matplotlib)"` (256 points); `"YlOrRd"` and other matplotlib maps don't exist; `ApplyPreset` **fails silently** on unknown names | probe: compare `lut.RGBPoints` point count and end colors after `ApplyPreset`; custom scales write `lut.RGBPoints = [...]` directly |
| `lut.GetRange()` errors | GetRange needs arguments in 5.6 | debug-print `lut.RGBPoints` end control points instead |
| Contours ~25% darker than the color bar | offscreen palette darkening ×(0.78,0.77,0.74) | self-consistent figures are fine; to fix, darken the bar with the same coefficients or pre-compensate the LUT and re-render (§3 last item) |

> **Preset-name trap details**: probing confirmed 5.6 ships only `Viridis/Plasma/Magma/Inferno (matplotlib)` (256 points each), `Cool to Warm (Extended)` (35 points), `Black-Body Radiation` (4 points) and the like; `Jet/Rainbow/Turbo/Hot/YlOrRd/OrRd/Reds` and plain `Cool to Warm` (without Extended) do not exist (silent failure keeps the default LUT).
