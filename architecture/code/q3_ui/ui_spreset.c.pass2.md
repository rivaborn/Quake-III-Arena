# code/q3_ui/ui_spreset.c — Enhanced Analysis

## Architectural Role

This file implements a single-player menu state within the **q3_ui VM**'s menu stack system. It acts as a defensive confirmation gate before `UI_NewGame()` is invoked, which triggers a full game state reset. The reset menu exemplifies Q3A's layered UI architecture: it operates entirely within the VM sandbox via `trap_*` syscalls, participates in the stack-managed menu hierarchy, and demonstrates conditional rendering logic (fullscreen vs. overlay) based on engine connection state.

## Key Cross-References

### Incoming (who depends on this)
- **Activated by:** Single-player level menu (`code/q3_ui/ui_spLevel.c`) when player chooses a "reset" action
- **Called from:** q3_ui VM entry point in `code/q3_ui/ui_main.c` dispatcher (indirectly via menu activation)

### Outgoing (what this file depends on)
- **Menu framework:** `Menu_Draw`, `Menu_DefaultKey`, `Menu_AddItem`, `Menu_SetCursorToItem` from `code/q3_ui/ui_qmenu.c`
- **UI utilities:** `UI_PushMenu`, `UI_PopMenu`, `UI_DrawNamedPic`, `UI_DrawProportionalString`, `UI_ProportionalStringWidth` from `code/q3_ui/ui_atoms.c`
- **Game state:** `UI_NewGame()` from `code/q3_ui/ui_gameinfo.c` (performs the actual reset); `UI_SPLevelMenu()` from `code/q3_ui/ui_spLevel.c` (navigation target)
- **Engine syscalls:** `trap_R_RegisterShaderNoMip` (renderer), `trap_Cvar_SetValue` (cvar write), `trap_GetClientState` (connection state query)

## Design Patterns & Rationale

**Menu Stack Navigation Pattern:**
The file demonstrates the menu hierarchy design: YES action invokes `UI_PopMenu()` twice (popping the reset dialog + parent level menu) then `UI_SPLevelMenu()` to re-initialize with updated state. This is safer than direct state mutation because it ensures the UI re-initializes consistently. Contrast with NO, which pops once and returns to the parent menu unchanged.

**Defensive Default:**
Cursor defaults to NO (line 182, `Menu_SetCursorToItem` on `&s_reset.no`). This reduces accidental resets — a common UX pattern in destructive operations. The entire menu is designed as a interrupt handler: acknowledge Yes/No, then continue.

**Conditional Fullscreen Logic:**
Lines 160–168 check `cstate.connState >= CA_CONNECTED` to decide whether the menu is fullscreen or overlay. This reflects Q3A's dual-mode architecture: menus can float over an active game (spectator mode, disconnect during match) or occupy the full screen (main menu). The distinction is enforced at the menu framework level, not here, but this file makes the decision based on engine state.

**Shader Pre-Registration (Caching):**
`Reset_Cache()` calls `trap_R_RegisterShaderNoMip()` before the menu is pushed. This follows the renderer's deferred-loading pattern: all assets that will be drawn must be registered before rendering begins, allowing the renderer to batch-compile and optimize multi-pass shaders. No runtime shader lookup happens during draw calls.

**Proportional String Layout:**
Lines 144–151 compute pixel offsets using `UI_ProportionalStringWidth()` with constant gap widths (`PROP_GAP_WIDTH`). The UI operates in virtual 640×480 coordinates (inferred from architecture overview); the layout engine scales proportionally for different aspect ratios. This is a resolution-independent UI technique common in early 2000s engines.

## Data Flow Through This File

**Initialization Flow:**
```
UI_ResetMenu() [entry]
  → memset( &s_reset, 0 )
  → Reset_Cache() → trap_R_RegisterShaderNoMip()
  → trap_GetClientState() [query connection state]
  → Menu_AddItem() x2 [populate menu items]
  → UI_PushMenu( &s_reset.menu ) [activate on stack]
```

**Per-Frame Loop:**
```
Engine calls q3_ui VM dispatcher
  → Reset_MenuKey() or Reset_MenuDraw() [top of menu stack]
  → Yes/No activation → Reset_MenuEvent()
```

**Activation Flow (YES path):**
```
Reset_MenuEvent( &s_reset.yes, QM_ACTIVATED )
  → UI_PopMenu() [pop reset dialog]
  → UI_NewGame() [reset game state in engine]
  → trap_Cvar_SetValue( "ui_spSelection", 0 ) [reset level index to 0]
  → UI_PopMenu() [pop parent level menu]
  → UI_SPLevelMenu() [re-initialize level menu with clean state]
```

NO path simply pops once and returns.

## Learning Notes

**VM Boundary Discipline:**
Every interaction with the engine is mediated by `trap_*` syscalls (indexed dispatch; see `code/q3_ui/ui_syscalls.c`). This file never calls engine functions directly. Modern game engines often blur VM/engine boundaries; Quake III's rigid syscall boundary is a artifact of its security model (VMs run untrusted QVM bytecode in a sandbox with `dataMask`-gated memory access). This explains why even trivial reads like "is the client connected?" go through `trap_GetClientState()`.

**Callback-Driven UI:**
No polling loop here. Menu state is purely reactive: keys/activation trigger callbacks, which modify the menu stack. This is idiomatic to the era; modern engines tend toward immediate-mode UI (ImGui style). The per-frame `Reset_MenuDraw()` callback is invoked by the framework, not by this file.

**Coupling Through Menu Stack:**
This file is loosely coupled to its caller (the level menu) via the menu stack abstraction. It doesn't know *which* menu opened it; it only knows it can pop and push menus. This enables reusable UI components.

**Parallel with cgame:**
The cgame VM uses the same VM syscall pattern (`code/cgame/cg_syscalls.c`), suggesting a consistent architectural style across all VMs (game, cgame, ui).

## Potential Issues

- **Dangling Fullscreen State:** If `trap_GetClientState()` fails or returns garbage, the fullscreen flag could be incorrectly set. No error handling visible. (Unlikely in practice; `GetClientState` is a core syscall.)
- **Menu Stack Underflow:** If NO is selected when the menu stack depth is 1 or 0, `UI_PopMenu()` could cause a crash. Assumed safe by contract (caller must have pushed the reset menu onto an existing stack).
- **Race Condition in YES Path:** `UI_NewGame()` is called *before* popping the level menu. If `UI_NewGame()` triggers an async operation or exception, the stack could be left inconsistent. Assumed synchronous and infallible.
