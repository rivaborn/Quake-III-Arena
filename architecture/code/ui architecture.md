# Subsystem Overview

## Purpose
The `code/ui` subsystem implements the QVM-hosted menu and user interface system for Quake III Arena. It provides a data-driven, script-parsed menu framework covering server browsing, player configuration, game setup, post-game scoring, and 3D player model display. It communicates with the engine exclusively through an indexed syscall ABI (`trap_*` wrappers), making it portable across DLL and QVM execution models.

## Key Files

| File | Role |
|---|---|
| `ui_main.c` | `vmMain` QVM entry point; master controller for init, shutdown, frame refresh, input dispatch, owner-draw rendering, server browser, and cvar registration |
| `ui_shared.c` | Data-driven menu framework runtime: script parsing, widget rendering, input routing, key binding management, and scripted command dispatch |
| `ui_shared.h` | Shared type definitions and public API for the menu system; declares `displayContextDef_t` vtable, all widget/menu/window types, and the full menu lifecycle interface |
| `ui_local.h` | Internal aggregation header; declares all UI-internal globals (`uiStatic_t`, `uiInfo_t`), `vmCvar_t` globals, legacy widget type system, and `trap_*` prototypes |
| `ui_public.h` | Public ABI contract: `uiImport_t` (engine→UI syscall indices), `uiExport_t` (UI entry points), `uiClientState_t`, `uiMenuCommand_t` |
| `ui_syscalls.c` | DLL-side syscall bridge; maps all `trap_*` functions to the engine-provided variadic `syscall` pointer via `dllEntry` |
| `ui_atoms.c` | Foundational drawing helpers, virtual-to-screen coordinate scaling, console command dispatch, and post-game best-score persistence |
| `ui_players.c` | 3D player model loading, animation state machines, hierarchical tag attachment, and per-frame renderer submission for menu character display |
| `ui_gameinfo.c` | Parses `.arena`/`.bot`/`arenas.txt`/`bots.txt` files into `ui_arenaInfos[]`, `ui_botInfos[]`, and `uiInfo.mapList[]` |
| `keycodes.h` | Canonical `keyNum_t` enum of all input identifiers; defines `K_CHAR_FLAG` for menu character-vs-key event disambiguation |
| `ui_util.c` | Empty stub; scaffolded for memory/string utilities; no implemented code |

## Core Responsibilities

- Expose a single `vmMain` dispatch function as the QVM entry point, routing all engine-initiated calls (init, shutdown, key event, mouse event, refresh, active-menu query) to internal handlers
- Manage a `displayContextDef_t` function-pointer vtable populated at init time, decoupling all widget rendering and input logic in `ui_shared.c` from direct engine calls
- Parse and evaluate data-driven menu definitions from script files at runtime, supporting text, listbox, slider, model, bind, and owner-draw widget types
- Scale and render 2D UI elements from a virtual 640×480 coordinate space to the actual screen resolution
- Load, animate, and render 3D player models with hierarchical tag attachment (legs→torso→head, weapon→torso) inside menu screens
- Register and maintain all UI-side cvars through a static descriptor table updated each frame
- Implement the server browser: LAN/internet query, response parsing, binary-insertion-sorted display list construction, and find-player search
- Load and parse arena and bot definition files to populate map lists and bot rosters used throughout the menu system
- Persist and restore per-map post-game best scores to `.game` files via filesystem traps

## Key Interfaces & Data Flow

**Exposes to others:**
- `vmMain` — sole engine-visible entry point; engine calls it with `uiExport_t` command identifiers to drive all UI behavior
- `uiExport_t` enum (`ui_public.h`) — documents the complete set of callable entry points (UI_INIT, UI_SHUTDOWN, UI_KEY_EVENT, UI_MOUSE_EVENT, UI_REFRESH, UI_IS_FULLSCREEN, UI_SET_ACTIVE_MENU, UI_CONSOLE_COMMAND, UI_DRAW_CONNECT_SCREEN)
- `uiMenuCommand_t` values — identify which named menu the engine requests to activate

**Consumes from other subsystems:**
- Engine syscalls via `trap_*` wrappers (`ui_syscalls.c`): renderer (`trap_R_*`), sound (`trap_S_*`), cvar (`trap_Cvar_*`), filesystem (`trap_FS_*`), server browser (`trap_LAN_*`), script parser (`trap_PC_*`), input (`trap_Key_*`), cinematic, and network address functions
- `keyNum_t` from `keycodes.h` — consumed by engine input layer (`cl_keys.c`) and menu key-event handlers
- `bg_public.h` types (`weapon_t`, `animation_t`, game-type enums) from the game module
- `tr_types.h` renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`) from the cgame/renderer headers
- `q_shared.h` base utilities (`COM_Parse`, `Q_stricmp`, `Info_*`, math functions) from the shared game library
- `bg_itemlist` weapon/item definitions from `bg_misc.c` (consumed by `ui_players.c` for weapon model loading)

## Runtime Role

- **Init (`UI_INIT`):** `_UI_Init` registers cvars, populates the `displayContextDef_t` vtable, loads arena/bot/gameinfo data files, initializes the string pool and memory allocator, loads cached assets, and sets up the menu system via `Init_Display`.
- **Frame (`UI_REFRESH`):** `_UI_Refresh` updates server/player lists, paints all active menus via `Menu_PaintAll`, renders 3D player models where present (via `UI_DrawPlayer`), draws the cursor, and updates cvar-backed UI state.
- **Shutdown (`UI_SHUTDOWN`):** Not inferable from provided docs beyond being dispatched through `vmMain`.
- **Input:** Key and mouse events are dispatched per-frame through `UI_KeyEvent` and `UI_MouseEvent`, routed through `ui_shared.c`'s focus-stack and widget input handlers using `keyNum_t` identifiers.

## Notable Implementation Details

- The subsystem compiles in two forms: as a QVM (bytecode, using `ui_syscalls.asm` for syscall dispatch) and as a native DLL (using `ui_syscalls.c` with a `dllEntry`-initialized function pointer); the `trap_*` API surface is identical in both cases.
- `ui_shared.c` is conditionally compiled for both the `ui` and `cgame` modules, providing a single implementation of the widget framework shared across subsystems.
- All float arguments passed through the variadic integer syscall ABI are reinterpreted via the `PASSFLOAT` macro in `ui_syscalls.c` to avoid undefined behavior from float-to-int truncation.
- `K_CHAR_FLAG` (defined in `keycodes.h`) is a bitmask applied to key values to distinguish raw character input events from hardware key-code events inside menu handlers.
- `ui_util.c` is an empty scaffolding file; its intended memory and string allocation utilities are functionally located in `ui_shared.c` (`UI_Alloc`, `String_Alloc`) and `ui_atoms.c`.
- The virtual coordinate system is fixed at 640×480; `ui_atoms.c` applies a uniform scale transform to all draw calls to map these coordinates to the runtime screen resolution.
