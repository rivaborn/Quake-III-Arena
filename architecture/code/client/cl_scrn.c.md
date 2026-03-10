# code/client/cl_scrn.c

## File Purpose
Manages the screen rendering pipeline for the Quake III Arena client, orchestrating the drawing of all 2D screen elements (HUD, console, debug graphs, demo recording indicator) and driving the per-frame refresh cycle. It also provides a set of virtual-resolution drawing utilities used throughout the client and UI code.

## Core Responsibilities
- Initialize screen-related CVars and set the `scr_initialized` flag
- Convert 640×480 virtual coordinates to actual screen resolution
- Draw 2D primitives: filled rectangles, named/handle-based shaders, big/small chars and strings with color codes
- Drive the per-frame screen update, handling stereo rendering and speed profiling
- Dispatch rendering to the appropriate subsystem based on connection state (cinematic, loading, active game, menus)
- Maintain and render the debug/timing graph overlay

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `graphsamp_t` | struct | Holds a single debug graph sample: a float `value` and an `int` color index |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scr_initialized` | `qboolean` | global | Guards rendering; set `qtrue` by `SCR_Init`, checked in `SCR_UpdateScreen` |
| `cl_timegraph` | `cvar_t *` | global | CVar enabling time graph overlay |
| `cl_debuggraph` | `cvar_t *` | global | CVar enabling debug graph overlay |
| `cl_graphheight` | `cvar_t *` | global | CVar controlling graph pixel height |
| `cl_graphscale` | `cvar_t *` | global | CVar scaling graph sample values |
| `cl_graphshift` | `cvar_t *` | global | CVar shifting graph baseline |
| `current` | `static int` | static | Ring-buffer write cursor for `values[]` |
| `values[1024]` | `static graphsamp_t[]` | static | Circular buffer of debug graph samples |

## Key Functions / Methods

### SCR_Init
- **Signature:** `void SCR_Init(void)`
- **Purpose:** Registers all screen-related CVars and marks the screen system ready.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes five `cvar_t *` globals; sets `scr_initialized = qtrue`
- **Calls:** `Cvar_Get` ×5
- **Notes:** Must be called before `SCR_UpdateScreen` will execute.

### SCR_AdjustFrom640
- **Signature:** `void SCR_AdjustFrom640(float *x, float *y, float *w, float *h)`
- **Purpose:** Scales 640×480 virtual coordinates to the current `vidWidth`/`vidHeight`. Wide-screen compensation is present but `#if 0`-disabled.
- **Inputs:** Pointers to x, y, width, height (any may be NULL)
- **Outputs/Return:** In-place modification of passed pointers
- **Side effects:** None
- **Calls:** None (reads `cls.glconfig`)
- **Notes:** Central scaling primitive; all 2D drawing helpers call this before invoking `re.*`.

### SCR_DrawNamedPic
- **Signature:** `void SCR_DrawNamedPic(float x, float y, float width, float height, const char *picname)`
- **Purpose:** Registers a shader by name and blits it at virtual coordinates.
- **Inputs:** Virtual-space rect, shader name string
- **Side effects:** May register a new renderer shader asset
- **Calls:** `re.RegisterShader`, `SCR_AdjustFrom640`, `re.DrawStretchPic`

### SCR_FillRect
- **Signature:** `void SCR_FillRect(float x, float y, float width, float height, const float *color)`
- **Purpose:** Fills a virtual-coordinate rectangle with a solid color using `cls.whiteShader`.
- **Calls:** `re.SetColor`, `SCR_AdjustFrom640`, `re.DrawStretchPic`

### SCR_DrawStringExt
- **Signature:** `void SCR_DrawStringExt(int x, int y, float size, const char *string, float *setColor, qboolean forceColor)`
- **Purpose:** Draws a string with a drop shadow; parses inline `^x` color escape codes unless `forceColor` is set.
- **Inputs:** Position, glyph size, string, base color, color-override flag
- **Side effects:** Changes renderer color state (restored to NULL on exit)
- **Calls:** `re.SetColor`, `Q_IsColorString`, `Com_Memcpy`, `SCR_DrawChar`
- **Notes:** Shadow pass first (black), then colored text pass.

### SCR_DebugGraph
- **Signature:** `void SCR_DebugGraph(float value, int color)`
- **Purpose:** Pushes one sample into the 1024-entry ring buffer.
- **Side effects:** Writes `values[current & 1023]`, increments `current`

### SCR_DrawDebugGraph
- **Signature:** `void SCR_DrawDebugGraph(void)`
- **Purpose:** Renders the debug graph ring buffer as a bar chart at the bottom of the screen at native resolution.
- **Calls:** `re.SetColor`, `re.DrawStretchPic`
- **Notes:** Operates in native pixel coordinates, not virtual 640×480.

### SCR_DrawScreenField
- **Signature:** `void SCR_DrawScreenField(stereoFrame_t stereoFrame)`
- **Purpose:** Orchestrates one full frame: begins renderer frame, dispatches content by `cls.state`, draws UI VM, console, and debug graph.
- **Inputs:** `stereoFrame` (LEFT / RIGHT / CENTER)
- **Side effects:** Calls into renderer, UI VM, cgame rendering, sound stop
- **Calls:** `re.BeginFrame`, `VM_Call` (UI_IS_FULLSCREEN, UI_REFRESH, UI_SET_ACTIVE_MENU, UI_DRAW_CONNECT_SCREEN), `SCR_DrawCinematic`, `CL_CGameRendering`, `SCR_DrawDemoRecording`, `Con_DrawConsole`, `SCR_DrawDebugGraph`, `S_StopAllSounds`, `Com_Error`, `Com_DPrintf`
- **Notes:** Called once per eye; `uivm` must be non-NULL or the function returns early.

### SCR_UpdateScreen
- **Signature:** `void SCR_UpdateScreen(void)`
- **Purpose:** Top-level per-frame entry point; guards against recursion, handles stereo dual-pass, and calls `re.EndFrame`.
- **Inputs:** None
- **Side effects:** Calls `SCR_DrawScreenField` once or twice; calls `re.EndFrame`; writes `time_frontend`/`time_backend` if `com_speeds` is set
- **Calls:** `SCR_DrawScreenField`, `re.EndFrame`, `Com_Error`
- **Notes:** `recursive` static prevents re-entrant calls; resets to 0 after frame completes.

## Control Flow Notes
`SCR_Init` runs during client startup. Every engine frame the main loop calls `SCR_UpdateScreen` → `SCR_DrawScreenField` → `re.BeginFrame` … `re.EndFrame`. The state machine inside `SCR_DrawScreenField` selects what to render (cinematic, connection dialog, loading screen, or live game via `CL_CGameRendering`) before layering the UI VM, console, and optional debug overlays on top.

## External Dependencies
- **Includes:** `client.h` (transitively pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`)
- **Defined elsewhere:** `re` (`refexport_t`), `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `uivm` (`vm_t *`), `g_color_table`, `com_speeds`, `time_frontend`, `time_backend`, `cl_debugMove`, `VM_Call`, `Con_DrawConsole`, `CL_CGameRendering`, `SCR_DrawCinematic`, `S_StopAllSounds`, `FS_FTell`, `Com_Error`, `Com_DPrintf`, `Com_Memcpy`, `Q_IsColorString`, `ColorIndex`, `Cvar_Get`
