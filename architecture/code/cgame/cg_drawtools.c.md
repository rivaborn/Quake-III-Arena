# code/cgame/cg_drawtools.c

## File Purpose
Provides low-level 2D rendering helper functions for the cgame module, including coordinate scaling, filled/outlined rectangles, image blitting, character/string rendering, and HUD utility queries. All functions operate in a virtual 640×480 coordinate space and scale to the actual display resolution.

## Core Responsibilities
- Scale 640×480 virtual coordinates to real screen pixels via `cgs.screenXScale`/`screenYScale`
- Draw filled rectangles, bordered rectangles, and textured quads
- Render individual characters and multi-style strings (color codes, shadows, proportional fonts, banner fonts)
- Tile background graphics around a reduced viewport
- Compute time-based fade alpha and team color vectors
- Map health/armor values to a color gradient for HUD display

## Key Types / Data Structures
None (all types are defined in `cg_local.h` / `q_shared.h`).

| Name | Kind | Purpose |
|---|---|---|
| `propMap[128][3]` | file-static array | UV + width data for proportional font characters (ASCII 0–127) |
| `propMapB[26][3]` | file-static array | UV + width data for large banner font characters (A–Z) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `propMap` | `static int [128][3]` | file-static | Lookup table: pixel X, Y, width for proportional charset |
| `propMapB` | `static int [26][3]` | file-static | Lookup table: pixel X, Y, width for banner charset |

Reads external globals `cg` and `cgs` (defined in `cg_main.c`).

## Key Functions / Methods

### CG_AdjustFrom640
- Signature: `void CG_AdjustFrom640(float *x, float *y, float *w, float *h)`
- Purpose: Scales a 640×480 virtual rect to actual screen pixel coordinates.
- Inputs: Pointers to x, y, width, height in virtual space.
- Outputs/Return: Modified in place.
- Side effects: None.
- Calls: None.
- Notes: Widescreen adjustment is commented out (`#if 0`).

### CG_FillRect
- Signature: `void CG_FillRect(float x, float y, float width, float height, const float *color)`
- Purpose: Draws a solid colored rectangle using the white shader.
- Inputs: Virtual coords, RGBA color array.
- Outputs/Return: void.
- Side effects: Issues `trap_R_SetColor` / `trap_R_DrawStretchPic` render commands.
- Calls: `CG_AdjustFrom640`, `trap_R_SetColor`, `trap_R_DrawStretchPic`.

### CG_DrawRect
- Signature: `void CG_DrawRect(float x, float y, float width, float height, float size, const float *color)`
- Purpose: Draws a hollow rectangle border of given line thickness.
- Inputs: Virtual coords, border thickness, RGBA color.
- Side effects: Render commands via `CG_DrawTopBottom` / `CG_DrawSides`.
- Calls: `trap_R_SetColor`, `CG_DrawTopBottom`, `CG_DrawSides`.

### CG_DrawChar
- Signature: `void CG_DrawChar(int x, int y, int width, int height, int ch)`
- Purpose: Renders a single character from the bitmap charset (16×16 grid, 1/16 UV step).
- Inputs: Virtual position, cell size, ASCII character.
- Side effects: `trap_R_DrawStretchPic` using `cgs.media.charsetShader`.
- Notes: Skips space characters; masks `ch` to 8 bits.

### CG_DrawStringExt
- Signature: `void CG_DrawStringExt(int x, int y, const char *string, const float *setColor, qboolean forceColor, qboolean shadow, int charWidth, int charHeight, int maxChars)`
- Purpose: Full-featured string renderer: optional drop shadow, inline `^n` color codes, character limit.
- Inputs: Position, string, base color, flags, char dimensions, max char count.
- Side effects: Multiple `trap_R_SetColor` + `CG_DrawChar` calls; reads `g_color_table`.
- Calls: `trap_R_SetColor`, `CG_DrawChar`, `Q_IsColorString`, `ColorIndex`.

### CG_TileClear
- Signature: `void CG_TileClear(void)`
- Purpose: Fills the screen borders outside a reduced 3D viewport with a tiling background texture.
- Inputs: None (reads `cg.refdef`, `cgs.glconfig`).
- Side effects: Up to four `trap_R_DrawStretchPic` calls via `CG_TileClearBox`.
- Notes: No-ops when the viewport is full-screen.

### CG_FadeColor
- Signature: `float *CG_FadeColor(int startMsec, int totalMsec)`
- Purpose: Returns a white `vec4_t` with alpha fading out over the last `FADE_TIME` ms of a timed interval.
- Inputs: Event start time, total display duration.
- Outputs/Return: Pointer to a static `vec4_t`; `NULL` if expired or not started.
- Side effects: Writes static `color` buffer.
- Notes: Callers must not store the returned pointer across frames.

### CG_GetColorForHealth
- Signature: `void CG_GetColorForHealth(int health, int armor, vec4_t hcolor)`
- Purpose: Computes a health-derived RGBA color (black→red→yellow→white gradient).
- Inputs: Raw health and armor values.
- Outputs/Return: Fills `hcolor`; black if `health <= 0`.
- Calls: `VectorClear`.

### UI_DrawProportionalString / UI_DrawBannerString
- Purpose: Render variable-width proportional text or large uppercase banner text using pre-baked `propMap`/`propMapB` UV tables, supporting alignment (`UI_CENTER`, `UI_RIGHT`), drop shadow, inverse, and pulse styles.
- Side effects: `trap_R_SetColor`, `trap_R_DrawStretchPic`, reads `cg.time` for pulse (`sin`).
- Notes: Guarded by `#ifndef Q3_STATIC`; `UI_` prefix functions are shared with `q3_ui`.

## Control Flow Notes
This file is a **render-frame utility library**. Its functions are called during `CG_DrawActive` / `CG_DrawActiveFrame` (the per-frame HUD draw pass). `CG_TileClear` is called before the 3D scene when `cg_viewsize < 100`. No init or shutdown logic exists here.

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `cg` (`cg_t`), `cgs` (`cgs_t`), `g_color_table`, `Q_IsColorString`, `ColorIndex`, `VectorClear`
- **Trap calls**: `trap_R_SetColor`, `trap_R_DrawStretchPic` (renderer syscalls)
- **Constants used**: `FADE_TIME`, `PULSE_DIVISOR`, `PROP_*`, `PROPB_*`, `UI_CENTER/RIGHT/DROPSHADOW/INVERSE/PULSE/SMALLFONT/FORMATMASK`, `ARMOR_PROTECTION`, `BIGCHAR_*`, `SMALLCHAR_*`
