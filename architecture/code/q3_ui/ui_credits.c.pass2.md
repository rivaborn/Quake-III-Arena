# code/q3_ui/ui_credits.c — Enhanced Analysis

## Architectural Role

This file implements a terminal UI menu within the legacy base-Q3A UI VM subsystem (`q3_ui`). It exemplifies the callback-driven menu architecture: a simple screen with no interactive elements, no update logic, only a draw function and a key handler. The file sits atop the UI framework provided by `ui_atoms.c` and `ui_qmenu.c`, demonstrating how decoupled UI menus are from engine rendering—all draws go through proportional string wrappers that issue renderer syscalls, and all input reaches the menu via the engine's key-dispatch mechanism after the menu is pushed onto the stack.

## Key Cross-References

### Incoming (who depends on this file)
- Called by the main menu or post-game flow (likely from `ui_main.c`) via `UI_CreditMenu()` entry point
- The function must be `extern` in `ui_local.h` for visibility to the rest of the q3_ui VM
- Menu callbacks (`draw`, `key`) are dispatched by the UI frame loop in `ui_atoms.c` after `UI_PushMenu` is called

### Outgoing (what this file depends on)
- **`UI_DrawProportionalString`** / **`UI_DrawString`** — `ui_atoms.c` — abstractions over renderer syscalls (`trap_R_*`) that handle virtual-coordinate blitting to 640×480 space
- **`UI_PushMenu`** — `ui_atoms.c` — registers this menu's callbacks into the active menu stack; engine calls `draw` every frame and routes input to `key` until popped
- **`trap_Cmd_ExecuteText(EXEC_APPEND, ...)`** — syscall into engine (`ui_syscalls.c`) — queues a command in the engine's text buffer for execution
- **`color_white`**, **`color_red`** — defined in `ui_qmenu.c` or global UI state — color vectors used by all proportional string renders
- **`menuframework_s`**, **`K_CHAR_FLAG`**, **`PROP_HEIGHT`**, **`PROP_SMALL_SIZE_SCALE`**, **`SMALLCHAR_HEIGHT`** — from `ui_local.h` / `q_shared.h` — layout constants and menu struct definition

## Design Patterns & Rationale

**Callback-driven state machine.** The menu system invokes `draw` and `key` function pointers; there is no explicit tick/update. This is idiomatic to Quake III's two-phase architecture: the engine controls the frame loop, and UI modules register handlers. The credits menu has zero state (no scrolling offset, no animation), so a static draw suffices.

**Hard-coded, non-data-driven content.** Unlike the MissionPack UI (`code/ui`), which parses widget definitions from `.menu` script files at runtime, the legacy UI bakes credits as literal `UI_DrawProportionalString` calls. This trades flexibility for simplicity and avoids the overhead of a script parser and widget framework. Reflects the era of Q3A (2000) vs. Team Arena (2001).

**Quit-on-any-keypress.** By filtering out character events (`K_CHAR_FLAG`) and invoking `quit`, the screen becomes a true **terminal menu**—no navigation, no selection state. User must actively dismiss it. This pattern is common for splash screens and credits rolls in games.

**Proportional + monospace hybrid rendering.** The main credit lines use proportional fonts for readability; the final legal/contact line uses monospace (`UI_DrawString`), likely because it contains URLs and contact info that must align precisely.

## Data Flow Through This File

```
[Engine main menu / post-game] 
    → calls UI_CreditMenu()
        → memset s_credits (reset state)
        → assign draw & key callbacks
        → call UI_PushMenu()
            → [engine registers menu on stack]

[Every frame while credits are topmost]
    → engine calls s_credits.menu.draw()
        → issues UI_DrawProportionalString() calls
        → syscalls render backend to blit text (trap_R_*)

[User presses any non-character key]
    → engine calls s_credits.menu.key(key)
        → filters K_CHAR_FLAG
        → calls trap_Cmd_ExecuteText(EXEC_APPEND, "quit\n")
            → [engine command buffer queues quit]
        → [engine processes quit next frame, shuts down]
```

No circular state updates; credits are purely read-only output.

## Learning Notes

**Why this is minimal.** Quake III's VM syscall ABI removes all rendering details from the UI module. The credits file knows nothing about OpenGL, virtual screens, or font rasterization—it delegates entirely to `trap_R_*` wrappers. This tight abstraction boundary is why a credits screen needs only ~130 lines. Modern engines would need asset loading, font systems, and so forth.

**Contrast with MissionPack UI.** The `code/ui` directory implements a full widget framework (`code/ui/ui_qmenu.c` equivalent, plus more) with buttons, sliders, and animated 3D player models. The legacy UI in `code/q3_ui` is more minimal and function-based. The credits module exemplifies this: no widget objects, just callbacks.

**Quake III UI philosophy.** The menu stack (`UI_PushMenu`, `UI_PopMenu`) decouples menu instances from the rendering system. Multiple menus can coexist on a stack; the topmost receives input. This is a classic stack-based UI pattern predating modern scene graphs.

**No scrolling or animation.** In a real game with longer credits, you'd add a frame counter and translate the `y` offset each draw. This file doesn't—the credits fit in a single screen. Shows how the rendering model is frame-by-frame, not curve-based.

**Syscall boundary discipline.** `trap_Cmd_ExecuteText` is a **messaging** syscall, not a **direct function call**. The engine executes `quit` on the next command-processing pass, not immediately. This decouples the UI VM's execution from the engine's command queue, preventing re-entrancy and stack corruption.

## Potential Issues

None directly inferable from the code alone, but **architectural constraints** to note:

1. **Hard-coded layout.** Adding or removing credits requires recompilation of the q3_ui module. A data-driven approach (like MissionPack) would allow mod-friendly changes.

2. **No bounds checking.** If credits text is too long to fit in virtual 640×480 space, rendering will clip silently. No error indication to the user.

3. **No localization.** All text is English; no i18n framework. Contrast with engine cvars and configstrings, which support arbitrary language bindings.

4. **Singleton menu instance.** `memset(&s_credits, 0, ...)` every time `UI_CreditMenu()` is called. If the credits screen is popped and re-pushed, the struct is re-zeroed. Not a bug, but assumes callbacks are lightweight and reentrant.
