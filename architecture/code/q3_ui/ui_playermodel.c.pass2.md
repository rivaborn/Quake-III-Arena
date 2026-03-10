# code/q3_ui/ui_playermodel.c — Enhanced Analysis

## Architectural Role

This file is the **UI-facing entry point for player model/skin selection** in the legacy Q3A UI subsystem. It bridges three distinct architectural layers: the UI VM (QVM bytecode sandbox), the renderer (via `trap_R_*` syscalls), and the persistent CVar system (model/headmodel/team_model/team_headmodel). The file encapsulates the complete player model discovery, preview rendering, and persistence flow—a self-contained menu module that does not directly interact with the cgame VM or game logic, only with the engine's public service boundaries.

## Key Cross-References

### Incoming (who depends on this file)
- **UI VM bootstrap** (`ui_main.c`, `ui_syscalls.c`): Calls `UI_PlayerModelMenu()` to activate this menu (typically from main menu or connection flow)
- **Menu framework** (`ui_qmenu.c`): Routes keyboard/mouse events and frame draws through the menu stack to this file's `PlayerModel_MenuKey`, `PlayerModel_MenuEvent`, `PlayerModel_PicEvent`, and `PlayerModel_DrawPlayer` callbacks

### Outgoing (what this file depends on)
- **Renderer** (`trap_R_RegisterShaderNoMip`, `trap_R_DrawStretchPic`): Precaches UI art shaders and renders bitmap widgets via the menu framework
- **UI Player Renderer** (`ui_players.c`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_DrawPlayer`): Manages the 3D player model lifecycle and per-frame skeletal animation rendering—called conditionally when memory allows
- **Virtual Filesystem** (`trap_FS_GetFileList`): Discovers all `models/players/*/icon_*.tga` files at menu init; also loads texture assets for player models
- **CVars** (`trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer`, `trap_Cvar_VariableValue`): Reads current model/name CVars on init; writes selected model to four CVars (model, headmodel, team_model, team_headmodel) on exit; checks `com_buildscript` flag for sound precaching
- **Sound** (`trap_S_RegisterSound`): Conditionally precaches `sound/player/announce/{skinname}_wins.wav` if `com_buildscript` is enabled (server build mode)
- **Memory Query** (`trap_MemoryRemaining`): Guards the expensive 3D player preview behind a 5 MB threshold

## Design Patterns & Rationale

**1. Sandbox Isolation via Syscall Boundaries**
All I/O to renderer, filesystem, sound, and cvar subsystems flows through `trap_*` syscalls. The UI code cannot link directly to engine symbols. This enforces a security boundary and allows the UI to be updated independently (e.g., via mod replacement or separate DLL).

**2. Precaching as Latency Hiding**
`PlayerModel_Cache()` precaches all UI art shaders at menu init (via `PlayerModel_BuildList`). This avoids hitches during grid navigation. Shaders are compiled once and cached in the renderer; subsequent page turns are O(1) assignments of cached shader handles to bitmap widgets.

**3. Owner-Draw Callback Pattern**
`PlayerModel_DrawPlayer` is an owner-draw hook invoked each frame by the menu framework. Rather than a full-screen render, it's a single bitmap widget that displays 3D content. The renderer's internal `Tess` pipeline is reused; no separate backbuffer or context switch is needed. This is idiomatic to the Q3A era (pre-shader-driven UI).

**4. Memory-Guarded Expensive Operations**
Both `PlayerModel_PicEvent()` (model selection) and `PlayerModel_DrawPlayer()` (per-frame rendering) check `trap_MemoryRemaining()` before invoking 3D player setup/rendering. If < 5 MB free, the preview is disabled and a warning string is shown instead. This prevents OOM crashes on console or low-memory systems (e.g., dedicated servers or old PCs).

**5. CVar as Persistent State Boundary**
The selected modelskin is not stored in UI module state; it is written to four CVars on menu exit (`PlayerModel_SaveChanges`). This ensures the selection persists across game restarts, and makes it visible to the cgame and game modules, which read these CVars during player initialization. The four CVars (model, headmodel, team_model, team_headmodel) suggest a **team color system** where team-specific shader paths can override the base model.

**6. Asset Naming Convention Embedded in UI**
The parsing of `icon_` from model paths (`strstr(buffptr,"icon_")`) and subsequent split of model/skin name is hardcoded. This couples the UI to the asset directory structure (`models/players/[modeldir]/icon_[skinname].tga`). The convention is not negotiable—it's the contract between the asset pipeline and the UI layer.

## Data Flow Through This File

```
INIT SEQUENCE:
  UI_PlayerModelMenu()
    ↓
  PlayerModel_MenuInit()
    ↓
  PlayerModel_Cache()  [precache all UI art shaders]
    ↓
  PlayerModel_BuildList()  [scan models/players/* → populate modelnames[], compute numpages]
    ↓
  PlayerModel_SetMenuItems()  [read current model CVar, find it in list, set selectedmodel/modelpage]
    ↓
  PlayerModel_UpdateGrid()  [populate pics[]/picbuttons[] for current page]
    ↓
  PlayerModel_UpdateModel()  [initialize playerinfo with current modelskin]

PER-FRAME (during menu active):
  Menu_Draw() [invokes owner-draw callbacks]
    ↓
  PlayerModel_DrawPlayer()  [check memory, call UI_DrawPlayer() if ok, else show warning]

USER INPUT (keyboard/mouse):
  PlayerModel_MenuKey()  [arrow keys → grid navigation with auto page-turn on edges]
    ↓
  PlayerModel_MenuEvent()  [page buttons → adjust modelpage, update grid]
    ↓
  PlayerModel_PicEvent()  [portrait button click → update selection, parse model/skin, conditional PlayerModel_UpdateModel()]

EXIT:
  PlayerModel_SaveChanges()  [write modelskin to four CVars]
    ↓
  UI_PopMenu()
```

**Key State Mutations:**
- `s_playermodel.modelnames[]` ← populated from filesystem during init (read-only after)
- `s_playermodel.selectedmodel` ← set during `PlayerModel_SetMenuItems`, updated on portrait click
- `s_playermodel.modelskin` ← derived from modelnames parse, written to CVars on exit
- `s_playermodel.playerinfo` ← re-initialized each time a portrait is clicked (if memory allows)
- `s_playermodel.modelpage` ← adjusted by prev/next page buttons, affects which portrait names appear in `pics[]`

## Learning Notes

**1. Q3A UI Architecture (Late 1990s Conventions)**
- Fixed virtual coordinate space (640×480) rather than resolution-independent scaling. The renderer handles the final scaling.
- Paginated grids rather than scrollable lists. Fixed 4×4 layout is simple to position and animate.
- Owner-draw callbacks for 3D content—the UI menu system doesn't know about 3D rendering; it just invokes per-item draw callbacks.
- Heavyweight precaching at menu init; minimal per-frame logic.

**2. Syscall-Based Sandboxing**
Every subsystem (renderer, sound, filesystem, cvar) is accessed through syscall indirection. This was a radical design in 1999—today's engines embed systems directly. The sandbox prevented malicious mods from direct hardware access or breaking networked game state.

**3. Multi-CVar Persistence Pattern**
Four CVars for "the player model" suggests a legacy design where base Q3A and MissionPack/Team Arena used different naming conventions. Writing all four ensures compatibility across game modes without mode-specific code in the UI.

**4. Memory Budgeting in UI**
The 5 MB threshold is conservative—a modern GPU easily handles a single 3D model in a menu. This reflects console/handheld constraints of the era. The per-frame check (not just init check) shows awareness that memory pressure can spike during gameplay if the user navigates the menu in-game.

**5. Shader Handle Reuse**
Each portrait bitmap widget caches a `shader` member (integer handle). Once `trap_R_RegisterShaderNoMip` is called, the handle is static; subsequent pages just reassign the shader handle to the correct `pics[i].shader`. The renderer's internal cache ensures no recompilation.

**6. Contrast with Modern UI**
A modern engine (e.g., Unreal, Unity) would:
- Use a data-driven layout (JSON/YAML mesh, not hardcoded coordinates)
- Stream textures/models on demand, not precache
- Use a resolution-independent coordinate system
- Render 3D content via full render target, not owner-draw
- Store state in a backend database, not persistent CVars

## Potential Issues

1. **Off-by-One in Com_sprintf** (line ~432, in PlayerModel_BuildList):
   ```c
   Com_sprintf( s_playermodel.modelnames[s_playermodel.nummodels++],
     sizeof( s_playermodel.modelnames[s_playermodel.nummodels] ),
     ...
   ```
   The `sizeof` uses `nummodels` (which has just been incremented), so it's always off by one. Should be `nummodels - 1` or increment after the sprintf. Minor bug, unlikely to cause overflow since the array is large (MAX_PLAYERMODELS=256).

2. **Unbounded String Operations**
   `strcat(s_playermodel.modelskin, pdest + 5)` in `PlayerModel_PicEvent` could overflow the 64-byte `modelskin` buffer if the parsed path is long. Uses `Q_strncpyz` elsewhere, but not here. Low risk in practice (model/skin names are usually short), but inconsistent defensive coding.

3. **Memory Guard Not Synchronized with Snapshots**
   The 5 MB check in `PlayerModel_DrawPlayer` is per-frame. If the user enters the menu, then the server sends a large entity update mid-frame, memory could drop and the preview flickers off. The guard is a heuristic, not a hard contract.

4. **No CVar Validation on Load**
   `PlayerModel_SetMenuItems` reads the `model` CVar and searches for an exact match in the built list. If the CVar contains a model that was deleted from the filesystem (or in a different mod), the selection silently defaults to the first model. No warning to the user.
