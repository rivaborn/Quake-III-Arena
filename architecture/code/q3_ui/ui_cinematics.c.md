# code/q3_ui/ui_cinematics.c

## File Purpose
Implements the Cinematics menu for the Quake III Arena UI, allowing players to replay pre-rendered RoQ cutscene videos (id logo, intro, tier completions, and ending). It builds and presents a scrollable text-button list that triggers `disconnect; cinematic <name>.RoQ` commands when activated.

## Core Responsibilities
- Define and initialize all menu items for the Cinematics screen (banner, frame art, text buttons, back button)
- Gray out tier cinematic entries that the player has not yet unlocked via `UI_CanShowTierVideo`
- Handle back-navigation by popping the menu stack
- On item activation, set the `nextmap` cvar and issue a disconnect + cinematic playback command
- Handle the demo version special case for the "END" cinematic
- Expose a console-command entry point (`UI_CinematicsMenu_f`) that also repositions the cursor to a specific item
- Precache menu art shaders via `UI_CinematicsMenu_Cache`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `cinematicsMenuInfo_t` | struct | Aggregates the entire menu's widgets: framework, banner, frame bitmaps, 10 cinematic text buttons, and a back button |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cinematicsMenuInfo` | `cinematicsMenuInfo_t` | static (file) | Single persistent instance of the cinematics menu state; re-initialized on each open |
| `cinematics[]` | `static char *[]` | static (file) | Ordered array of RoQ base filenames aligned to `ID_CIN_*` IDs for index-based lookup |

## Key Functions / Methods

### UI_CinematicsMenu_BackEvent
- **Signature:** `static void UI_CinematicsMenu_BackEvent( void *ptr, int event )`
- **Purpose:** Callback for the Back button; pops the cinematics menu off the UI stack.
- **Inputs:** `ptr` — pointer to the triggering menu item; `event` — event type
- **Outputs/Return:** void
- **Side effects:** Calls `UI_PopMenu`, modifying `uis.menusp` and `uis.activemenu`
- **Calls:** `UI_PopMenu`
- **Notes:** Ignores all events except `QM_ACTIVATED`

---

### UI_CinematicsMenu_Event
- **Signature:** `static void UI_CinematicsMenu_Event( void *ptr, int event )`
- **Purpose:** Callback for each cinematic text button; sets `nextmap` and issues a console command to disconnect and play the selected RoQ file.
- **Inputs:** `ptr` — `menucommon_s *` with `.id` identifying the cinematic; `event` — event type
- **Outputs/Return:** void
- **Side effects:** Sets cvar `nextmap`; appends a `disconnect; cinematic <name>.RoQ` console command; demo version uses `demoEnd.RoQ` with the `1` (loop?) flag for `ID_CIN_END`
- **Calls:** `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `va`
- **Notes:** Index `n` is derived as `id - ID_CIN_IDLOGO`, used to index `cinematics[]`; `uis.demoversion` gates the alternate end-cinematic path

---

### UI_CinematicsMenu_Init
- **Signature:** `static void UI_CinematicsMenu_Init( void )`
- **Purpose:** Constructs all menu items, sets positions/flags/callbacks, and registers them with the menu framework.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Zeroes and populates `cinematicsMenuInfo`; calls `Menu_AddItem` 14 times; conditionally sets `QMF_GRAYED` on locked tier entries
- **Calls:** `UI_CinematicsMenu_Cache`, `memset`, `UI_CanShowTierVideo`, `Menu_AddItem`
- **Notes:** `y` starts at 100 and increments by `VERTICAL_SPACING` (30) per entry; the back button is anchored at `y=416` (480−64); demo-version grays `cin_intro`

---

### UI_CinematicsMenu_Cache
- **Signature:** `void UI_CinematicsMenu_Cache( void )`
- **Purpose:** Precaches all four UI art shaders used by this menu.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for `back_0`, `back_1`, `frame2_l`, `frame1_r`
- **Calls:** `trap_R_RegisterShaderNoMip` ×4

---

### UI_CinematicsMenu
- **Signature:** `void UI_CinematicsMenu( void )`
- **Purpose:** Public entry point; initializes and pushes the cinematics menu.
- **Calls:** `UI_CinematicsMenu_Init`, `UI_PushMenu`

---

### UI_CinematicsMenu_f
- **Signature:** `void UI_CinematicsMenu_f( void )`
- **Purpose:** Console-command variant; opens the menu and moves cursor to the cinematic at index `argv[1]`.
- **Inputs:** Console argument 1 — integer index into the cinematic list
- **Side effects:** Calls `UI_CinematicsMenu`, then `Menu_SetCursorToItem`; uses `items[n + 3]` to skip banner, framel, framer
- **Calls:** `atoi`, `UI_Argv`, `UI_CinematicsMenu`, `Menu_SetCursorToItem`
- **Notes:** The `+3` offset skips the three non-selectable decorative items at the front of the items array

## Control Flow Notes
This file participates in UI menu lifecycle only: `UI_CinematicsMenu` / `UI_CinematicsMenu_f` are called from the UI atom/command dispatcher. There is no per-frame update logic; all interaction flows through the menu framework's key/mouse event pump calling the registered callbacks. No render or physics involvement.

## External Dependencies
- **`ui_local.h`** — pulls in all menu framework types, trap syscalls, `uis` global, `UI_CanShowTierVideo`, `UI_PopMenu`, `UI_PushMenu`, `va`, `color_red`, `color_white`, `QMF_*`, `QM_ACTIVATED`, `MTYPE_*`
- **Defined elsewhere:** `UI_CanShowTierVideo` (`ui_gameinfo.c`), `UI_PopMenu`/`UI_PushMenu` (`ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_AddItem`/`Menu_SetCursorToItem` (`ui_qmenu.c`), `uis` global state (`ui_atoms.c`)
