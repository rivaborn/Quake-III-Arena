# code/q3_ui/ui_spskill.c — Enhanced Analysis

## Architectural Role

This file implements the **difficulty selection menu** (aka "Skill Menu") in the single-player campaign flow. It serves as the final UI gate before `UI_SPArena_Start` transitions control to the game engine. The menu is a leaf node in the `q3_ui` menu stack—it has no child menus, only sibling navigation (back to arena selection, forward to game start). The difficulty choice persists to the `g_spSkill` cvar, which the server-side game VM (`code/game/`) reads at level load to configure difficulty-affecting gameplay parameters (bot skill, damage scaling, item availability). Thus this file bridges the UI VM and game VM across the cvar layer, exemplifying Q3A's use of cvars as the primary cross-VM state channel.

## Key Cross-References

### Incoming (who depends on this file)
- **UI framework (`ui_main.c` / `ui_atoms.c` via menu stack):** Calls `UI_SPSkillMenu(const char *arenaInfo)` to push the difficulty menu onto the stack after arena/level selection.
- **Menu stack dispatcher:** Invokes `UI_SPSkillMenu_Key` as the menu's custom key handler each frame.
- **Widget framework:** Calls the three event callbacks (`UI_SPSkillMenu_SkillEvent`, `UI_SPSkillMenu_FightEvent`, `UI_SPSkillMenu_BackEvent`) in response to item selection.

### Outgoing (what this file depends on)
- **Menu framework** (`ui_local.h`): `Menu_AddItem`, `Menu_DefaultKey`, `Menu_SetCursorToItem`, `menuframework_s`, `menutext_s`, `menubitmap_s`, menu flags (`QMF_*`), notification codes (`QM_ACTIVATED`), color constants (`color_red`, `color_white`).
- **Game flow integration:** `UI_SPArena_Start(arenaInfo)` — transitions from menu to game initialization; `UI_PushMenu` / `UI_PopMenu` — stack management.
- **Engine syscall layer (trap_*):** 
  - **Cvar bridge:** `trap_Cvar_VariableValue("g_spSkill")`, `trap_Cvar_SetValue("g_spSkill", skill)` — reads current and writes new difficulty (read by server game VM for difficulty-dependent logic).
  - **Sound system:** `trap_S_RegisterSound(...)`, `trap_S_StartLocalSound(...)` — nightmare/silence sound feedback.
  - **Renderer:** `trap_R_RegisterShaderNoMip(...)` — pre-caches difficulty-level artwork.
- **Shared utilities:** `Com_Clamp` — clamps skill to 1–5 range on init.

## Design Patterns & Rationale

### Cvar-Mediated Cross-VM State
The central pattern: `g_spSkill` is a shared configuration variable read by both the UI (to display current selection) and the server game VM (to configure bot skill, health scaling, etc.). This avoids direct function calls between VMs, which would require the server to poll the UI VM. Instead, the UI writes the cvar, the game VM reads it. **Rationale:** Cvars were Q3A's primary mechanism for decoupled, persistent configuration; this pattern scales to mods and user overrides.

### Event-Driven Menu with Typed Callbacks
Each menu item is a widget with a `callback` field pointing to a handler that fires on `QM_ACTIVATED`. The skill-text items all share `UI_SPSkillMenu_SkillEvent`, but the fight and back buttons have distinct handlers. **Rationale:** Reusable callback structure; minimal per-widget code; no per-frame dispatch needed.

### Retained-Mode Widget State + Immediate Resource Caching
All widget properties (position, color, shader, callback) are set once in `UI_SPSkillMenu_Init`, stored in the `skillMenuInfo` struct, and mutated only on user action (e.g., `SetSkillColor`). Resources (shaders, sounds) are registered upfront in `UI_SPSkillMenu_Cache`. **Rationale:** Avoids per-frame shader registration or widget property updates; reduces CPU cost; ensures all assets are valid before the menu is interactive.

### Single-Skill Color Dispatch via Switch Statement
`SetSkillColor` uses a simple 1–5 switch to pick which text item to recolor. No array of items, no loop. **Rationale:** Explicit and fast; skill count is constant (5); avoids indirection; readable code.

### Magic-Number Layout
All positions (e.g., `x=320, y=170` for "I Can Win") are hardcoded constants. The code assumes a fixed 640×480 virtual resolution. **Rationale:** Q3A's UI runs at fixed resolution (scaled to physical screen size by renderer). Avoids layout engine complexity; typical for early 2000s console/PC game UIs.

### Global Singleton vs. Dynamic Allocation
`skillMenuInfo` is a file-static struct zeroed once per menu open. No dynamic allocation, no fragmentation risk. **Rationale:** Quake engines prefer fixed-size structures and stack/static storage; avoids malloc/free churn in hot loops.

## Data Flow Through This File

1. **Entry:** `UI_SPSkillMenu(arenaInfo)` called by parent menu (arena/level selection).
   - Initializes `skillMenuInfo` to zero.
   - Calls `UI_SPSkillMenu_Cache()` to register all shaders and sounds (stored in `skillMenuInfo.skillpics[]`, `nightmareSound`, `silenceSound`).
   - Constructs all menu widgets in-place with layout properties and callbacks.
   - Reads `g_spSkill` cvar, clamps to 1–5, calls `SetSkillColor(..., color_white)` to highlight current selection.
   - Swaps `art_skillPic.shader` to display the corresponding skill-level artwork (e.g., map completion image).
   - If current skill is nightmare (5), plays nightmare sound.
   - Pushes menu onto UI stack; sets cursor focus to FIGHT button.

2. **Interactive Loop:** Menu sits on top of UI stack; each frame, input is dispatched to this menu:
   - **Key events:** `UI_SPSkillMenu_Key(key)` intercepts ESCAPE/MOUSE2 to play silence sound, then delegates to `Menu_DefaultKey` for standard key handling (navigation, selection).
   - **Item selection:** User clicks a skill-level text item → `UI_SPSkillMenu_SkillEvent` fires:
     - Reads old `g_spSkill`, colors that item red (deselect).
     - Extracts new skill from widget ID, writes `g_spSkill` cvar, colors new item white (select).
     - Swaps shader to display new skill-level graphic.
     - Plays nightmare sound if ID==NIGHTMARE; silence sound otherwise.
   - **Fight button:** User clicks FIGHT → `UI_SPSkillMenu_FightEvent`:
     - Calls `UI_SPArena_Start(skillMenuInfo.arenaInfo)`, transferring control to game init.
   - **Back button or ESCAPE:** User clicks BACK or presses ESCAPE → `UI_SPSkillMenu_BackEvent`:
     - Plays silence sound, calls `UI_PopMenu` to return to arena menu.

3. **Exit:** When FIGHT is clicked, `UI_SPArena_Start` pops this menu and initiates server connection / game load. The `g_spSkill` cvar persists for the server to read. If BACK is clicked, menu is popped and parent menu regains focus.

## Learning Notes

### Q3A-Era Game UI Idioms
1. **Cvar as application state glue:** Configuration that needs to survive across VM boundaries (UI → game) lives in cvars, not function returns or shared structs. The game module has no direct dependency on the UI VM; it simply reads `g_spSkill` at startup. This is a clean decoupling pattern.

2. **Resource pre-registration and handle reuse:** All renderer/sound assets are registered once at menu init and stored as opaque handles (`qhandle_t`). The menu then reuses these handles without re-registering. This avoids thrashing the asset system and ensures handles are valid throughout the menu's lifetime.

3. **Sound feedback for every interaction:** Even a menu selection change triggers a sound (nightmare for nightmare, silence for others). This is high-touch UX; modern games might reserve sound for confirmation, but Q3A/early arcade games played sounds for all feedback. **Lesson:** Audio feedback was a critical part of perceived responsiveness in low-latency offline menus.

4. **No animation, direct state swaps:** Color changes and shader swaps happen immediately, no easing or interpolation. This is practical given the menu runs at whatever frame rate the engine produces (uncapped, typically 60–120 FPS on 2000s hardware). **Modern difference:** Modern engines often animate UI transitions for visual polish; Q3A prioritizes simplicity.

5. **Global singleton + file-static scope:** The entire menu state is in a single `static` struct, not a class or dynamically allocated instance. This pattern is pervasive in Quake engine code and reflects C idioms from the 1990s–2000s. **Modern difference:** Object-oriented UI frameworks (or functional/data-driven approaches) now dominate.

6. **Explicit callback dispatch vs. message bus:** Each widget's callback is a direct function pointer. There's no event bus or signal/slot system. **Rationale:** Simple, fast, no indirection; downside is tightly coupled code.

### Connections to Game Engine Concepts

- **VM Boundary Crossing via Cvars:** This file exemplifies how Quake III's multi-VM architecture (cgame, game, ui in separate QVM sandboxes) communicates: not via direct calls, but via cvars and entity state. This is an early example of **decoupled state synchronization** seen in modern networked games.
  
- **Menu Stack / Retained Mode UI:** The menu framework uses a stack-based navigation model (push/pop) common in console games and early GUI toolkits. This contrasts with modern immediate-mode UI or HTML-like declarative menus.

- **Spatial Layout in 2D:** The hardcoded x/y positions reflect a **fixed virtual canvas** (640×480), which the renderer scales to any physical resolution. This is different from responsive/adaptive UIs that reflow based on screen size.

## Potential Issues

1. **Silent Failure on Asset Registration:** If `trap_R_RegisterShaderNoMip` or `trap_S_RegisterSound` returns an invalid handle, the menu proceeds anyway. The shader/sound will simply be NULL/missing, and rendering/playback will silently fail. For robustness, one might add `if (!handle) Com_Error(...)`, though this is not critical in practice (the engine logs warnings).

2. **Out-of-Range Skill Values Ignored:** `SetSkillColor` has a `default: break` case that silently ignores skill values outside 1–5. If a bug causes skill=0 or skill=6, the color won't update but the cvar will be set. Low risk, but could hide logic errors. Mitigation: `Com_Clamp` in `UI_SPSkillMenu_Init` ensures the initial value is safe; subsequent calls from `UI_SPSkillMenu_SkillEvent` only pass valid IDs.

3. **Nightmare Sound on Re-Entry:** In `UI_SPSkillMenu_Init`, if the player closes and re-opens the menu with skill==5, the nightmare sound plays again. Some players might find repeated sound plays annoying. Mitigation: Accept this as intended feedback (every time you open the menu with nightmare selected, you hear it), or store the previous skill and only play if it changed.

4. **Hardcoded Layout Brittle to Resolution Changes:** All positions and dimensions assume 640×480. If a mod or port changes resolution, this menu would break (items off-screen, overlapping, etc.). Mitigation: Q3A's modding community didn't change the virtual resolution, so this was never an issue in practice.

5. **Global State Prevents Menu Reuse:** Only one instance of `skillMenuInfo` exists. If somehow the skill menu were pushed twice, the second push would overwrite the first's state. In practice, the menu stack and callback structure prevent this, but the design is not reentrant. Modern UI frameworks avoid this via instance-per-widget.
