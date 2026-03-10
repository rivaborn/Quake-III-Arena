# code/q3_ui/ui_qmenu.c

## File Purpose
Implements the core menu framework and all standard widget types for Quake III Arena's legacy UI system (`q3_ui`). It provides initialization, drawing, and input handling for every interactive menu element, plus the top-level menu management routines.

## Core Responsibilities
- Register and cache all shared UI assets (shaders, sounds) via `Menu_Cache`
- Initialize widget bounding boxes and state on `Menu_AddItem`
- Dispatch per-frame drawing for all widget types via `Menu_Draw`
- Route keyboard/mouse input to the focused widget via `Menu_DefaultKey`
- Manage menu cursor movement, focus transitions, and wrap-around via `Menu_AdjustCursor` / `Menu_CursorMoved`
- Provide sound feedback (move, buzz, in/out) for all interactive events
- Support a debug overlay (bounding-box visualization) under `#ifndef NDEBUG`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `menuframework_s` | struct | Top-level menu container: item array, cursor, draw/key callbacks, wrap flag |
| `menucommon_s` | struct | Base "class" embedded first in every widget; holds type, position, flags, callbacks |
| `menuaction_s` | struct | Clickable text action button |
| `menuradiobutton_s` | struct | Binary on/off toggle |
| `menuslider_s` | struct | Continuous float-range slider |
| `menulist_s` | struct | Shared type for both SpinControl (cycle list) and ScrollList (scrollable list) |
| `menubitmap_s` | struct | Image widget with optional focus/pulse shader |
| `menutext_s` | struct | Static or proportional text label |
| `mfield_t` | struct | Editable text field buffer (defined in header, used via `menufield_s`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `menu_in_sound` | `sfxHandle_t` | global | Sound played when entering a menu |
| `menu_move_sound` | `sfxHandle_t` | global | Sound played on successful cursor movement |
| `menu_out_sound` | `sfxHandle_t` | global | Sound played when leaving a menu |
| `menu_buzz_sound` | `sfxHandle_t` | global | Sound played when input is blocked/at boundary |
| `menu_null_sound` | `sfxHandle_t` | global | Silent placeholder (value `-1`) |
| `weaponChangeSound` | `sfxHandle_t` | global | Weapon switch sound, shared with other UI screens |
| `sliderBar` | `qhandle_t` | static | Shader handle for slider track graphic |
| `sliderButton_0` | `qhandle_t` | static | Shader handle for unfocused slider thumb |
| `sliderButton_1` | `qhandle_t` | static | Shader handle for focused/pulsing slider thumb |
| `menu_text_color` … `text_color_status` | `vec4_t` | global | Shared RGBA palette constants used by all widgets |

## Key Functions / Methods

### Menu_Cache
- **Signature:** `void Menu_Cache( void )`
- **Purpose:** Loads all assets shared across every menu screen.
- **Inputs:** None (reads `uis.glconfig.hardwareType` for Rage Pro workaround).
- **Outputs/Return:** void
- **Side effects:** Writes shader handles into `uis.*`, writes sound handles into the six global `sfxHandle_t` vars and `sliderBar`/`sliderButton_*`.
- **Calls:** `trap_R_RegisterShaderNoMip`, `trap_S_RegisterSound`
- **Notes:** Must be called once before any menu is displayed; `menu_null_sound` is set to `-1` (nonzero sentinel, never played).

### Menu_AddItem
- **Signature:** `void Menu_AddItem( menuframework_s *menu, void *item )`
- **Purpose:** Appends a widget to a menu and runs its type-specific `_Init` function.
- **Inputs:** `menu` — owning framework; `item` — any widget pointer cast to `void*`.
- **Outputs/Return:** void
- **Side effects:** Sets `parent`, `menuPosition`, clears `QMF_HASMOUSEFOCUS` on the item; increments `menu->nitems`.
- **Calls:** Per-type `*_Init` dispatch; `trap_Error` on overflow or unknown type.
- **Notes:** Items flagged `QMF_NODEFAULTINIT` skip the init dispatch entirely.

### Menu_Draw
- **Signature:** `void Menu_Draw( menuframework_s *menu )`
- **Purpose:** Iterates all items and calls the appropriate `*_Draw` function; invokes `statusbar` callback of the cursor item.
- **Inputs:** `menu` — active menu.
- **Outputs/Return:** void
- **Side effects:** Calls renderer (`trap_R_SetColor`, `UI_Draw*`). Under `NDEBUG=0`, draws bounding-box rectangles for each active item.
- **Calls:** All `*_Draw` functions, `Menu_ItemAtCursor`, `itemptr->ownerdraw`, `itemptr->statusbar`.

### Menu_DefaultKey
- **Signature:** `sfxHandle_t Menu_DefaultKey( menuframework_s *m, int key )`
- **Purpose:** Central input router: handles Escape/mouse2 globally, delegates to the focused widget's `*_Key` function, then handles cursor navigation keys.
- **Inputs:** `m` — active menu; `key` — key constant.
- **Outputs/Return:** `sfxHandle_t` sound to play, or `0`.
- **Side effects:** May pop the menu stack (`UI_PopMenu`), move the cursor, activate items, or trigger `Menu_CursorMoved`.
- **Calls:** `UI_PopMenu`, `Menu_ItemAtCursor`, `SpinControl_Key`, `RadioButton_Key`, `Slider_Key`, `ScrollList_Key`, `MenuField_Key`, `Menu_AdjustCursor`, `Menu_CursorMoved`, `Menu_ActivateItem`.
- **Notes:** Debug keys `K_F11`/`K_F12` toggle `uis.debug` and trigger screenshot; compiled out in release.

### Menu_AdjustCursor
- **Signature:** `void Menu_AdjustCursor( menuframework_s *m, int dir )`
- **Purpose:** Advances the cursor in direction `dir` (+1/-1), skipping grayed/mouseonly/inactive items; optionally wraps.
- **Inputs:** `m`, `dir`.
- **Side effects:** Modifies `m->cursor`; may restore `m->cursor_prev` if no valid slot found.
- **Notes:** Uses a `goto wrap` label for the wrap-around retry; sets `wrapped` flag to prevent infinite loops.

### ScrollList_Key
- **Signature:** `sfxHandle_t ScrollList_Key( menulist_s *l, int key )`
- **Purpose:** Full keyboard + mouse navigation for multi-column scrollable lists; includes first-character type-ahead search.
- **Inputs:** `l` — list widget; `key` — key constant.
- **Outputs/Return:** Sound handle.
- **Side effects:** Updates `l->curvalue`, `l->oldvalue`, `l->top`; fires `QM_GOTFOCUS` callback on selection change.
- **Notes:** Multi-column layout is supported; `K_PGUP`/`K_PGDN` return `menu_null_sound` for multi-column lists.

### Slider_Draw (active `#if 1` branch)
- **Signature:** `static void Slider_Draw( menuslider_s *s )`
- **Purpose:** Draws slider label, track graphic, and thumb at position derived from normalized `s->range`.
- **Side effects:** Clamps and updates `s->range`; calls `UI_SetColor`, `UI_DrawHandlePic`.
- **Notes:** An alternate character-based implementation exists in the `#else` block (disabled).

## Control Flow Notes
`Menu_Cache` is called during UI initialization. Each frame, `UI_Refresh` calls the active menu's `draw` callback, which typically calls `Menu_Draw`. Key events from the engine flow through `UI_KeyEvent` → active menu's `key` callback → `Menu_DefaultKey`. This file has no direct involvement in game simulation or rendering outside the 2D menu pass.

## External Dependencies
- **`ui_local.h`** — brings in all widget type definitions, flag constants, `uis` global, and `trap_*` syscall declarations.
- **`trap_R_RegisterShaderNoMip`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_S_StartLocalSound`** — renderer/audio syscalls, defined in `ui_syscalls.c`.
- **`UI_Draw*`, `UI_FillRect`, `UI_SetColor`, `UI_CursorInRect`** — defined in `ui_atoms.c`.
- **`MenuField_Init`, `MenuField_Draw`, `MenuField_Key`** — defined in `ui_mfield.c`.
- **`UI_PopMenu`** — defined in `ui_atoms.c`.
- **`uis`** (`uiStatic_t`) — singleton global defined in `ui_atoms.c`.
- **`Menu_ItemAtCursor`** — defined in this file; also declared `extern` in `ui_local.h` for use by other modules.
