# code/q3_ui/ui_spskill.c

## File Purpose
Implements the single-player difficulty selection menu in Quake III Arena's UI module. It presents five skill levels ("I Can Win" through "NIGHTMARE!"), persists the selection to the `g_spSkill` cvar, and transitions into the arena start flow.

## Core Responsibilities
- Initialize and lay out all menu widgets for the skill selection screen
- Highlight the currently selected skill in white; all others in red
- Update `g_spSkill` cvar when the player selects a difficulty
- Swap the displayed skill-level illustration (`art_skillPic`) on selection change
- Play a special sound for NIGHTMARE difficulty; silence sound otherwise
- Navigate back to the previous menu or forward to `UI_SPArena_Start`
- Pre-cache all shaders and sounds required by this menu

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `skillMenuInfo_t` | struct | Aggregates all menu widgets, arena info pointer, preloaded shader handles, and sound handles for the skill menu |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `skillMenuInfo` | `skillMenuInfo_t` | static (file) | Single instance holding all runtime state for the skill menu |

## Key Functions / Methods

### SetSkillColor
- **Signature:** `static void SetSkillColor( int skill, vec4_t color )`
- **Purpose:** Sets the text color of the menu item corresponding to `skill` (1–5).
- **Inputs:** `skill` — 1-based difficulty index; `color` — RGBA vec4
- **Outputs/Return:** void; mutates `skillMenuInfo.item_*` color fields
- **Side effects:** Modifies file-static `skillMenuInfo`
- **Calls:** None
- **Notes:** Default case is a no-op; out-of-range skill values are silently ignored

### UI_SPSkillMenu_SkillEvent
- **Signature:** `static void UI_SPSkillMenu_SkillEvent( void *ptr, int notification )`
- **Purpose:** Callback fired when a skill-level text item is activated; deselects old skill, selects new one, swaps illustration shader, plays appropriate sound.
- **Inputs:** `ptr` — `menucommon_s*` of the activated item; `notification` — event type
- **Outputs/Return:** void
- **Side effects:** Writes `g_spSkill` cvar; mutates `skillMenuInfo.art_skillPic.shader`; triggers local sound
- **Calls:** `SetSkillColor`, `trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_S_StartLocalSound`
- **Notes:** Guards on `QM_ACTIVATED`; NIGHTMARE plays `nightmareSound`, all others play `silenceSound`

### UI_SPSkillMenu_FightEvent
- **Signature:** `static void UI_SPSkillMenu_FightEvent( void *ptr, int notification )`
- **Purpose:** Callback for the FIGHT button; begins the arena with the currently stored arena info.
- **Inputs:** `ptr` — unused; `notification` — event type
- **Outputs/Return:** void
- **Side effects:** Calls into `UI_SPArena_Start`, which pushes game-start logic
- **Calls:** `UI_SPArena_Start`
- **Notes:** Guards on `QM_ACTIVATED`

### UI_SPSkillMenu_BackEvent
- **Signature:** `static void UI_SPSkillMenu_BackEvent( void *ptr, int notification )`
- **Purpose:** Callback for the BACK button; pops the skill menu off the menu stack.
- **Inputs:** `ptr` — unused; `notification` — event type
- **Outputs/Return:** void
- **Side effects:** Plays silence sound; calls `UI_PopMenu`
- **Calls:** `trap_S_StartLocalSound`, `UI_PopMenu`

### UI_SPSkillMenu_Key
- **Signature:** `static sfxHandle_t UI_SPSkillMenu_Key( int key )`
- **Purpose:** Custom key handler; intercepts MOUSE2/ESCAPE to play silence sound before delegating to default key processing.
- **Inputs:** `key` — key code
- **Outputs/Return:** `sfxHandle_t` from `Menu_DefaultKey`
- **Side effects:** May play silence sound
- **Calls:** `trap_S_StartLocalSound`, `Menu_DefaultKey`

### UI_SPSkillMenu_Cache
- **Signature:** `void UI_SPSkillMenu_Cache( void )`
- **Purpose:** Pre-registers all shaders and sounds needed by the skill menu; stores handles in `skillMenuInfo`.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Populates `skillMenuInfo.skillpics[]`, `nightmareSound`, `silenceSound`; issues renderer/sound registration calls
- **Calls:** `trap_R_RegisterShaderNoMip`, `trap_S_RegisterSound`

### UI_SPSkillMenu_Init
- **Signature:** `static void UI_SPSkillMenu_Init( void )`
- **Purpose:** Zeros the menu state, configures all widget properties and layout, registers items with the menu framework, and applies the initial skill highlight.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Fully overwrites `skillMenuInfo`; may play nightmare sound if current skill is 5
- **Calls:** `memset`, `UI_SPSkillMenu_Cache`, `Menu_AddItem`, `Com_Clamp`, `trap_Cvar_VariableValue`, `SetSkillColor`, `trap_S_StartLocalSound`

### UI_SPSkillMenu
- **Signature:** `void UI_SPSkillMenu( const char *arenaInfo )`
- **Purpose:** Public entry point; initializes the menu, stores the arena info pointer, pushes the menu, and sets default cursor focus to the FIGHT button.
- **Inputs:** `arenaInfo` — opaque arena descriptor string passed through to `UI_SPArena_Start`
- **Outputs/Return:** void
- **Side effects:** Pushes a new menu frame onto the UI stack
- **Calls:** `UI_SPSkillMenu_Init`, `UI_PushMenu`, `Menu_SetCursorToItem`

## Control Flow Notes
This file is purely event-driven UI. `UI_SPSkillMenu` is called by the SP level menu when the player chooses an arena. It pushes onto the menu stack (frame-driven by the UI refresh loop). Widget callbacks handle selection changes and navigation; no per-frame update logic exists in this file.

## External Dependencies
- `ui_local.h` — menu framework types, widget types, trap syscall declarations, helper functions
- **Defined elsewhere:** `UI_SPArena_Start`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Com_Clamp`, all `trap_*` syscall wrappers, `color_red`, `color_white`
