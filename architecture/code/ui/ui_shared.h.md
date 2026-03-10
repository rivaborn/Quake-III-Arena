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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `rectDef_t` / `Rectangle` | typedef struct | 2D float rectangle (x, y, w, h) |
| `windowDef_t` / `Window` | typedef struct | Core visual window: rect, colors, flags, cinematic, border, effects |
| `itemDef_t` | typedef struct | Full UI item: inherits window, adds type, text, cvar binding, scripts, colors, type-specific data ptr |
| `menuDef_t` | typedef struct | Menu container: window, item array (up to 96), fade params, open/close/ESC scripts |
| `displayContextDef_t` | typedef struct | Vtable + state: ~50 function pointers for renderer, sound, cvar, feeder, key, cinematic access |
| `cachedAssets_t` | typedef struct | Loaded handles for shared UI textures, fonts, sounds, crosshairs, FX pics |
| `listBoxDef_t` | typedef struct | List box state: scroll position, element size, column layout (up to 16 columns) |
| `editFieldDef_t` | typedef struct | Edit field constraints: min/max/default values, character limits, paint offset |
| `multiDef_t` | typedef struct | Multi-value selector: up to 32 cvar/string/float value pairs |
| `modelDef_t` | typedef struct | 3D model display: angle, origin, FOV, rotation speed |
| `colorRangeDef_t` | typedef struct | Color range entry: color + float low/high bounds |
| `columnInfo_t` | typedef struct | List box column: position, width, max characters |
| `commandDef_t` | typedef struct | Named UI command: string name + handler function pointer |
| `scriptDef_t` | typedef struct | Script invocation: command string + up to 12 args |

## Global / File-Static State

None declared in this header (defined in `ui_shared.c`).

## Key Functions / Methods

### String_Alloc / String_Init / String_Report
- **Signature:** `const char *String_Alloc(const char *p)` / `void String_Init()` / `void String_Report()`
- **Purpose:** Manage a fixed-size string intern pool (`STRING_POOL_SIZE`: 384KB UI, 128KB cgame). All `const char*` fields in menus/items point into this pool.
- **Notes:** Pool is never freed at runtime; exhaustion is a fatal error.

### Init_Display
- **Signature:** `void Init_Display(displayContextDef_t *dc)`
- **Purpose:** Install the platform-specific display context (renderer vtable + state) used by all subsequent UI calls.

### Menu_Paint / Menu_PaintAll
- **Signature:** `void Menu_Paint(menuDef_t *menu, qboolean forcePaint)` / `void Menu_PaintAll()`
- **Purpose:** Render one or all active menus, calling through `displayContextDef_t` function pointers for drawing.

### Menu_HandleKey / Display_HandleKey
- **Signature:** `void Menu_HandleKey(menuDef_t *menu, int key, qboolean down)` / `void Display_HandleKey(int key, qboolean down, int x, int y)`
- **Purpose:** Route keyboard input to the focused menu/item; dispatch scripted actions on activation.

### Menus_ActivateByName / Menus_OpenByName / Menus_CloseByName / Menus_CloseAll
- **Purpose:** Named menu activation/deactivation. Up to `MAX_OPEN_MENUS` (16) menus can be open simultaneously.

### Menu_New
- **Signature:** `void Menu_New(int handle)`
- **Purpose:** Parse and register a new menu from an open PC script source handle into the global menu table (up to `MAX_MENUS` = 64).

### PC_*_Parse helpers
- **Notes:** `PC_Float_Parse`, `PC_Int_Parse`, `PC_Color_Parse`, `PC_Rect_Parse`, `PC_String_Parse`, `PC_Script_Parse` — wrap `trap_PC_ReadToken` to extract typed values from preprocessed script files. Mirror the text-stream `Float_Parse` etc. variants.

### trap_PC_* functions
- **Signature:** `int trap_PC_LoadSource(const char *filename)` etc.
- **Purpose:** VM syscall wrappers for the preprocessed script (`.menu` file) parser subsystem. Defined in the VM syscall layer, declared here.

## Control Flow Notes
This header is included at the top of both `code/ui/` and `code/cgame/` modules. Initialization flow: `Init_Display` → `String_Init` → `Menu_New` (per menu file) → `Menu_PostParse`. Per-frame: `Menu_PaintAll` is called during the render phase; `Display_HandleKey` / `Display_MouseMove` are called from input dispatch. The `displayContextDef_t` vtable decouples UI logic from the engine, enabling the same `.menu` scripts and `ui_shared.c` logic to run in both the cgame HUD and the main UI VM.

## External Dependencies
- `../game/q_shared.h` — `vec4_t`, `qboolean`, `qhandle_t`, `sfxHandle_t`, `fontInfo_t`, `glconfig_t`, `refEntity_t`, `refdef_t`, `pc_token_t`
- `../cgame/tr_types.h` — `refEntity_t`, `refdef_t`, `glconfig_t`
- `keycodes.h` — `keyNum_t` enum
- `../../ui/menudef.h` — `ITEM_TYPE_*`, `FEEDER_*`, `CG_SHOW_*`, owner-draw constants
- `trap_PC_*` functions — defined elsewhere in the VM syscall table (`ui_syscalls.c` / `cg_syscalls.c`)
- `UI_Alloc` / `UI_InitMemory` / `UI_OutOfMemory` — VM-local memory pool, defined in `ui_main.c`
- `Controls_GetConfig` / `Controls_SetConfig` / `Controls_SetDefaults` — defined in `ui_shared.c`
