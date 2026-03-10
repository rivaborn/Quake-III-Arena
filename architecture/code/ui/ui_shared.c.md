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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `scrollInfo_t` | struct | Tracks auto-scroll state (timing, direction, captured item) for listbox/slider thumb drag |
| `scrollInfo_s` / `scrollInfo` | struct / global instance | Single live scroll capture state |
| `bind_t` | struct | Maps a command string to up to two key bindings (default + current) |
| `configcvar_t` | struct | Unused legacy cvar config storage (non-MissionPack only) |
| `keywordHash_t` | struct | Hash table node for keyword → parse-function dispatch |
| `commandDef_t` | typedef (from header) | Maps a script command name to its handler function |
| `displayContextDef_t` | typedef (from header) | Engine callback table + cursor position + assets, injected via `Init_Display` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `scrollInfo` | `scrollInfo_t` | static | Single active scroll-capture state |
| `captureFunc` | `void (*)(void*)` | static | Callback invoked each frame while an item has mouse capture |
| `captureData` | `void*` | static | Data passed to `captureFunc` |
| `itemCapture` | `itemDef_t*` | static | Item currently holding mouse capture |
| `DC` | `displayContextDef_t*` | global | Injected engine drawing/cvar/sound context |
| `g_waitingForKey` | `qboolean` | static | True while a bind item is waiting for a keypress |
| `g_editingField` | `qboolean` | static | True while a text/numeric field is being edited |
| `g_bindItem` | `itemDef_t*` | static | Item currently in key-bind capture mode |
| `g_editItem` | `itemDef_t*` | static | Item currently being text-edited |
| `Menus` | `menuDef_t[MAX_MENUS]` | global | All loaded menus |
| `menuCount` | `int` | global | Number of loaded menus |
| `menuStack` | `menuDef_t*[MAX_OPEN_MENUS]` | global | Stack of previously focused menus |
| `openMenuCount` | `int` | global | Depth of menu stack |
| `debugMode` | `qboolean` | static | Enables debug rect overlays (toggled by F11) |
| `memoryPool` | `char[]` | static | Fixed UI allocation arena (128 KB cgame / 1 MB ui) |
| `allocPoint` | `int` | static | Bump-pointer offset into `memoryPool` |
| `strPool` | `char[]` | static | String interning character pool |
| `strHandle` | `stringDef_t*[2048]` | static | Hash table for interned strings |
| `g_bindings` | `bind_t[]` | static | Full key-binding table for all game commands |
| `g_nameBind1/2` | `char[32]` | global | Formatted binding strings used by `Item_Bind_Paint` |
| `itemParseKeywordHash` | `keywordHash_t*[512]` | static | Hash table for item keyword dispatch |
| `menuParseKeywordHash` | `keywordHash_t*[512]` | static | Hash table for menu keyword dispatch |
| `lastListBoxClickTime` | `int` | static | Timestamp for double-click detection |

## Key Functions / Methods

### String_Init
- **Signature:** `void String_Init()`
- **Purpose:** Resets all UI state: string pool, menu arrays, memory pool; builds keyword hash tables; loads key binding config if available.
- **Inputs:** None (uses globals)
- **Outputs/Return:** None
- **Side effects:** Resets all global UI state; calls `Controls_GetConfig` if `DC->getBindingBuf` is available.
- **Calls:** `UI_InitMemory`, `Item_SetupKeywordHash`, `Menu_SetupKeywordHash`, `Controls_GetConfig`

### Item_RunScript
- **Signature:** `void Item_RunScript(itemDef_t *item, const char *s)`
- **Purpose:** Tokenizes a semicolon-delimited script string and dispatches each command to `commandList` handlers or falls back to `DC->runScript`.
- **Inputs:** `item` — context item; `s` — script string
- **Side effects:** May open/close menus, set cvars, play sounds, modify item colors/visibility.
- **Calls:** `String_Parse`, `Q_stricmp`, `DC->runScript`, all `Script_*` handlers

### Menu_HandleKey
- **Signature:** `void Menu_HandleKey(menuDef_t *menu, int key, qboolean down)`
- **Purpose:** Central input dispatcher for a menu: routes to bind capture, text field editing, focused item key handler, or default navigation/action handling.
- **Inputs:** `menu`, `key`, `down`
- **Side effects:** May change focus, trigger scripts, open/close menus, enter bind/edit modes.
- **Calls:** `Item_Bind_HandleKey`, `Item_TextField_HandleKey`, `Item_HandleKey`, `Item_Action`, `Menu_SetPrevCursorItem`, `Menu_SetNextCursorItem`, `Menus_HandleOOBClick`
- **Notes:** Uses a `static qboolean inHandler` re-entrancy guard.

### Item_Paint
- **Signature:** `void Item_Paint(itemDef_t *item)`
- **Purpose:** Per-frame render entry for a single item: updates orbit/transition animations, checks cvar-driven visibility, then dispatches to the type-specific paint function.
- **Inputs:** `item`
- **Side effects:** Modifies `item->window.rectClient` for orbit/transition; calls DC drawing functions.
- **Calls:** `Window_Paint`, `Item_OwnerDraw_Paint`, `Item_Text_Paint`, `Item_TextField_Paint`, `Item_ListBox_Paint`, `Item_Model_Paint`, `Item_YesNo_Paint`, `Item_Multi_Paint`, `Item_Bind_Paint`, `Item_Slider_Paint`

### Menu_Paint
- **Signature:** `void Menu_Paint(menuDef_t *menu, qboolean forcePaint)`
- **Purpose:** Renders a menu's background, window, and all child items.
- **Calls:** `Window_Paint`, `Item_Paint` (for each item), `DC->drawHandlePic`, `DC->drawRect`

### Menu_PaintAll
- **Signature:** `void Menu_PaintAll()`
- **Purpose:** Top-level per-frame UI render: runs `captureFunc` if active, then paints all menus.
- **Calls:** `captureFunc`, `Menu_Paint`

### Item_ListBox_HandleKey
- **Signature:** `qboolean Item_ListBox_HandleKey(itemDef_t *item, int key, qboolean down, qboolean force)`
- **Purpose:** Handles all keyboard and mouse input for listbox items including cursor movement, page navigation, scrollbar interaction, and double-click detection.
- **Calls:** `DC->feederSelection`, `Item_RunScript`, `Rect_ContainsPoint`

### Item_Bind_HandleKey
- **Signature:** `qboolean Item_Bind_HandleKey(itemDef_t *item, int key, qboolean down)`
- **Purpose:** Manages key-binding UI: initiates capture on mouse1/enter, handles ESC/backspace to clear, assigns new bindings and evicts conflicts.
- **Side effects:** Modifies `g_bindings`, sets `g_waitingForKey`, calls `Controls_SetConfig`.
- **Calls:** `BindingIDFromName`, `Controls_SetConfig`, `DC->setBinding`

### Controls_GetConfig / Controls_SetConfig
- **Signatures:** `void Controls_GetConfig(void)` / `void Controls_SetConfig(qboolean restart)`
- **Purpose:** Read current engine key bindings into `g_bindings` / write `g_bindings` back to the engine and optionally restart input.
- **Calls:** `DC->getBindingBuf` / `DC->setBinding`, `DC->executeText`

### Menu_New
- **Signature:** `void Menu_New(int handle)`
- **Purpose:** Allocates and parses a new menu from a PC token stream into the `Menus` array.
- **Calls:** `Menu_Init`, `Menu_Parse`, `Menu_PostParse`

### UI_Alloc
- **Signature:** `void *UI_Alloc(int size)`
- **Purpose:** Bump-pointer allocator from the fixed `memoryPool`; 16-byte aligned. Sets `outOfMemory` on exhaustion.

### String_Alloc
- **Signature:** `const char *String_Alloc(const char *p)`
- **Purpose:** Interns a string into `strPool` with hash-chained deduplication; returns pointer into pool.

## Control Flow Notes
- **Init:** `String_Init` → keyword hashes built, bindings loaded.
- **Load:** `Menu_New` called per menu file block → `Menu_Parse` → `Item_Parse` per item.
- **Frame:** `Menu_PaintAll` → `captureFunc` → `Menu_Paint` → `Item_Paint` per item; animations (orbit, transition, fade) advance per-item before draw.
- **Input:** `Display_HandleKey` → `Menu_HandleKey` → item-type-specific handler; `Display_MouseMove` → `Menu_HandleMouseMove` → `Item_SetFocus` / `Item_MouseEnter`.

## External Dependencies
- `ui_shared.h` → `q_shared.h`, `tr_types.h`, `keycodes.h`, `menudef.h`
- `trap_PC_ReadToken`, `trap_PC_SourceFileAndLine`, `trap_PC_LoadSource` — defined in platform-specific syscall stubs
- `COM_ParseExt`, `Q_stricmp`, `Q_strcat`, `Q_strupr` — defined in `q_shared.c`
- `Com_Printf` — engine print, defined elsewhere
- `AxisClear`, `AnglesToAxis`, `VectorSet`, `VectorCopy` — math, defined in `q_math.c`
- All `DC->*` function pointers — resolved at runtime via `Init_Display`
