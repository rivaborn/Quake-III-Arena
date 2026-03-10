# code/q3_ui/ui_startserver.c — Enhanced Analysis

## Architectural Role

This file implements the **server launch configuration pipeline** in the UI VM, bridging player intent (map choice, game options, bot rosters) to engine execution. It sits at a critical point in the client→server launch pathway: after the player decides to start a game, this code translates that decision into a set of cvars and console commands that drive the actual server initialization. The three menus form a **linear wizard flow** (`map selection` → `server options` → `bot picker`), using the modal menu stack to ensure the player cannot skip or reorder steps.

## Key Cross-References

### Incoming (who depends on this file)
- **Public entry points**: `UI_StartServerMenu()`, `UI_ServerOptionsMenu()`, `UI_BotSelectMenu()` called by the main menu system (in `ui_main.c`) when user selects "Start Server"
- **Menu framework routing**: `Menu_Draw()` and `Menu_HandleKey()` (in `ui_qmenu.c`) dispatch input events to the event handlers (`StartServer_GametypeEvent`, `StartServer_MenuEvent`, etc.)
- **Modal menu stack**: Called via `UI_PushMenu()` / `UI_PopMenu()` for wizard-style linear flow

### Outgoing (what this file depends on)
- **Trap syscalls**: 
  - `trap_R_RegisterShaderNoMip()` (shader caching)
  - `trap_Cvar_SetValue()`, `trap_Cvar_Set()` (cvar control)
  - `trap_Cvar_VariableValue()`, `trap_Cvar_VariableStringBuffer()` (cvar query)
  - `trap_Cmd_ExecuteText(EXEC_APPEND)` (launch server via command buffer)
- **Engine info queries** (from other UI modules):
  - `UI_GetNumArenas()`, `UI_GetArenaInfoByNumber()`, `UI_GetArenaInfoByMap()` — level metadata
  - `UI_GetNumBots()`, `UI_GetBotInfoByNumber()`, `UI_GetBotInfoByName()` — bot roster
  - `UI_GetBotModelSkin()` — bot portrait asset names
- **Shared utilities** (`code/qcommon` and `code/game`):
  - `COM_ParseExt()`, `Info_ValueForKey()`, `Q_stricmp()`, `Q_strncpyz()`, `Q_strupr()` — text parsing
  - `Com_sprintf()`, `Com_Clamp()` — formatting and bounds
  - `atoi()` — string-to-int conversion
- **Renderer draw primitives** (via ownerdraw callbacks):
  - `UI_DrawHandlePic()`, `UI_DrawString()`, `UI_FillRect()` — 2D drawing (called from `StartServer_LevelshotDraw()`)

## Design Patterns & Rationale

### Wizard-Flow Modal Stack
Three interconnected menus form a **linear progression** rather than a hierarchical tree:
- **Map Selection** (`s_startserver`) → **Options** (`s_serveroptions`) → **Bot Select** (per-slot `botSelectInfo`)
- Each menu is fully modal (`UI_PushMenu` prevents access to previous menu)
- **Why**: Ensures players can't accidentally skip configuration steps; simplifies state management (no cross-menu conflicts)
- **Trade-off**: Less flexible than a tabbed interface, but faster to implement and less error-prone for late-90s hardware

### Event-Driven Widget Updates
`StartServer_Update()` is called only on *state-changing events* (gametype change, page flip, map select), not every frame:
- Updates `QMF_HIGHLIGHT`, `QMF_INACTIVE`, `QMF_PULSEIFFOCUS` flags on map buttons/pictures
- Updates the map name label
- Renderer checks these flags during `Menu_Draw()` and renders conditionally
- **Why**: Avoids redundant computation; game engine runs at 60+ FPS, but map list rarely changes
- **Pattern**: Common in pre-2000s UI frameworks that couldn't afford per-frame callbacks

### Filter-by-Gametype via Loop
`StartServer_GametypeEvent()` rebuilds the entire filtered map list each time gametype changes:
- Loops through `UI_GetNumArenas()` (typically <64 arenas)
- Uses `GametypeBits()` to parse arena `type` field and check against selected gametype bitmask
- Recalculates `maxpages`, resets `page` and `currentmap` to 0
- **Why**: Simple, brute-force approach; no need for reverse indices or caching
- **Trade-off**: O(N) per gametype change, but N is small and operation is rare (player changes gametype once per session)

### Deferred Bot Name Copying
`ServerOptions_LevelshotDraw()` handles the actual slot-name update when `newBot` flag is set:
- `UI_BotSelectMenu_SelectEvent()` pops the menu and sets `newBot = qtrue` **without** copying the name
- On the next rendered frame, `ServerOptions_LevelshotDraw()` detects the flag and copies `newBotName` into the player slot
- **Why**: Avoids race conditions with menu stack manipulation and popup/popdown timing
- **Idiomatic pattern**: Q3A menus often defer side effects to the render phase to maintain modal stack integrity

## Data Flow Through This File

```
[ Player clicks "Start Server" ]
    ↓
UI_StartServerMenu()
    ├─ StartServer_MenuInit(): populate all widgets
    ├─ StartServer_GametypeEvent(): build filtered map list for default gametype
    └─ Menu pushed to modal stack
    
[ Each frame ]
    ├─ Menu_Draw() → StartServer_LevelshotDraw() renders thumbnails, map name
    └─ Menu_HandleKey() routes input
    
[ Player selects map ]
    → StartServer_MapEvent() updates currentmap
    → StartServer_Update() syncs widget flags
    
[ Player clicks "Next" (Start Server Options) ]
    → StartServer_MenuEvent(ID_STARTSERVERNEXT)
    → UI_ServerOptionsMenu() pushed
    
[ Player configures options, adds bots ]
    → Each bot slot click → UI_BotSelectMenu() pushed (modal)
       ├─ User selects bot
       → UI_BotSelectMenu_SelectEvent() sets newBot=true, pops menu
       ← On next frame, ServerOptions_LevelshotDraw() copies name
    
[ Player clicks "Go" ]
    → ServerOptions_Start()
    ├─ Reads all UI state (options, bot selections)
    ├─ Sets cvars: sv_maxclients, timelimit, dedicated, etc.
    └─ Executes: "dedicated 1 ; wait ; map q3dm1 ; addbot Doom ; team red ; addbot Anarki ; team blue ; ..."
    
[ Server initializes and game begins ]
```

The flow is **purely command-based**: the UI builds a sequence of console commands that the engine's command buffer eventually executes. This mirrors Q3A's architecture where **the console is the universal control plane**.

## Learning Notes

### Idiomatic Q3A UI Patterns
1. **Modal menu stack** — Linear flow is enforced by modal overlay (much simpler than event publishing or state machines)
2. **Trap syscalls only** — UI VM never calls engine functions directly; all access is sandboxed through numbered syscall indices
3. **Event-driven, not tick-driven** — Menus don't have a per-frame `update()` hook; they react to user input and explicit events
4. **Flag-driven rendering** — Widgets carry `QMF_*` bitflags; the menu renderer consults them each frame
5. **File-static globals** — All per-menu state is `static` to this file; enables closure-like behavior in callbacks

### Why Q3A Design Differs from Modern Engines
- **No dynamic allocation for UI** — Everything allocated at init; menus are pre-sized (32 button slots, 64 map slots, etc.)
- **No per-frame callback** — Modern engines might call `onUpdate()` every frame; Q3A does it only on events
- **Command-based architecture** — Server launch is just a **sequence of text commands** that can be recorded, debugged in console, or played back (critical for demos and mods)
- **VM sandbox** — UI runs as untrusted QVM bytecode; engine must mediate all access via `trap_*` syscall ABI

### Architectural Insights
1. **Separation of concerns**: UI layer knows *nothing* about how cvars/game logic work—it just names them and passes values
2. **Late binding**: Server launch is **not a function call**; it's a command string. This allows:
   - Mods to override server initialization via `.cfg` files and console scripts
   - Demos to replay the exact server launch sequence
   - Remote console to send the same commands without the UI
3. **Pagination is stateful but simple**: Page number + item count determine visible region; no iterator objects or callbacks needed

## Potential Issues

### Buffer Overflows (Medium Risk)
- `strcpy(s_startserver.mapname.string, ...)` assumes buffer is large enough — vulnerable if a map name exceeds the string buffer size
- `strcpy(s_serveroptions.newBotName, ...)` — same issue
- **Mitigation**: Codebase has `Q_strncpyz()` available (used elsewhere), but not used consistently here
- **Modern fix**: Replace all `strcpy()` with `Q_strncpyz(..., size)`

### Unchecked User Input (Low Risk)
- Hostname field (`s_serveroptions.hostname`) accepts arbitrary input, which is then executed as a cvar value
- If hostname contains quotes, newlines, or semicolons, it could break the command line or execute unintended commands
- **Example**: Hostname `"; rcon_password hacked` could inject console commands
- **Mitigation**: Validate/quote the hostname before passing it to the command buffer

### Hardcoded Limits (Low-Medium Risk)
- `MAX_SERVERMAPS = 64` — if a mod has >64 maps, extras are silently dropped
- `MAX_NAMELENGTH = 16` — map names longer than 16 chars are truncated
- `MAX_BOTS = 64` — if a mod includes >64 bots, extras are dropped
- **Mitigation**: These limits are appropriate for late-90s hardware; modern Q3A mods rarely exceed them, but documentation should note the caps

### Missing Levelshot Fallback
- If `GAMESERVER_UNKNOWNMAP` shader fails to register, rendering will try to display a null shader
- **Impact**: Unlikely in practice; shaders are pre-cached during `StartServer_Cache()`
- **Modern fix**: Wrap in error callback or pre-validate all critical shaders at startup

### Pagination Math Edge Case
- `maxpages = (nummaps + MAX_MAPSPERPAGE - 1) / MAX_MAPSPERPAGE` — correct ceiling division
- But if `nummaps = 0`, `maxpages = 0`, `page = 0` is still valid (renders "NO MAPS FOUND")
- **Status**: Not actually a bug; the design handles it gracefully

---

## Summary

This file exemplifies **late-90s game UI architecture**: modal menu stacks, event-driven updates, command-based server launch, and memory-efficient static allocation. Its wizard-style flow and deferred-update pattern are idiomatic to Q3A's sandbox UI VM design. The code prioritizes **simplicity and determinism** over flexibility—a reasonable trade-off for the era's hardware constraints and the engine's emphasis on reproducible gameplay (demos, LAN servers, mods).
