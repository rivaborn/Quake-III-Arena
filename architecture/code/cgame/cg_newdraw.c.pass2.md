# code/cgame/cg_newdraw.c — Enhanced Analysis

## Architectural Role

This file is the **MissionPack team-game HUD renderer**, bridging the data-driven UI system (`code/ui/ui_shared.c`) with live cgame state. It implements the owner-draw callback mechanism: the UI framework invokes `CG_OwnerDraw` during menu painting (`Menu_PaintAll`) to render dynamic HUD elements (team overlays, player portraits, medals, scores). It also routes mouse and keyboard input from the cgame back to the UI system's widget event handlers, completing a closed loop: input → UI dispatch → cgame state updates → next frame's render.

## Key Cross-References

### Incoming (who depends on this file)
- **UI subsystem** (`code/ui/ui_shared.c`): Calls `Display_OwnerDraw` callback (which points to `CG_OwnerDraw`) during `Menu_PaintAll` render phase for each owner-draw element ID
- **Input system** (`cg_main.c`): Routes mouse deltas and key events to `CG_MouseEvent` and `CG_KeyEvent` for forwarding to UI framework
- **cg_draw.c**: Provides computed state (`sortedTeamPlayers[]`, `numSortedTeamPlayers`, chat strings) via file-scope externs

### Outgoing (what this file depends on)
- **Renderer** (`code/renderer/`): Calls `CG_DrawPic`, `CG_Draw3DModel`, `CG_Text_Paint`, `trap_R_SetColor`, `trap_R_RegisterShader`
- **UI framework** (`code/ui/ui_shared.c`): Calls `Display_MouseMove`, `Display_HandleKey`, `Display_CaptureItem` for event routing; reads `cgDC` context
- **cgame state layer**: Reads `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items` globals; calls `CG_ConfigString`, `CG_GetColorForHealth`, `CG_StatusHandle`
- **Game layer** (`code/game/bg_misc.c`): Calls `BG_FindItemForPowerup` for item lookups
- **VM boundary**: Syscalls `trap_Cvar_Set`, `trap_SendConsoleCommand`, `trap_Key_SetCatcher`

## Design Patterns & Rationale

**1. Owner-Draw Dispatch Pattern**: `CG_OwnerDraw` is a massive switch statement (~1850 lines) with ~150 cases, each calling a static helper (`CG_Draw*`). This mimics Win32's `WM_DRAWITEM` owner-drawn control pattern: the UI framework owns the layout, cgame owns the rendering logic. Tradeoff: monolithic but decouples UI layout from game logic.

**2. Visibility Predicates**: `CG_OwnerDrawVisible` performs sequential flag checks (`if (flags & CG_SHOW_DURINGINCOMINGVOICE) ...`), not exclusive OR. This allows a single element to satisfy multiple conditions. Modern engines use data-driven visibility queries; Q3 uses bit flags for performance.

**3. Callback Vtable**: `cgDC` (a `displayContextDef_t` struct filled by `cg_main.c`) holds function pointers (`fnDrawText`, `fnDrawPic`, etc.) that cgame provides to the UI system. This allows the UI system (statically linked or QVM) to call back into cgame without a reverse link dependency.

**4. State Reuse Pattern**: Heavy reliance on precomputed state in `cg_draw.c` (`sortedTeamPlayers`, chat buffers) rather than cgame recomputing it here. This is a performance hack: sorting players once per frame in cg_draw.c is cheaper than redoing it for every owner-draw call.

**5. Lerp-Based Animation**: Head portrait damage reaction uses classic state machines: `headStartYaw` → `headEndYaw` over `[headStartTime, headEndTime]` with smooth-step lerp (`frac * frac * (3 - 2*frac)`). This avoids event-driven systems; the animation state lives in the cgame_t struct.

## Data Flow Through This File

**Input Flow**:
1. User presses mouse/key → `CG_MouseEvent` / `CG_KeyEvent` 
2. Routes to `Display_MouseMove` or `Display_HandleKey` (in `ui_shared.c`)
3. UI framework updates menu/widget state (e.g., `cg_currentSelectedPlayer` cvar)
4. `cgs.orderPending` flag is set if a team command needs issuing

**Render Flow**:
1. UI calls `CG_OwnerDraw(ownerDraw=OD_TEAMMATE_1_HEALTH, ...)`
2. Dispatch switch selects `case OD_TEAMMATE_1_HEALTH`
3. Call `CG_DrawSelectedPlayerHealth(rect, scale, color, shader, style)`
4. Read from `sortedTeamPlayers[CG_GetSelectedPlayer()]` → clientInfo
5. Call `CG_Text_Paint` or `CG_DrawPic` → renderer syscall

**Team State Flow**:
1. Server sends snapshot with entity deltas
2. `CG_ParseTeamInfo` (cg_draw.c) sorts players into `sortedTeamPlayers[]`
3. Team overlay functions read from this sorted array each frame (no caching)

## Learning Notes

**Idiomatic to Q3A / Early 2000s**:
- **Immediate-mode rendering**: No retained widget tree; everything redrawn every frame. Modern engines (Unreal, Unity) use scene graphs or data-driven UI frameworks.
- **Owner-draw callback dispatch**: Replaces modern data binding (UI elements describe *what* to show; engine provides *how*). In modern Qt/web/game engines, the UI framework would directly read game state.
- **No widget state machine**: Widget behavior is baked into rendering logic. Modern engines would have widget classes with lifecycle events.
- **Color code parsing inline**: `Q_IsColorString` embedded in text painting. Modern UI engines use font rasterization + color palettes.

**Connections to Engine Design**:
- **Snapshot-driven**: Every HUD element reads from a server-provided snapshot (`cg.snap`). This enforces determinism and replay-ability.
- **CVar-mediated selection**: Player selection is stored in `cg_currentSelectedPlayer` cvar, allowing UI to persist it and console scripts to set it.
- **Tight UI↔Game coupling**: MissionPack UI cannot exist without cgame HUD extensions; they're co-designed.

**Architecture Lesson**:
This file demonstrates why Q3 required a *modular UI VM separate from cgame*: the UI system is generic (buttons, sliders, menus), while owner-draw callbacks are game-specific (health bars, team overlays, medals). By splitting them, the engine could reuse the UI framework across game mods while letting each mod define custom HUD logic in cgame.

## Potential Issues

1. **Dead Code / Inverted Guard** (line ~230): `healthColors` is defined under `#ifndef MISSIONPACK`, but this file *only* compiles in MISSIONPACK builds. The array is never used. Should be removed or guarded as `#ifdef MISSIONPACK`.

2. **Commented-Out Visibility Check** (in `CG_OwnerDraw`): The line checking `ownerDrawFlags` against element visibility is commented out. Unclear if this is intentional (debug?) or a regression.

3. **Unchecked Array Access** (e.g., line ~400): `sortedTeamPlayers[cg_currentSelectedPlayer.integer]` is accessed without bounds-checking in some functions (though `CG_GetSelectedPlayer()` clamps in others). Risk of out-of-bounds read if `numSortedTeamPlayers` changes mid-frame.

4. **Global State Mutation** (team chat, spectator scroll state): Multiple static variables (`cg.spectatorOffset`, `cg.spectatorTime`) are mutated during rendering. This breaks thread safety if SMP rendering is enabled (Q3 does support it). Should use atomic updates or per-thread locals.

5. **String Buffer Overflow**: `CG_CheckOrderPending` calls `va()` without bounds validation on the resulting command string. If team task enum is malformed, could overflow.
