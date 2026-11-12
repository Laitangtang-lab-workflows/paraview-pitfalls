# paraview-pitfalls

**pvpython batch rendering without surprises: every confirmed trap of ParaView 5.6.3 + OpenFOAM 9 — silent camera resets, per-panel color ranges, 400×400 screenshots, fake empty data, frozen timesteps — with its verified countermeasure.**

> [!NOTE]
> This repository is **written and maintained entirely by AI agents** under human direction — see [AI-maintained](#ai-maintained).
> Documentation is English-first; a Chinese README is available ([README.zh-CN.md](README.zh-CN.md)), and a Chinese SKILL companion ([SKILL.zh-CN.md](SKILL.zh-CN.md)).

## Why

Scripting ParaView for reproducible, publication-grade figures is full of silent failures: the renderer accepts your camera and then throws it away; every panel quietly gets its own color range; a keyword argument turns your 1800×1200 render into 400×400; a preset name that doesn't exist fails without a word; extracted data freezes at the first timestep while claiming success. Each entry in this handbook was confirmed in the field (most against real batch renders), and each ships with a verified fix and a detection method:

- **Camera & uniform scale** — the `S = IMG_H / (2×ParallelScale)` invariant, anchored-camera formulas, and the private-API call that stops the first Render from resetting everything.
- **Pipeline & API** — old-name properties (`ThresholdRange`, `Invert`), Clip-vs-Threshold geometry, the `ColorBy`-resets-range trap, offscreen palette darkening, boundary/field patch-count mismatches.
- **Verification discipline** — alpha-channel bbox checks, the white-pixel fake-bbox trap, per-panel md5 as a freeze detector, "measure before render".
- **A one-line quick-reference table** mapping 17 symptoms → causes → fixes, including the preset-name trap (why `"Inferno"` is wrong and `"Inferno (matplotlib)"` is right).

Scoped to ParaView 5.6.3 / pvpython / Python 2.7 without numpy (the OpenFOAM 9 bundle) — where most online advice for "modern" ParaView is actively wrong.

## The skill format

The core of this repository is a **`SKILL.md` agent skill** (the convention used by Claude Code, DeepSeek Harness and other agent harnesses). Drop the repo into your harness's skill directory and an AI session picks up the whole handbook with it:

```
git clone https://github.com/Laitangtang-lab-workflows/paraview-pitfalls ~/.agents/skills/paraview-pitfalls
```

It works equally as plain documentation for human readers.

## Repository layout

| Path | Content |
|---|---|
| `SKILL.md` | Skill entry point: the five headline traps |
| `references/paraview-pitfalls.md` | The full handbook: environment, camera, pipeline API, verification, snapshots, quick-reference table |

## AI-maintained

Every file in this repository — documentation, this README — is written and updated by AI agents under human supervision, and every trap was first stepped on by an AI session batch-rendering real figures. Expect commits to be machine-authored; humans review direction and decisions, not lines.

## License

[CC0-1.0](LICENSE) — released to the public domain. Use, fork and vendor it into anything, no attribution required (a nod is always appreciated).

By submitting a pull request you agree that your contributions are dedicated under CC0 as well.
