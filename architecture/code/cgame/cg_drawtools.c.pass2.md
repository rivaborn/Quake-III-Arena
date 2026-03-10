# code/cgame/cg_drawtools.c — Enhanced Analysis

## Architectural Role

This file is the **HUD rendering primitive layer** for the cgame VM—the exclusive bottleneck through which all 2D UI coordinates, colors, and text rendering must pass before reaching the renderer. Every function here wraps `trap_R_*` syscalls and applies coordinate scaling, making `CG_AdjustFrom640` the single point of resolution independence for the entire client-side HUD. The file bridges high-level HUD logic (cg_draw.c, cg_scoreboard.c, cg_newdraw.c) to low-level renderer capabilities, implementing the virtual 640×480 → actual display pixel mapping that Q3A's UI philosophy depends on.

## Key Cross-References

### Incoming (who depends on this file)
- **cg_draw.c**, **cg_scoreboard.c**, **cg_newdraw.c**, **cg_info.c**: Call `CG_DrawStringExt`, `CG_DrawBigString`, `CG_DrawSmallString`, `CG_FillRect`, `CG_DrawRect` for all HUD elements
- **cg_players.c**, **cg_ents.c**: Call `CG_DrawPic` for 2D player/entity overlays
- **cg_predict.c**: May call `CG_DrawStrlen` to measure text
- All cgame renderers use `CG_FadeColor` and `CG_TeamColor` for transient effect timing and team-based coloring
- **cg_view.c**: Calls `CG_TileClear` before the 3D scene to clear viewport borders

### Outgoing (what this file depends on)
- **trap_R_SetColor**, **trap_R_DrawStretchPic**: Renderer syscalls (the only outbound dependencies outside cgame locals)
- **q_shared.c**: `Q_IsColorString`, `ColorIndex` for inline color code parsing
- **q_shared.h**: `VectorClear` macro, `TEAM_*` constants
- **cg_local.h**: Global `cg` (`cg_t`), `cgs` (`cgs_t`), `g_color_table` (defined in cg_main.c)
- **tr_types.h**: `qhandle_t` shader type

## Design Patterns & Rationale

**Single-Point Scaling** — `CG_AdjustFrom640` is *the* only place coordinate/size conversion happens. This centralizes the scaling model and permits future per-frame resolution adjustments (e.g., dynamic zoom, picture-in-picture) without touching HUD logic. Widescreen adjustment code is commented out (`#if 0`), suggesting it was prototyped but disabled—likely due to UI composition assumptions (e.g., center-aligned menus).

**Font Lookup Tables** — `propMap[128][3]` and `propMapB[26][3]` pre-bake UV coordinates and glyph widths at initialization, avoiding per-character calculations during rendering. This is a **static metric cache** pattern; comments note the code duplicates `q3_ui/ui_atoms.c`, indicating a shared design across both UI VMs.

**Static Color Buffers** — `CG_FadeColor` and `CG_TeamColor` return pointers to static `vec4_t` buffers. This is a **dangerous but efficient pattern** for frame-limited rendering: callers *must not* store returned pointers across frames. The tradeoff favors short-lived HUD operations where copy semantics are unnecessary.

**Drop-Shadow via Offset Render** — `CG_DrawStringExt` renders text twice: once in black at offset `(xx+2, yy+2)`, then in color at the original position. This **layered composition** approach is simpler than shader-based shadow kernels and fits Q3A's software-composited HUD pipeline.

## Data Flow Through This File

```
HUD Logic Layer (cg_draw, cg_scoreboard)
  ↓ [virtual 640×480 coords + RGBA/text]
CG_DrawRect / CG_FillRect / CG_DrawStringExt
  ↓ [apply cgs.screenXScale / screenYScale]
CG_AdjustFrom640
  ↓ [screen pixel coords]
trap_R_SetColor + trap_R_DrawStretchPic
  ↓
Renderer (GLimp back-end)
```

**Time-based effects**: `cg.time` → `CG_FadeColor` computes alpha using `startMsec` + `FADE_TIME` interval, enabling transient notifications (chat, item pickups) with built-in fade-out.

**Team/Health coloring**: Raw stats → `CG_GetColorForHealth` / `CG_TeamColor` → RGBA → `trap_R_SetColor` before text/rect rendering.

## Learning Notes

- **Era-specific design**: Q3A locks HUD to a virtual 640×480 grid. Modern engines use scalar resolution or DPI-aware layouts. The `#if 0` widescreen code suggests early awareness of the limitation.
- **Proportional fonts via UV**: Instead of glyph-by-glyph shader dispatch, Q3A bakes font metrics into lookup tables. Modern GPU-accelerated engines use signed-distance-field (SDF) or compute shaders.
- **Color code parsing**: Inline `^n` color markers (where `n` is a Doom-style color index) are parsed in `CG_DrawStringExt`. This is a lightweight alternative to markup; the `g_color_table` is a 16-entry fixed palette (indexed by `ColorIndex(ch)`).
- **Virtual resolution philosophy**: By centralizing scaling in one function, Q3A ensures all UI code is written in a resolution-neutral way—enabling port to different aspect ratios and displays without modifying 100+ HUD functions.

## Potential Issues

- **Static buffer lifetime**: `CG_FadeColor` and `CG_TeamColor` return stale pointers if called more than once per frame without re-reading. No documentation warns callers; a frame-local copy is safer but adds overhead.
- **Duplication with q3_ui**: The proportional/banner font code (lines ~340+) duplicates `q3_ui/ui_atoms.c` (noted in comments). Divergence risks inconsistent rendering across UI and cgame.
- **Disabled widescreen support**: The commented-out adjustment in `CG_AdjustFrom640` suggests the 640×480 grid was intended to flex, but was abandoned—possibly due to menu centering or aspect ratio issues in 2005-era hardware expectations.
- **No bounds checking**: `CG_DrawChar` and `CG_DrawStringExt` assume valid character codes; malformed strings or out-of-range indices could cause undefined behavior in `propMap` lookups.
