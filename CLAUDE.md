# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a **documentation-only repository** — no build, test, or run commands exist. All content is Markdown (Vietnamese) describing the redesign of an automated PCB CNC machine from STM32 to Siemens S7-1200 PLC, with the addition of automated optical inspection (AOI).

## Source-annotation convention

Every numeric value and design decision is tagged — **do not mix them up**:

| Tag | Meaning |
|---|---|
| `[ĐA]` | Verified data from the original thesis — measured or calculated by the authors |
| `[DS]` | Manufacturer datasheet specification |
| `[ĐX]` | **New design proposal — estimated, not yet validated, must be re-measured** |

When editing or extending documentation, always tag every new value with one of these markers.

## System architecture

Two processors, strict master–slave split, connected via Ethernet (S7 protocol, port 102, `python-snap7`):

```
Gerber ──USB──► Pi 4 ──► .nc ──► G-code analysis, z-compensation ──Ethernet──► PLC ──► stepper drivers ──► motors
                                                                                  │
                         inspection result ◄── YOLO ◄── image ◄── camera ◄───────┘
```

**Raspberry Pi 4 (Master)** owns all data-heavy work:
- Gerber + Excellon → `.nc` via `pcb2gcode`
- G-code parsing, arc linearisation, z-offset pre-computation, segment splitting
- 66-point leveling map: raw probe data → plane equations → z(x,y) function
- Tool-path ordering (nearest-neighbour)
- Full AOI pipeline: image stitching, HSV segmentation, Gerber XOR, YOLO inference
- PCB blank localisation and waste-optimisation

**S7-1200 1214C DC/DC/DC (Slave)** owns all real-time work:
- 3-axis PTO step/direction (X=PTO1, Y=PTO2, Z=PTO3)
- Spindle PWM speed + direction relay (ATC)
- Hardware interrupt on I0.3 for probe leveling (latency ~µs — mandatory; OB1 polling would cause z-error equal to one full scan cycle, which destroys PCB traces)
- Safety: hard limits, E-stop, driver alarms
- Camera positioning during AOI

**The PLC only executes ready-to-run commands** — absolute coordinates with z already compensated, speeds already scaled. It never parses files or does arithmetic on paths. This boundary is the central design decision; keep it strict.

## Critical hardware constraints

- **PTO/PWM channels only work on onboard outputs Q0.0–Q0.7**, never on SM 1223 expansion modules. This is a TIA Portal fixed constraint.
- `Q1.0` reverses the spindle (tool release during ATC). The PWM timing constants `T_nhả`/`T_siết` were measured at 12 V in the original thesis and **must be re-measured at 24 V** before commissioning.
- Probe I0.3 **must use hardware interrupt**, not OB1 polling.

## Flowchart notation

All Mermaid diagrams use this symbol set consistently:

| Notation | Shape | Meaning |
|---|---|---|
| `([...])` | Rounded | Start / End |
| `[/.../]` | Parallelogram | Input / Output |
| `[...]` | Rectangle | Process |
| `{...}` | Diamond | Decision (two branches: **Đúng** / **Sai**) |

## Document map

| File | Role |
|---|---|
| `docs/thiet-bi-va-chuc-nang.md` | Full BOM and feature list — primary reference for hardware specs |
| `docs/chuc-nang-plc.md` | PLC spec: I/O allocation, function blocks, Data Blocks, wiring |
| `docs/gerber-sang-nc.md` | Toolpath generation pipeline: Gerber + Excellon → `.nc` |
| `docs/kiem-tra-quang-hoc.md` | AOI algorithm: tiling, HSV segmentation, Gerber XOR, YOLO |
| `docs/dinh-vi-va-toi-uu-phoi.md` | PCB blank localisation and waste-area optimisation |
| `docs/luu-do-giai-thuat.md` | All 10 system flowcharts collected in one place |
| `docs/so-sanh-stm32-vs-s7.md` | Original STM32 design vs. this PLC redesign — what changed and why |
| `docs/images/` | Pre-exported PNG + SVG versions of all flowcharts for reports |

## System state machine

States: `IDLE → HOMING → LOCATING → LEVELING → READY → RUNNING ⇌ PAUSED → INSPECTING → READY`. Error from any active state returns to `IDLE` via Reset. Transitions are documented in `docs/luu-do-giai-thuat.md` §10.

## AOI tile memory constraint

The PCB work area is captured as 16 tiles. Each raw tile is ~24 MB; all 16 together are ~0.39 GB, which exhausts Pi 4 RAM. The pipeline **releases each tile immediately after extracting defect candidates** — YOLO runs only on small candidate crops, not on full images.
