# code/q3_ui/ui_atoms.c

## File Purpose
Core UI module for Quake III Arena's legacy menu system (`q3_ui`), providing the foundational drawing primitives, menu stack management, input dispatch, and per-frame refresh logic used by all menu screens.

## Core Responsibilities
- Maintain and manage the menu stack (`UI_PushMenu`, `UI_PopMenu`, `UI_ForceMenuOff`)
- Dispatch keyboard and mouse input events to the active menu
- Draw proportional (bitmap font) strings in multiple styles (normal, banner, shadow, pulse, inverse, wrapped)
- Draw fixed-width strings with Quake color code support
- Provide 640×480 virtual-coordinate primitives (`UI_FillRect`, `UI_DrawRect`, `UI_DrawHandlePic`, etc.)
- Initialize and refresh the UI system each frame (`UI_Init`, `UI_Refresh`)
- Route console commands to specific menu entry points (`UI_ConsoleCommand`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `uiStatic_t` | struct (typedef) | Global UI state: cursor, menu stack, GL config, cached shader handles, scale/bias |
| `menuframework_s` | struct (typedef) | A single menu screen: item list, draw/key callbacks, cursor state, display flags |
| `menucommon_s` | struct (typedef) | Base type for all menu items: position, bounds, flags, callbacks |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `uis` | `uiStatic_t` | global | Master UI state singleton |
| `m_entersound` | `qboolean` | global | Deferred flag to play menu-enter sound after first draw |
| `propMap[128][3]` | `static int` | file-static | UV + width table for proportional font glyphs (256×256 atlas) |
| `propMapB[26][3]` | `static int` | file-static | UV + width table for large banner font (A–Z only) |

## Key Functions / Methods

### UI_Init
- **Signature:** `void UI_Init( void )`
- **Purpose:** Initialize the entire UI subsystem at startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `uis.glconfig`; computes `uis.scale` and `uis.bias` for virtual 640×480 mapping; calls `Menu_Cache()`, `UI_RegisterCvars()`, `UI_InitGameinfo()`; resets active menu state.
- **Calls:** `trap_GetGlconfig`, `Menu_Cache`, `UI_RegisterCvars`, `UI_InitGameinfo`
- **Notes:** Wide-screen bias is applied only when `vidWidth/vidHeight > 640/480`.

### UI_Refresh
- **Signature:** `void UI_Refresh( int realtime )`
- **Purpose:** Per-frame UI update and render: draws active menu background, invokes menu draw callback, renders cursor, plays deferred sounds.
- **Inputs:** `realtime` — current time in ms
- **Outputs/Return:** None
- **Side effects:** Updates `uis.frametime`/`uis.realtime`; issues renderer draw calls; plays `menu_in_sound` once via `m_entersound`.
- **Calls:** `UI_UpdateCvars`, `UI_DrawHandlePic`, `Menu_Draw`, `UI_MouseEvent`, `UI_SetColor`, `trap_S_StartLocalSound`
- **Notes:** Early-exits if `KEYCATCH_UI` is not set. Debug cursor-coordinate overlay behind `#ifndef NDEBUG`.

### UI_PushMenu
- **Signature:** `void UI_PushMenu( menuframework_s *menu )`
- **Purpose:** Push a menu onto the stack, making it active; prevents duplicate stack entries.
- **Inputs:** `menu` — menu to push
- **Outputs/Return:** None
- **Side effects:** Modifies `uis.stack`, `uis.menusp`, `uis.activemenu`; sets `KEYCATCH_UI`; sets `m_entersound = qtrue`; resets cursor to first non-grayed item.
- **Calls:** `trap_Key_SetCatcher`, `Menu_SetCursor`, `trap_Error` (on overflow)

### UI_PopMenu
- **Signature:** `void UI_PopMenu( void )`
- **Purpose:** Pop the topmost menu, restoring the previous one or clearing the UI if the stack empties.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Decrements `uis.menusp`; calls `UI_ForceMenuOff` when stack reaches 0; plays `menu_out_sound`.
- **Calls:** `trap_S_StartLocalSound`, `UI_ForceMenuOff`, `trap_Error` (on underflow)

### UI_KeyEvent
- **Signature:** `void UI_KeyEvent( int key, int down )`
- **Purpose:** Route a key event to the active menu's key handler.
- **Inputs:** `key` — keycode; `down` — pressed state
- **Side effects:** May play a sound via `trap_S_StartLocalSound`.
- **Calls:** `uis.activemenu->key` or `Menu_DefaultKey`

### UI_MouseEvent
- **Signature:** `void UI_MouseEvent( int dx, int dy )`
- **Purpose:** Update cursor position and perform hit-testing against active menu items, updating focus/hover state.
- **Inputs:** `dx`, `dy` — relative mouse deltas
- **Side effects:** Clamps `uis.cursorx/cursory` to screen bounds; sets/clears `QMF_HASMOUSEFOCUS`; calls `Menu_SetCursor`; plays `menu_move_sound`.

### UI_DrawProportionalString
- **Signature:** `void UI_DrawProportionalString( int x, int y, const char* str, int style, vec4_t color )`
- **Purpose:** Draw a string using the proportional bitmap font with alignment, drop shadow, inverse, and pulse style flags.
- **Inputs:** Screen coords, string, style bitmask, base color
- **Side effects:** Issues `trap_R_DrawStretchPic` / `trap_R_SetColor` calls.
- **Calls:** `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`, `UI_DrawProportionalString2`

### UI_SetActiveMenu
- **Signature:** `void UI_SetActiveMenu( uiMenuCommand_t menu )`
- **Purpose:** Engine-facing entry point to activate a named menu by enum (main, in-game, CD-key prompts, etc.).
- **Calls:** `Menu_Cache`, then the appropriate `UI_*Menu()` function per enum value.
- **Notes:** This is documented as the **only** intended way to open menus from outside the UI module.

### UI_ConsoleCommand
- **Signature:** `qboolean UI_ConsoleCommand( int realTime )`
- **Purpose:** Handle UI-specific console commands (`levelselect`, `postgame`, `ui_cache`, `iamacheater`, etc.).
- **Outputs/Return:** `qtrue` if consumed, `qfalse` otherwise.

## Control Flow Notes
- **Init:** `UI_Init` is called once at startup by the engine VM entry point.
- **Frame:** `UI_Refresh` is called every frame by the engine when the UI VM is active.
- **Input:** `UI_KeyEvent` / `UI_MouseEvent` are called by the engine on input events.
- **Activation:** `UI_SetActiveMenu` is the external API; internally menus call `UI_PushMenu`/`UI_PopMenu`.
- **Shutdown:** `UI_Shutdown` is a no-op stub.

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_*` syscall wrappers (defined in `ui_syscalls.c`) — all renderer, sound, key, cvar, and cmd operations
- `Menu_Cache`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_SetCursor` — defined in `ui_qmenu.c`
- `g_color_table`, `Q_IsColorString`, `ColorIndex` — defined in `q_shared.c`
- All `UI_*Menu()` and `*_Cache()` functions — defined in their respective `ui_*.c` files
