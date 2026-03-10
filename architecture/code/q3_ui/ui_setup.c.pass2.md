# code/q3_ui/ui_setup.c — Enhanced Analysis

## Architectural Role

This file implements a single **hub menu screen** within the `q3_ui` VM's menu stack hierarchy. It serves as the primary access point for all player configuration sub-menus (player settings, controls, graphics, game options, CD key). The Setup menu participates in the **menu stack pattern** documented in the q3_ui architecture: it pushes itself onto the stack via `UI_PushMenu`, routes all activation events through a unified dispatcher, and pops itself when the user selects "BACK". This exemplifies Q3's event-driven, stack-based UI system where each screen is stateless between frames and rendering is handled generically by the menu framework (not here).

## Key Cross-References

### Incoming (who depends on this file)
- **Callers of `UI_SetupMenu()`**: Presumed to be a parent menu (e.g., main menu or in-game menu hub) in another `ui_*.c` file; not visible in the cross-reference excerpt but typical pattern
- **Menu framework infrastructure**: The `menuframework_s` instance stored in `setupMenuInfo` is registered with the menu stack and consumed by the generic `Menu_Draw` / `Menu_HandleKey` dispatch in `ui_qmenu.c`

### Outgoing (what this file depends on)
- **UI menu navigation layer** (`ui_atoms.c`): `UI_PushMenu`, `UI_PopMenu` (stack management); `color_white`, `color_red`, `color_yellow` (palette constants)
- **Menu framework core** (`ui_qmenu.c`): `Menu_AddItem` (widget registration); generic rendering/input handled elsewhere
- **Sub-menu entry points** (sibling `ui_*.c` files): `UI_PlayerSettingsMenu`, `UI_ControlsMenu`, `UI_GraphicsOptionsMenu`, `UI_PreferencesMenu`, `UI_CDKeyMenu`
- **Modal confirm dialog** (`ui_confirm.c`): `UI_ConfirmMenu` for the "SET TO DEFAULTS?" dialog
- **Trap syscalls** (VM→engine boundary): `trap_R_RegisterShaderNoMip` (asset pre-cache), `trap_Cvar_VariableValue` (read `cl_paused`), `trap_Cmd_ExecuteText` (execute `exec`, `cvar_restart`, `vid_restart`)
- **UI utilities** (`ui_atoms.c`): `UI_DrawProportionalString` (custom draw callback for confirm dialog)

## Design Patterns & Rationale

### Menu Stack & Event Dispatch
This file exemplifies Q3's **event-driven menu system**:
- No per-frame update loop; all behavior is reactive to activation (`QM_ACTIVATED`) events.
- Single dispatcher (`UI_SetupMenu_Event`) handles all menu item callbacks, routing on ID to specific sub-menus.
- Follows the **Command pattern**: each menu item encodes its action as an ID and callback function pointer.

### Asset Pre-Caching
The `UI_SetupMenu_Cache()` function pre-registers all bitmap shaders used by this menu. This is a consistent pattern across Q3's UI screens and reflects the engine's requirement that all renderer assets be GPU-resident before the frame containing their first draw call. The separate cache function allows assets to be loaded during a "warm-up" phase (e.g., when transitioning between menus) rather than lazily during first render.

### Conditional Item Visibility
The `cl_paused` check at init time gates the "DEFAULTS" option. This enforces a design rule: players cannot reset all settings from within an active game session, only from menus. However, **this check is one-time at menu creation**; if the game transitions from paused→unpaused after the Setup menu is open, the DEFAULTS button will incorrectly remain visible. This is a minor architectural limitation of the init-time check.

### Why This Structure?
- **Separation of concerns**: Each sub-menu owns its own state and widgets; Setup only routes navigation.
- **Reusability**: The callback and widget registration pattern is identical across all q3_ui menus, enabling code reuse in the framework layer.
- **VM sandbox safety**: Use of `trap_*` syscalls ensures the UI VM cannot directly manipulate engine state (cvars, commands, rendering) without explicit permission.

## Data Flow Through This File

**Input** → **Processing** → **Output**

1. **Input**: User presses a key or clicks a menu item. The generic menu framework (in `ui_qmenu.c`) dispatches a `QM_ACTIVATED` event to the topmost menu's registered callback.

2. **Processing**:
   - `UI_SetupMenu_Event` receives the event and extracts the item ID from the `menucommon_s` pointer.
   - A `switch` statement routes to the appropriate action:
     - **Navigation items** (PLAYER, CONTROLS, SYSTEM, GAME, CDKEY): Call the corresponding sub-menu entry point, which pushes a new menu onto the stack.
     - **DEFAULTS**: Push a modal confirm dialog (`UI_ConfirmMenu`) with a custom draw function (`Setup_ResetDefaults_Draw`) and callback (`Setup_ResetDefaults_Action`).
     - **BACK**: Pop this menu from the stack, returning to the parent.

3. **Output**:
   - **Sub-menu navigation**: Calls like `UI_PlayerSettingsMenu()` push a new menu onto the stack. The UI system's main loop will then dispatch input to that menu instead.
   - **Confirm dialog**: `UI_ConfirmMenu(...)` intercepts further events and re-routes based on user confirmation.
   - **Command execution**: On confirmed reset, `trap_Cmd_ExecuteText` enqueues three console commands (`exec default.cfg`, `cvar_restart`, `vid_restart`) to be executed by the engine's command buffer.
   - **Rendering**: The menu framework calls `Menu_Draw` on this menu's `menuframework_s` each frame; all rendering is automatic via the framework (width/height/color/style fields), except for the custom `Setup_ResetDefaults_Draw` which is invoked only by the confirm dialog.

## Learning Notes

### Idiomatic to Q3 / Era
- **Event-driven menus**: Modern engines typically use **retained-mode UI** (render once, update only when state changes) or **scene graph** systems. Q3's menu system is **immediate-mode** styled: each frame, the framework re-renders the menu from scratch based on the widget state structure. No render caching or dirty-rect optimization.
- **VM sandbox for UI**: Rather than linking UI code directly to the engine, Q3 runs the UI in its own VM and routes all operations through a trap syscall ABI. This allowed id Software to ship the UI source while preventing modifications that could break multiplayer security or stability.
- **Stack-based menu hierarchy**: Q3 uses a simple push/pop stack to manage menu nesting, similar to immediate-mode GUI libraries. No state machines or hierarchical scene graphs.
- **String-based cvar I/O**: The check `trap_Cvar_VariableValue("cl_paused")` returns a float; comparing `!= 0` is a common pattern in Q3 for boolean cvars.

### Contrast with Modern Practice
- Modern game engines (Unreal, Unity) use **data-driven UI** (markup, layout constraints, computed styling).
- Modern UIs often use **reactive/functional** patterns (e.g., React-style state → render).
- Modern engines typically do **not** sandbox UI in a separate VM; UI code is trusted and can directly manipulate engine state.

### Connections to Engine Concepts
- **Menu stack** ↔ **Call stack**: Pushing a submenu is analogous to a function call; popping is like return.
- **trap_* syscalls** ↔ **Sandbox enforcement**: The VM trap layer is the only bridge between untrusted code (UI) and trusted code (engine), similar to kernel/user-space boundaries in an OS.
- **Asset pre-caching** ↔ **Resource management**: The pattern of loading assets before use is fundamental to real-time graphics; Q3's explicit cache functions make this dependency visible.

## Potential Issues

1. **Dead Code**: The Load/Save menu items are permanently `#if 0`-guarded and will never compile into the menu. This should either be removed or restored with a feature flag.

2. **Cl_Paused Check Timing**: The `cl_paused` cvar is only read once, at menu init. If a game starts while paused and then unpauses, the DEFAULTS button will still be visible (or invisible, depending on initial state). A more robust approach would gate the item visibility in the draw or event-handling phase.

3. **Synchronous Heavy Command**: `vid_restart` is a full renderer restart, which is an expensive operation. Resetting all config with a renderer restart happens atomically; if the user cancels at the wrong moment, the state is undefined. Most modern engines queue such operations and execute them on a safe frame boundary.

4. **No Error Feedback**: If `exec default.cfg` fails (e.g., file missing), the user has no indication. The menu just silently executes `cvar_restart` and `vid_restart`. Ideally, a script error or console message would inform the user.
