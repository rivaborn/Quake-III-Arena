# code/q3_ui/ui_controls2.c — Enhanced Analysis

## Architectural Role

This file is the complete keybinding and input configuration subsystem for the legacy Q3A UI VM (`q3_ui`). It sits at the crucial boundary between the UI layer and the engine's input system, managing the cached copy of all 34 bindable actions and their key assignments, cvar state snapshots, and a live 3D player model preview. The file is entirely self-contained—no other UI code calls into it except the framework entry point `UI_ControlsMenu`.

## Key Cross-References

### Incoming (who depends on this file)
- **`q3_ui` menu framework** (`ui_main.c`, menu activation system): calls `UI_ControlsMenu()` when the player enters the Controls menu
- **Menu framework event dispatch** (`ui_qmenu.c`): routes all key events to `Controls_MenuKey` via `s_controls.menu.key` callback
- **Per-frame render loop** (`ui_main.c`): calls the menu framework's per-frame update, which invokes per-item `ownerdraw` callbacks (`Controls_DrawKeyBinding`, `Controls_DrawPlayer`)

### Outgoing (what this file depends on)
- **Engine syscalls** (via `ui_syscalls.c` stubs):
  - `trap_Key_SetBinding`, `trap_Key_GetBindingBuf`, `trap_Key_KeynumToStringBuf` — key binding query/update
  - `trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_Cvar_Reset` — cvar snapshot and persistence
  - `trap_R_RegisterShader`, `trap_R_RegisterModel` — asset caching for UI art and player model
  - `trap_Cmd_ExecuteText` — executes `"in_restart"` to notify input system of binding changes
- **q3_ui utilities** (`ui_local.h`):
  - `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_DrawPlayer` (from `ui_players.c`) — 3D player preview animation
  - `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_ClampCvar` — rendering and widget helpers
  - `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (from `ui_qmenu.c`) — menu framework primitives
  - `UI_ConfirmMenu` (from `ui_confirm.c`) — dialog for "reset to defaults" confirmation
- **Game module types** (`code/game/`):
  - `bg_public.h` / `bg_misc.c`: weapon enum (`WP_*`), item/armor lists, and leg/torso animation enums (`LEGS_*`, `TORSO_*`)
  - Player animation constants reused (`ANIM_*` → `LEGS_*/TORSO_*` mapping)

## Design Patterns & Rationale

**Singleton + Monolithic State:** All 140+ menu widgets live in a single static `controls_t s_controls` struct. This avoids heap allocation and parameter passing, typical of early-2000s VMs with limited memory. Trade-off: inflexible; scaling to more bindings would require struct expansion.

**Dual Caching (bindings + cvars):** Local `g_bindings[]` and `g_configcvars[]` tables shadow engine state. Only on explicit save (`Controls_SetConfig`) or key binding capture are changes written back. Why: minimize `trap_*` syscall overhead during menu interaction. Each read from engine is expensive; batch updates on exit.

**Owner-Draw for Dynamic Binding Display:** `Controls_DrawKeyBinding` is called per-frame by the menu framework, allowing live update of which key is bound to each action. The color (orange) and string rendering are computed fresh each frame, so if the user presses a key outside the menu and returns, the display updates automatically.

**Waiting-for-Key State Machine:** The `waitingforkey` flag acts as a modal overlay. When set, `Controls_MenuKey` captures the next key (ignoring `K_CHAR_FLAG` printables) and either assigns it or clears it. While waiting, `Controls_Update` grays out all items. Why: prevents accidental menu navigation while the user is mid-rebind.

**Tabbed Section Visibility:** Rather than multiple menus, a single menu with dynamic show/hide of widget rows. `g_controls[C_MOVEMENT/LOOKING/WEAPONS/MISC]` arrays index which items to show. `Controls_Update` repositions and hides items off-screen. Why: simpler code path; unified widget lifecycle.

## Data Flow Through This File

```
Engine State (at menu open)
    ↓
Controls_InitCvars → snapshot current + default cvar values → g_configcvars[]
Controls_GetConfig → query engine binding table → populate g_bindings[].bind1/2
                  → read slider/checkbox widget values from cvars
    ↓
User interaction in menu
    ↓
[While waiting for key]
Controls_MenuKey(key) → trap_Key_SetBinding (immediate flush to engine)
                     → Controls_Update (gray items, update display)
    ↓
[On save/exit]
Controls_SetConfig → write all g_bindings[] → trap_Key_SetBinding (batch)
                  → write all widget cvars → trap_Cvar_SetValue (batch)
                  → trap_Cmd_ExecuteText("in_restart") → engine input system reinits
    ↓
Engine State (updated)
```

Player preview state is **independent**: `Controls_UpdateModel(anim)` is called whenever a binding item gains focus; it mutates `s_controls.playerLegs/Torso/Weapon/ViewAngles` and calls `UI_PlayerInfo_SetInfo`. The 3D renderer then draws this preview state via `UI_DrawPlayer` each frame.

## Learning Notes

**Key Architectural Insight:** This file shows how a lightweight UI VM achieves complex functionality by caching and deferring updates. Rather than querying the engine every frame (expensive syscall), it reads once at open, caches locally, and batches writes on close. This pattern is repeated throughout Q3A's VM design.

**Idiomatic to Q3A Era:**
- All-static file scope; no heap allocation within the UI module
- Heavy reliance on fixed-size arrays and pre-allocated structs (no dynamic reallocation)
- Owner-draw pattern for menu items (not modern data-driven UI)
- Direct syscall bridge; no middleware abstraction layer
- Modal state (waitingforkey) managed with a simple flag, not event queues

**Modern Engine Differences:**
- Contemporary engines separate input binding from UI: use a dedicated input remap system with validation, conflict detection, and undo
- q3_ui handles conflicts by **eviction** (new key steals old binding), not conflict dialog
- No undo/rollback: if you rebind and don't save, the old binding is lost
- No input profiles or preset layouts; only one active binding set

**Connection to Game Engine Concepts:**
- **VM boundary pattern:** The syscall stubs (`trap_*`) in `ui_syscalls.c` define the ABI. This file never calls engine functions directly; all coupling is through this vtable.
- **Scene graph adjacent:** The player preview uses the renderer's model/animation system, similar to how cgame populates entities. Here, a single hardcoded player entity is animated locally.
- **State machine:** The menu's `waitingforkey` state + `section` index create a simple FSM for mode (input capture vs. navigation) and content (which tab).

## Potential Issues

**Conflict Resolution by Eviction:** When a key is rebound, any existing binding to that key is silently cleared (`Controls_MenuKey` → `trap_Key_SetBinding(key, "")`). A user may not realize they've overwritten a crucial binding (e.g., rebinding Attack to a held modifier and losing Fire). Modern UIs show a conflict dialog.

**No Persistence of Unsaved Changes:** If the user binds keys, then exits without saving, the changes are lost without warning. The `changesmade` flag is set but never checked before pop.

**Player Preview Desync Risk:** The preview player state is computed fresh each frame from `s_controls.playerLegs/Torso`, which is updated only when a binding item gains focus. If the renderer glitches or the animation data is corrupted, there is no fallback or warning; the menu simply displays corrupted animation.

**Monolithic Size:** At 1667 lines in a single file, adding new control categories (e.g., "Advanced" section) requires struct growth and hardcoded section indices. No plugin architecture.

---

**Last Enhanced:** 2026-03-01  
**Scope:** Cross-system data flow, UI↔Engine boundary, and Q3A-era design idioms.
