# code/ui/keycodes.h
## File Purpose
Defines the canonical enumeration of all input key codes used by the Quake III Arena UI and input systems. It provides a hardware-agnostic numeric identity for every bindable input, including keyboard keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs.

## Core Responsibilities
- Define `keyNum_t`, the master enum of all recognized input identifiers
- Anchor ASCII-compatible keys at their literal ASCII values (Tab=9, Enter=13, Escape=27, Space=32)
- Enumerate extended keys (function keys, numpad, arrows, modifiers) starting at 128
- Enumerate mouse, scroll wheel, joystick (32 buttons), and auxiliary (16) inputs
- Define `K_CHAR_FLAG` bitmask to distinguish character events from key events in the menu system
- Assert via comment that `K_LAST_KEY` must remain below 256

## External Dependencies
- No includes. Self-contained.
- `keyNum_t` values are consumed by:
  - `KeyEvent()` — defined elsewhere in the client/input layer
  - Key-binding tables in `cl_keys.c`
  - Menu input handlers in `ui_main.c` / `ui_atoms.c`

# code/ui/ui_atoms.c
## File Purpose
Provides foundational UI utility functions for the Quake III Arena menu system, including drawing primitives, coordinate scaling, console command dispatch, and post-game score tracking/persistence.

## Core Responsibilities
- Bridges `q_shared.c` error/print functions to UI trap calls (when not hard-linked)
- Scales 640×480 virtual coordinates to actual screen resolution
- Dispatches UI console commands (`postgame`, `ui_cache`, `remapShader`, etc.)
- Persists and loads per-map post-game best scores to/from `.game` files
- Provides primitive 2D drawing helpers (filled rects, outlines, named/handle pics)
- Manages the `m_entersound` flag for menu interaction audio

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`
- `trap_*` functions — all defined in `ui_syscalls.c`; bridge to engine VM syscalls
- `uiInfo` (global `uiInfo_t`) — defined in `ui_main.c`
- `Display_CacheAll` — defined in `ui_shared.c`
- `UI_ShowPostGame`, `UI_Report`, `UI_Load` — defined in `ui_main.c`
- `Com_sprintf`, `Q_strncpyz`, `Q_stricmp`, `Info_ValueForKey` — defined in `q_shared.c`

# code/ui/ui_gameinfo.c
## File Purpose
Loads and parses arena map and bot definition files (`.arena`, `.bot`, `arenas.txt`, `bots.txt`) into global UI-accessible arrays. Provides lookup functions for bot info strings and populates `uiInfo.mapList` with parsed arena metadata used by the UI menu system.

## Core Responsibilities
- Parse key-value info blocks from arena/bot text files via `UI_ParseInfos`
- Load all arena definitions from `scripts/arenas.txt` and `*.arena` files into `ui_arenaInfos[]`
- Load all bot definitions from `scripts/bots.txt` and `*.bot` files into `ui_botInfos[]`
- Populate `uiInfo.mapList[]` with map name, load name, image path, and game-type bitfields
- Provide bot lookup by index or name
- Respect `g_arenasFile` / `g_botsFile` cvars to override default paths

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `COM_Parse`, `COM_ParseExt`, `COM_Compress`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `UI_Alloc`, `UI_OutOfMemory`, `String_Alloc`, all `trap_*` syscall wrappers, `uiInfo` global, `MAX_BOTS`, `MAX_ARENAS`, `MAX_ARENAS_TEXT`, `MAX_BOTS_TEXT`, `MAX_MAPS`, game-type enum constants (`GT_FFA`, etc.)

# code/ui/ui_local.h
## File Purpose
This is the primary internal header for the Quake III Arena UI VM module. It aggregates all type definitions, constants, extern declarations, and trap (syscall) function prototypes needed by the UI subsystem's implementation files.

## Core Responsibilities
- Declares all `vmCvar_t` globals used across UI screens (game rules, server browser, scores, etc.)
- Defines the legacy `menuframework_s` / `menucommon_s` widget type system and associated flags
- Declares the `uiStatic_t` singleton holding frame-level UI state and asset handles
- Declares the large `uiInfo_t` aggregate holding all new-UI runtime state (server lists, maps, tiers, players, mods, demos)
- Provides the complete `trap_*` syscall interface the UI VM uses to call into the engine
- Forward-declares every UI screen module's public cache/init/display functions

## External Dependencies
- `../game/q_shared.h` — base types (`vec3_t`, `qboolean`, `vmCvar_t`, etc.)
- `../cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, etc.)
- `ui_public.h` — exported UI entry point enum (`uiExport_t`), `uiMenuCommand_t`, `uiClientState_t`
- `keycodes.h` — `keyNum_t` enum
- `../game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, game type enums
- `ui_shared.h` — new-UI `displayContextDef_t` and shared menu-def types (defined elsewhere)
- Engine syscall dispatch — all `trap_*` targets are defined in the engine, not this module

# code/ui/ui_main.c
## File Purpose
The primary entry point and master controller for Quake III Arena's Team Arena UI module. It implements the `vmMain` dispatch function (the QVM entry point), manages all menu data, handles owner-draw rendering, input routing, server browser logic, and asset lifecycle for the entire UI system.

## Core Responsibilities
- Dispatch all UI VM commands via `vmMain` (init, shutdown, key/mouse events, refresh, active menu)
- Initialize and wire the `displayContextDef_t` function table with UI callbacks during `_UI_Init`
- Render per-frame UI: paint menus, draw cursor, update server/player lists via `_UI_Refresh`
- Implement all owner-draw items (handicap, player model, clan logo, map preview, team slots, etc.)
- Manage server browser: refresh, display list construction, binary insertion sorting, find-player searches
- Parse game data files: `gameinfo.txt`, `teaminfo.txt`, map lists, game types, character/alias tables
- Register and update all UI cvars through a static `cvarTable[]` descriptor array
- Execute menu scripts (`UI_RunMenuScript`) for game start, server join, bot add, settings changes

## External Dependencies
- `ui_local.h` → `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `ui_shared.h`, `keycodes.h`
- **Defined elsewhere:** `Menu_Count`, `Menu_PaintAll`, `Menu_GetFocused`, `Menu_HandleKey`, `Menu_New`, `Menu_Reset`, `Menu_SetFeederSelection`, `Menus_*`, `Display_*`, `Init_Display`, `String_*`, `Controls_*`, `UI_DrawPlayer`, `UI_PlayerInfo_*`, `UI_RegisterClientModelname`, `UI_LoadArenas`, `UI_LoadBestScores`, `UI_ClearScores`, `UI_LoadBots`, `UI_GetBotNameByNumber`, `UI_GetNumBots`, `trap_*` syscalls (all defined in `ui_syscalls.c`)

# code/ui/ui_players.c
## File Purpose
Handles 3D player model rendering and animation state management for the Quake III Arena UI. Provides the `UI_DrawPlayer` function used to display animated player characters in menus (character selection, player settings, etc.), along with model/skin/animation loading utilities.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate animation frame data
- Drive per-frame animation state machines for legs and torso (idle, jump, land, attack, drop/raise weapon)
- Compute hierarchical entity positioning via tag attachment (torso→legs, head→torso, weapon→torso, barrel→weapon)
- Calculate smooth angle transitions (yaw swing, pitch) for the displayed model
- Issue renderer calls to assemble and submit the full player scene each UI frame
- Manage weapon switching sequencing with audio cue

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `uiInfo` (global UI context, provides `uiDC.frameTime`); `bg_itemlist` (item/weapon definitions from `bg_misc.c`); all `trap_R_*`, `trap_CM_LerpTag`, `trap_S_*`, `trap_FS_*` syscall wrappers; math utilities (`AnglesToAxis`, `MatrixMultiply`, `AngleSubtract`, `AngleMod`, etc.); animation enum constants (`LEGS_JUMP`, `TORSO_ATTACK`, etc.)

# code/ui/ui_public.h
## File Purpose
Defines the public ABI contract between the Quake III Arena engine and the UI dynamic module (VM). It enumerates all syscall trap numbers the UI module uses to call into the engine (`uiImport_t`) and all entry points the engine calls on the UI module (`uiExport_t`).

## Core Responsibilities
- Declare the UI API version constant for compatibility checks
- Define `uiClientState_t` to carry connection/server state to the UI
- Enumerate all engine→UI import syscalls (`uiImport_t`)
- Enumerate all UI→engine export entry points (`uiExport_t`)
- Define `uiMenuCommand_t` for identifying which menu to activate
- Define server-list sort-order constants

## External Dependencies
- `connstate_t` — defined in engine connection-state headers (e.g., `client.h`)
- `MAX_STRING_CHARS` — defined in `q_shared.h`
- No includes are present in this header; consumers must include prerequisite headers before this file

# code/ui/ui_shared.c
## File Purpose
This is the shared UI framework implementation for Quake III Arena, providing the complete runtime for a data-driven menu system. It handles menu/item parsing from script files, rendering of all widget types, input routing (mouse, keyboard, key binding), and memory management for UI resources. It is shared between the `ui` and `cgame` modules via conditional compilation.

## Core Responsibilities
- Fixed-pool memory allocation and interned string storage for UI data
- Parsing menu/item definitions from PC (parser context) token streams using keyword hash tables
- Painting all window and item types (text, listbox, slider, model, bind, ownerdraw, etc.)
- Routing keyboard and mouse input to the focused menu/item
- Managing key bindings (read, write, defaults) via the `g_bindings` table
- Scripting: tokenizing and dispatching `commandList` scripts attached to items/menus
- Managing menu focus stack, visibility, transitions, orbiting, and fade effects

## External Dependencies
- `ui_shared.h` → `q_shared.h`, `tr_types.h`, `keycodes.h`, `menudef.h`
- `trap_PC_ReadToken`, `trap_PC_SourceFileAndLine`, `trap_PC_LoadSource` — defined in platform-specific syscall stubs
- `COM_ParseExt`, `Q_stricmp`, `Q_strcat`, `Q_strupr` — defined in `q_shared.c`
- `Com_Printf` — engine print, defined elsewhere
- `AxisClear`, `AnglesToAxis`, `VectorSet`, `VectorCopy` — math, defined in `q_math.c`
- All `DC->*` function pointers — resolved at runtime via `Init_Display`

# code/ui/ui_shared.h
## File Purpose
Defines the complete shared data model and public API for the Quake III Arena UI system, used by both the `ui` and `cgame` modules. It declares all menu/item/window types, the display context vtable, cached assets, and the full set of functions for menu lifecycle management and rendering.

## Core Responsibilities
- Define all UI structural types: `windowDef_t`, `itemDef_t`, `menuDef_t`, and their sub-types
- Declare the `displayContextDef_t` vtable that abstracts all renderer/engine calls away from UI code
- Declare `cachedAssets_t` for shared UI texture/font/sound handles
- Define window state flag bitmasks (WINDOW_*, CURSOR_*)
- Declare the full public API surface for menu/display management (init, paint, input, feeder, etc.)
- Define string pool constants and management API (`String_Alloc`, `String_Init`)
- Declare parser helpers for both text (`Float_Parse`, etc.) and PC (preprocessed script) token streams

## External Dependencies
- `../game/q_shared.h` — `vec4_t`, `qboolean`, `qhandle_t`, `sfxHandle_t`, `fontInfo_t`, `glconfig_t`, `refEntity_t`, `refdef_t`, `pc_token_t`
- `../cgame/tr_types.h` — `refEntity_t`, `refdef_t`, `glconfig_t`
- `keycodes.h` — `keyNum_t` enum
- `../../ui/menudef.h` — `ITEM_TYPE_*`, `FEEDER_*`, `CG_SHOW_*`, owner-draw constants
- `trap_PC_*` functions — defined elsewhere in the VM syscall table (`ui_syscalls.c` / `cg_syscalls.c`)
- `UI_Alloc` / `UI_InitMemory` / `UI_OutOfMemory` — VM-local memory pool, defined in `ui_main.c`
- `Controls_GetConfig` / `Controls_SetConfig` / `Controls_SetDefaults` — defined in `ui_shared.c`

# code/ui/ui_syscalls.c
## File Purpose
Provides the DLL-side system call bridge for the UI module, mapping high-level `trap_*` functions to indexed engine syscalls via a single function pointer. This file is only compiled for DLL builds; the QVM equivalent is `ui_syscalls.asm`.

## Core Responsibilities
- Store and initialize the engine-provided `syscall` function pointer via `dllEntry`
- Wrap every engine service (rendering, sound, cvars, filesystem, networking, input, cinematics) behind typed `trap_*` C functions
- Handle float-to-int reinterpretation via `PASSFLOAT` to safely pass floats through the variadic integer syscall ABI
- Expose CD-key validation and PunkBuster status reporting to the UI module
- Provide LAN/server browser query traps for the multiplayer server list UI

## External Dependencies
- `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`)
- `UI_*` syscall index constants — defined in `ui_public.h` (not in this file)
- All type definitions (`vmCvar_t`, `refEntity_t`, `glconfig_t`, `qtime_t`, `e_status`, `fontInfo_t`, etc.) — defined elsewhere in shared/game/renderer headers
- `QDECL` calling convention macro — defined in `q_shared.h`

# code/ui/ui_util.c
## File Purpose
A stub or placeholder utility file for the Quake III Arena UI module, intended to house memory and string allocation helpers for the new UI system. The file contains no implemented functions — only a license header and comment annotations.

## Core Responsibilities
- Reserved as the location for UI utility functions (memory, string allocation)
- No active responsibilities in current state; file is empty beyond the header

## External Dependencies
- None declared. No `#include` directives are present.
- Comment attributes (`origin: rad`) suggest authorship from the "rad" (RAD Game Tools / id internal) development context.

---

> **Note:** This file is effectively empty. The header comment indicates it was scaffolded to hold UI utility code (memory and string allocation), but no implementation was committed. Any actual utility functions intended for this file likely reside in `code/ui/ui_shared.c` or `code/ui/ui_atoms.c`.

