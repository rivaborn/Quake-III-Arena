# code/ui/ui_main.c — Enhanced Analysis

## Architectural Role

This file is the **primary orchestrator and dispatcher** for Quake III Arena's MissionPack (Team Arena) UI VM module. It acts as the sole engine-facing entry point (`vmMain`) and the master controller for all UI subsystems: menu lifecycle, rendering infrastructure, server browser, game data parsing, and cvar registry. Within the broader architecture, it bridges the engine (client and server processes) to all downstream UI components (cgame/renderer for drawing, qcommon for filesystem/console/collision), while serving as the QVM sandbox boundary. The file initializes the display context function table that enables all owner-draw and text-rendering callbacks throughout the menu system.

## Key Cross-References

### Incoming (who depends on this file)

**Engine entry point:**
- `code/client/cl_ui.c`: Calls `vmMain` dispatcher for all UI commands (init, shutdown, refresh, key/mouse events, active menu, console commands, connect screen)
- `CL_UISystemCalls` (in client.h): Routes all `trap_*` syscalls issued by the UI VM back to engine implementations

**Menu system hierarchy:**
- All functions in `ui_shared.c`, `ui_atoms.c`, `ui_local.h` depend on `uiInfo` global state and `uiInfo.uiDC` display context wired by `_UI_Init`
- `Menu_*` functions (Menu_Count, Menu_PaintAll, Menu_GetFocused) operate on menu structures populated during init

**Game data consumers:**
- cgame reads `ui/menudef.h` constants (owner-draw IDs, feeder IDs, `CG_SHOW_*` visibility flags)
- Server communicates with UI via configstrings (team names, game info)

### Outgoing (what this file depends on)

**Display infrastructure:**
- `Init_Display`: Wires the `displayContextDef_t` vtable; called by `_UI_Init`
- `trap_R_*` syscalls: All shader registration, drawing, text rendering (`trap_R_RegisterShaderNoMip`, `trap_R_DrawStretchPic`, `trap_R_SetColor`)
- Renderer subsystem processes owner-draw paint operations from `UI_OwnerDraw`

**Menu and widget framework:**
- `Menu_New`, `Menu_SetFeederSelection`, `Menu_HandleKey`: Core menu logic (defined in ui_shared.c)
- `UI_BuildQ3Model_List`, `UI_LoadBots`: Player model system for preview rendering
- `UI_LoadMenus`: Script-parser loads `.menu` files into menu tree

**Game data parsing:**
- `trap_FS_FOpenFile`, `trap_FS_Read`: Filesystem access for `gameinfo.txt`, `teaminfo.txt`, map lists
- PC token stream functions: Parse scripts into `uiInfo.gameTypes`, `uiInfo.mapList`, `uiInfo.teamList`

**Server browser:**
- `trap_LAN_LoadCachedServers`, `trap_LAN_SaveCachedServers`: Persistent server cache
- `trap_LAN_GetServerCount`, `trap_LAN_GetServerAddressString`: Query LAN/cached servers
- `trap_LAN_MarkServerVisible`: Filter operations

**Cvar and console:**
- `trap_Cvar_Register`, `trap_Cvar_Set`, `trap_Cvar_VariableValue`: Cvar lifecycle
- `trap_Cmd_ExecuteText`: Execute console commands from menu scripts (StartServer, JoinServer, addBot)

**Audio and cinematics:**
- `trap_S_RegisterSound`: Register UI feedback sounds (new high score)
- `trap_CIN_PlayCinematic`, `trap_CIN_StopCinematic`: Play cinematic sequences (team intros, etc.)

## Design Patterns & Rationale

**1. QVM Dispatcher Pattern**  
`vmMain` switches on a command enum (`UI_INIT`, `UI_REFRESH`, `UI_KEY_EVENT`, etc.), delegating to private `_UI_*` functions. This is the **only way the engine calls into the module**—all UI behavior is demand-driven, never initiated internally.

**2. Display Context Vtable Initialization**  
`_UI_Init` populates `uiInfo.uiDC.drawText`, `uiInfo.uiDC.drawRect`, etc., with function pointers. This enables decoupled rendering: owner-draw code can call generic drawing ops without knowing whether the backend is the software renderer or OpenGL.

**3. Data-Driven Menu Definition**  
Rather than hardcoding menus in C, `ui_main.c` parses scripts (`gameinfo.txt`, `teaminfo.txt`) and `.menu` files at runtime. This allows rapid iteration and mod-friendly configuration without recompilation. The `UI_ParseGameInfo` / `UI_ParseTeamInfo` pipeline converts textual data into the `uiInfo` arrays (gameTypes, mapList, teamList, characterList, aliasList).

**4. Dirty-Flag Optimization**  
Static flags like `updateModel`, `updateOpponentModel`, `q3Model` avoid redundant player model rebuilds each frame. Only when cvars change (e.g., `ui_headmodel`) is `UI_PlayerInfo_SetModel` called to regenerate skeletal state.

**5. Cvar Descriptor Table**  
All UI cvars are registered via a static `cvarTable[]` array, centralizing cvar definitions and enabling batch registration in `UI_RegisterCvars`. This is cleaner than scattered `trap_Cvar_Register` calls.

**6. Owner-Draw Dispatch**  
`UI_OwnerDraw` is a large switch on `ownerDraw` enum, routing to specialized draw functions (e.g., `UI_DrawHandiEp`, `UI_DrawMap`, `UI_DrawTeamModel`). This allows menu scripts to reference draw-time computed values (player preview models, map screenshots) without exposing internal state.

**Rationale for these choices:**
- QVM sandboxing isolates UI from engine crashes
- Data-driven design supports mods and rapid content iteration
- Lazy initialization (dirty flags) reduces per-frame overhead
- Centralized cvar registry prevents state fragmentation
- Owner-draw pattern decouples menu description from complex rendering logic

## Data Flow Through This File

### Initialization Flow
```
vmMain(UI_INIT)
  → _UI_Init(inGameLoad)
    → UI_RegisterCvars()                    // Register all UI cvars
    → trap_GetGlconfig()                    // Query renderer capabilities
    → AssetCache()                          // Load shaders (gradientbar, scrollbar, crosshairs, etc.)
    → Init_Display(uiInfo.uiDC)             // Wire display context vtable
    → String_Init()                         // Initialize string library
    → UI_ParseTeamInfo("ui/teaminfo.txt")   // Parse team definitions
    → UI_LoadTeams()                        // Load team icons/info
    → UI_ParseGameInfo("ui/gameinfo.txt")   // Parse game types, maps
    → UI_LoadMenus()                        // Parse .menu script files
    → trap_LAN_LoadCachedServers()          // Load server cache from disk
    → UI_LoadBestScores()                   // Load high scores
    → UI_BuildQ3Model_List()                // Build player model list
    → UI_LoadBots()                         // Load bot character data
    → Initialize cinematic handles to -1
```

### Per-Frame Render Loop
```
vmMain(UI_REFRESH, realtime)
  → _UI_Refresh(realtime)
    → uiInfo.uiDC.realTime = realtime
    → uiInfo.uiDC.frameTime = realtime - uiInfo.uiDC.realTime (delta)
    → UI_UpdateCvars()                      // Read engine cvar values into vmCvar_t
    → Menu_PaintAll()                       // Recursively paint menu tree
      ├─ For each menu widget:
      │   ├─ UI_OwnerDraw() [if owner-draw item]
      │   └─ Text_Paint() / Text_PaintWithCursor() [text items]
    → UI_DoServerRefresh()                  // Poll LAN servers, refresh list
    → UI_BuildServerStatus(qfalse)          // Update server status window
    → UI_BuildFindPlayerList(qfalse)        // Update find-player search results
    → UI_DrawHandlePic(...)                 // Draw cursor
```

### Input Routing
```
vmMain(UI_KEY_EVENT, key, down)
  → _UI_KeyEvent(key, down)
    → Menu_GetFocused()
    → Menu_HandleKey(focusedMenu, key, down)  // Route to active menu
```

### Server Browser Data Flow
```
UI_BuildServerDisplayList(force)
  → Iterate all cached servers (uiInfo.serverStatus.servers[])
  → Apply filters (game type, mod filter)
  → Binary insertion sort into uiInfo.serverStatus.displayServers[]
  → trap_LAN_MarkServerVisible() [filter visibility bitmap]
```

### Game Data Parsing Pipeline
```
UI_ParseGameInfo("ui/gameinfo.txt")
  → PC_LoadSourceHandle() → lexical tokenization
  → Loop over game type blocks:
    │ ├─ Parse "name", "rules", "gameType" fields
    │ └─ Store in uiInfo.gameTypes[], uiInfo.mapList[]
  → Populate associated Ui lists
```

## Learning Notes

**QVM Sandboxing & Syscall Boundary**  
This file exemplifies the QVM architecture: `code/ui/` compiles to a `.qvm` bytecode file loaded at runtime. All system access (rendering, filesystem, console) goes through indexed `trap_*` syscalls, enabling version-independent module loading. The display context vtable is the primary **abstraction layer** isolating UI logic from renderer implementation details.

**Virtual Resolution & UI Scaling**  
The menu system operates in fixed 640×480 virtual space, with `UI_AdjustFrom640` transforming coordinates to actual screen resolution. This is idiomatic for this era (pre-widescreen); modern engines typically use percentage-based or viewport-relative layouts.

**Bitmap Font Rendering with Glyph Tables**  
`Text_Width`, `Text_Height`, `Text_Paint` functions operate on `fontInfo_t` glyph tables (defined in assets), selecting small/normal/big font variants by scale thresholds. This is a **fixed-function font system**—no TrueType or dynamic font generation. Q3 color escape sequences (`^1`, `^2`, etc.) are handled inline during text painting.

**Owner-Draw Pattern**  
Complex dynamic elements (animated 3D player models, map previews, team slots with health bars) are delegated to `UI_OwnerDraw` rather than being baked into the menu definition. This separates **declarative UI layout** (menu scripts) from **imperative rendering logic** (C code).

**Server Browser Design**  
The server browser implements a full async polling loop: `UI_StartServerRefresh` initiates LAN queries (via `trap_LAN_*`), `UI_DoServerRefresh` polls in-progress queries each frame, and `UI_BuildServerDisplayList` applies filters + sorts the result. This is typical of late-1990s/early-2000s game UI—a far cry from modern REST/JSON APIs.

**Cvar Registry Pattern**  
The static `cvarTable[]` array centralizes all cvar definitions in one place, preventing scattered registrations and making it easy to audit what state the UI controls. This pattern is common in idTech engines.

## Potential Issues

**Server List Binary Search Correctness**  
`UI_BinaryServerInsertion` is called per-server during list building. The sort key depends on `uiInfo.serverStatus.sortKey` (enum 0–4). If a sort key changes at runtime mid-refresh, the list could become partially unsorted. Mitigation: `UI_BuildServerDisplayList(qfalse)` is called every frame, ensuring eventual consistency.

**Hardcoded Array Sizes**  
Many global arrays (maps, teams, characters, servers) likely have fixed `#define` limits (e.g., `MAX_MAPS 1024`). Exceeding these silently truncates data; no bounds checking is visible in the first-pass excerpt. Risk: modders adding too many maps/teams crash the UI VM.

**Cvar Type Safety**  
The `vmCvar_t` struct holds `int` and `string` values. Type mismatches (e.g., reading `ui_smallFont` as a string when it's a float) are not caught at compile time. This relies on runtime discipline and correct usage in menu scripts.

**Memory Lifetime of Parsed Data**  
`UI_ParseGameInfo` / `UI_ParseTeamInfo` populate `uiInfo` arrays via the script parser. If scripts are re-parsed without clearing old data, duplicates or memory leaks could occur. The code relies on `UI_InitMemory` to pre-allocate and clear these arrays once at startup.
