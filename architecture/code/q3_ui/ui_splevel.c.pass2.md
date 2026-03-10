# code/q3_ui/ui_splevel.c — Enhanced Analysis

## Architectural Role

This file implements a **menu node** within Quake III's single-player campaign progression system, acting as a visual tier browser and level selector. It bridges the UI VM syscall boundary to retrieve arena/progression metadata from the engine, manages the menu widget lifecycle, and routes the player forward to either difficulty selection (`UI_SPSkillMenu`) or custom skirmish mode (`UI_StartServerMenu`). It enforces tier-based unlocking via cvar-persisted state (`ui_spSelection`) and integrates bot opponent data from the game module's arena definitions.

## Key Cross-References

### Incoming (who depends on this file)
- **Menu navigation flow:** Called by sibling menus or the main menu framework via `UI_SPLevelMenu()` public entry point
- **Menu framework (`ui_qmenu.c`):** `Menu_Draw`, `Menu_AddItem`, `Menu_SetCursorToItem` dispatch input events to registered callbacks
- **Menu stack:** `UI_PushMenu`/`UI_PopMenu` manage the recursive menu stack; this file contributes a menu layer
- **Confirmation dialog:** `UI_ConfirmMenu` surfaces in the reset flow, then re-invokes `UI_SPLevelMenu()` after confirmation

### Outgoing (what this file depends on)
- **Arena/progression data layer:** `UI_GetArenaInfoByNumber()`, `UI_GetSpecialArenaInfo()`, `UI_GetCurrentGame()`, `UI_GetNumSPTiers()`, `UI_GetNumSPArenas()`, `UI_GetBestScore()`, `UI_GetBotInfoByName()`, `UI_GetAwardLevel()` — all defined elsewhere in `code/q3_ui` (likely `ui_gameinfo.c`)
- **Target menus:** `UI_SPSkillMenu()` (difficulty selection), `UI_StartServerMenu()` (custom skirmish), `UI_PlayerSettingsMenu()` (player model/skin editor)
- **Game state reset:** `UI_NewGame()` clears all single-player progress variables
- **Widget framework:** `Bitmap_Init()` from `ui_qmenu.c` for bitmap button layout
- **Renderer syscalls:** `trap_R_RegisterShaderNoMip()` caches levelshot and completion badge shaders; `trap_R_SetColor()`, `UI_DrawHandlePic()`, `UI_DrawProportionalString()` for per-frame rendering
- **Sound syscalls:** `trap_S_StartLocalSound()` plays award notification sounds
- **Cvar syscalls:** `trap_Cvar_SetValue("ui_spSelection", ...)` and `trap_Cvar_VariableStringBuffer()` persist/retrieve player model and UI selection
- **Global data:** `ui_medalPicNames[]`, `ui_medalSounds[]` arrays (defined elsewhere, indexed by award level)
- **String utilities:** `Info_ValueForKey()` from `qcommon` parses bot/arena info strings (lightweight K-V format)

## Design Patterns & Rationale

1. **Menu Stack & Nested Callbacks:**  
   This file exemplifies the recursive menu pattern in id Tech 3: each menu owns a `menuframework_s` struct, registers event-driven callbacks for each widget, and can spawn child menus (`UI_ConfirmMenu`, `UI_PlayerSettingsMenu`) or navigate to siblings (`UI_SPSkillMenu`). This avoids global state pollution and allows menus to stack naturally.

2. **Deferred Re-initialization:**  
   The `reinit` flag in `levelMenuInfo` is set and consumed on the next `UI_SPLevelMenu_MenuDraw()` call. This allows a menu to request rebuild without immediately popping and repushing itself—a pragmatic workaround to avoid breaking the call stack mid-event.

3. **Cvar-Persisted UI State:**  
   `ui_spSelection` stores `(selectedArenaSet * ARENAS_PER_TIER + selectedArena)` as a single integer, persisting the player's last visited tier and level across game restarts. This is lightweight and plays well with the cvar serialization system.

4. **Tier-Based Progression Enforcement:**  
   Checks like `if (selectedArenaSet > currentSet) { levelMenuInfo.item_maps[n].generic.flags |= QMF_GRAYED; }` visually and functionally lock unearned tiers. The training and final tiers are hardcoded special cases with single-arena layouts, sacrificing DRY for simplicity.

5. **Asset Registration & Caching:**  
   Shader handles (`levelCompletePic[5]`, `botPics[7]`) are registered once during init and reused every frame. This amortizes the registration cost and avoids repeated renderer queries. `trap_R_RegisterShaderNoMip()` test-loads to confirm existence before caching the handle.

6. **Info String Parsing for Metadata:**  
   Bot opponent and arena definitions are encoded as compact info strings (e.g., `"bots cp lt grunt"` or `"map q3dm1"`). Parsing with `Info_ValueForKey()` avoids larger structured formats while maintaining shareability between engine and VM.

## Data Flow Through This File

**Input:**
- **Server state:** `currentSet`, `currentGame` computed from cvar queries in `UI_SPLevelMenu()` at menu init (read-once)
- **Cvar input:** `ui_spSelection` (player's last selected tier/arena pair), `model`, `name` (player settings)
- **Arena metadata:** Retrieved on-demand via `UI_GetArenaInfoByNumber()`, containing map name, bot list, and awards
- **Progression state:** `UI_GetCurrentGame()`, `UI_GetBestScore()`, `UI_GetBestAwardLevel()` provide earned medals

**Transform:**
- Parse arena info strings to populate `levelMenuInfo.levelNames[]`, `levelMenuInfo.botNames[]`, `levelMenuInfo.awardLevels[]`
- Resolve bot model icons via `PlayerIconHandle()` and levelshot textures via `trap_R_RegisterShaderNoMip()`
- Compute unlock state: if `selectedArenaSet <= currentSet`, enable selection; otherwise gray out
- Handle special tiers (training/final) as single-arena cases; normal tiers as 4-arena grids

**Output:**
- **To next menu:** Call `UI_SPSkillMenu(levelMenuInfo.selectedArenaInfo)` to pass selected arena to skill screen
- **To renderer:** Register all shaders and each frame draw levelshot thumbnails, bot portraits, player icon, awards, tier label
- **To sound:** Play award notification on click
- **To cvar:** Persist selected arena pair in `ui_spSelection`
- **To game logic:** Call `UI_NewGame()` to reset all progression (triggers server-side state reset)

## Learning Notes

1. **Menu-Driven Campaign Flow:**  
   Unlike modern engines with scene/timeline-based progression, Quake III's single-player flow is entirely menu-driven. Each screen (level select, difficulty, postgame) is a menu node with event-driven callbacks. This is idiomatic to early 2000s UI architecture and surprisingly flexible—conditional logic can easily gate progression or alter flows.

2. **Cvar as Lightweight Persistence:**  
   The use of `ui_spSelection` as a compact integer avoids serializing complex state. This pattern recurs throughout Q3A: cvars are the primary persistence mechanism for both engine and UI state. Modern engines would use structured save files or databases.

3. **Special-Case Encoding:**  
   Training and final tiers receive duplicate-heavy, special-case code (separate branches in `UI_SPLevelMenu_SetMenuItems()`) rather than data-driven tier definitions. This suggests the tier system evolved organically; a cleaner design would parameterize tier layout (1 arena vs. 4 arenas) in arena metadata.

4. **Syscall Isolation:**  
   This file never imports engine headers or calls engine functions directly—all communication is via `trap_*` indices. This enforces the VM sandbox. Porting or reimplementing the UI would only require changing the `trap_*` dispatch layer.

5. **Widget State in the Menu Struct:**  
   `levelMenuInfo` is a singleton holding both the menu framework and all derived state (scores, player model, awards). This collapses the model-view separation typical in modern UI frameworks but simplifies lifetime management for the stack-based menu system.

6. **Player Customization Bridging:**  
   The file caches player model/skin and reads it each frame (`trap_Cvar_VariableStringBuffer("model", ...)`), allowing on-the-fly appearance changes to be reflected in the menu. This is a simplicity-over-efficiency pattern: real-time polling rather than event listeners.

## Potential Issues

- **Fragile Static State:**  
  Global variables (`selectedArenaSet`, `selectedArena`) are mutable across menu pushes and pops. If another menu or the reset flow fails to reinitialize correctly, stale state can leak into a new session.

- **Hardcoded Array Limits:**  
  `levelMenuInfo.numBots < 7` and `levelMenuInfo.levelCompletePic[5]` are inflexible. Adding more bots or completion tiers requires code changes, not data.

- **Special-Case Tier Handling:**  
  The training/final tier branches duplicate the normal-tier code, violating DRY. Parameterizing tier metadata (layout, behavior) would consolidate logic.

- **Missing Bounds Checks:**  
  Functions like `UI_SPLevelMenu_SetMenuArena()` assume valid indices without defensive checks. Out-of-bounds arena numbers could corrupt `levelMenuInfo` arrays.

- **Deferred Reinit Footgun:**  
  The `reinit` flag pattern is fragile—if a draw call is skipped, reinit is deferred indefinitely. A more explicit state machine would prevent this class of bug.
