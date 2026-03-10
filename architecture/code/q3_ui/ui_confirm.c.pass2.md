# code/q3_ui/ui_confirm.c — Enhanced Analysis

## Architectural Role

This file provides reusable modal dialog infrastructure for the legacy Q3A UI VM (`q3_ui`), serving as a bridge between application code needing confirmation (e.g., "disconnect?", "exit game?") and the q3_ui menu framework. It is a **modal overlay controller** that integrates tightly with the menu stack management system (`UI_PushMenu`/`UI_PopMenu`) and leverages the proportional text rendering and widget event callback patterns established throughout q3_ui. The file demonstrates idiomatic event-driven UI patterns from the era: callback-based result delivery, singleton state reuse, and keyboard-first input handling optimized for both gamepad and keyboard.

## Key Cross-References

### Incoming (who depends on this file)

- **q3_ui module files** (implicit): Other modules in `code/q3_ui/` call `UI_ConfirmMenu()`, `UI_ConfirmMenu_Style()`, or `UI_Message()` to prompt the user. Examples include disconnect confirmations, exit-game confirmations, and informational alerts.
- **UI callback mechanism**: The `action` callback parameter is invoked by callers to receive the user's yes/no result asynchronously, decoupling the confirmation initiator from the menu system.

### Outgoing (what this file depends on)

- **q3_ui menu framework** (same module):
  - `Menu_AddItem()`, `Menu_SetCursorToItem()`, `Menu_DefaultKey()`, `Menu_Draw()` — core menu widget management
  - `UI_ProportionalStringWidth()` — text layout calculations for centered button positioning
  - `UI_DrawNamedPic()`, `UI_DrawProportionalString()` — rendering primitives
  - `UI_PushMenu()`, `UI_PopMenu()` — menu stack lifecycle (part of `ui_atoms.c` or `ui_main.c`)
- **Engine syscalls** (via `trap_*` macros):
  - `trap_R_RegisterShaderNoMip()` — asset precaching for the frame artwork
  - `trap_GetClientState()` — determines fullscreen vs. overlay mode based on connection state
- **Global constants** (from `q_shared.h` and q3_ui headers):
  - Key codes (`K_TAB`, `K_LEFTARROW`, `K_RIGHTARROW`, `K_KP_*`)
  - Menu flags (`QMF_LEFT_JUSTIFY`, `QMF_PULSEIFFOCUS`)
  - Text style flags (`UI_LEFT`, `UI_CENTER`, `UI_INVERSE`, `UI_SMALLFONT`)
  - `color_red` global

## Design Patterns & Rationale

### 1. **Singleton State Reuse for Modal Overlay**
The `s_confirm` static structure is zeroed and reused for every confirmation invocation. This reflects a design constraint of the Q3A UI: **only one modal overlay can be active at a time**. Unlike a modern UI framework with a modal stack, q3_ui maintains a single "current menu frame" (pushed via `UI_PushMenu`). This simplifies state management at the cost of preventing nested confirmations.

### 2. **Event-Driven Callback Architecture**
Rather than blocking or returning a result immediately, `UI_ConfirmMenu()` accepts a function pointer (`action`) that is invoked asynchronously when the user selects YES/NO. This pattern is idiomatic for QVM-hosted UI modules: the UI runs frame-by-frame, processes input, and fires callbacks—never blocks. The `ConfirmMenu_Event()` handler is the **convergence point** for all user interactions, funneling them through a single `result` boolean.

### 3. **Proportional Layout Positioning**
The YES/NO button positions are calculated using `UI_ProportionalStringWidth()` to achieve **centered text layout** in a virtual 640×480 coordinate space:
```c
n1 = UI_ProportionalStringWidth( "YES/NO" );  // total width
l1 = 320 - ( n1 / 2 );                        // center offset
```
This ensures the buttons remain centered even as the font/resolution scales. It's a common pattern in this era of game UI design (pre-DPI-aware, pre-responsive).

### 4. **Keyboard-First Input with Game-Specific Hotkeys**
The `ConfirmMenu_Key()` handler aggressively remap inputs:
- Arrow keys → Tab (for menu navigation)
- 'Y'/'N' → Direct activation (single-keystroke confirmation)

This reflects the design priority: **gamepad/keyboard navigation must work smoothly without requiring a mouse**. The single-letter hotkeys are essential for accessibility in a fast-paced game context.

### 5. **Polymorphic Draw Function via Function Pointers**
The `menu.draw` field is set to either `ConfirmMenu_Draw` or `MessageMenu_Draw` depending on the variant (yes/no vs. message box). Both draw functions:
- Render the frame artwork
- Render their text (question string vs. multi-line array)
- Call `Menu_Draw()` to render buttons
- Optionally call `s_confirm.draw()` for custom overlays

This is a form of **composition-based polymorphism** (C-style function pointers), avoiding subclassing.

### 6. **Connection State Awareness**
```c
if ( cstate.connState >= CA_CONNECTED ) {
    s_confirm.menu.fullscreen = qfalse;  // Overlay mode (show game behind)
} else {
    s_confirm.menu.fullscreen = qtrue;   // Fullscreen mode (main menu)
}
```
This distinction allows the same confirmation UI to behave appropriately in two contexts: during gameplay (overlay), or in the main menu (fullscreen). It's a practical affordance that reduces cognitive friction—the game world remains visible during in-game confirmations.

## Data Flow Through This File

1. **Initialization** (per invocation):
   - Caller invokes `UI_ConfirmMenu()` or `UI_ConfirmMenu_Style()` with a question/message, style flags, and result callback.
   - `s_confirm` is zeroed; layout calculations position YES/NO buttons proportionally.
   - Menu items are created and added to the menu framework.
   - `UI_PushMenu()` installs the confirmation as the active menu frame.

2. **Per-Frame Rendering** (inside the UI VM loop):
   - The menu framework calls `s_confirm.menu.draw()` (either `ConfirmMenu_Draw` or `MessageMenu_Draw`).
   - The draw function renders the background frame, text, buttons, and optional custom overlay.

3. **Input Processing** (when user presses a key):
   - The menu framework calls `s_confirm.menu.key()` (always `ConfirmMenu_Key`).
   - Key handler optionally remaps (arrows → Tab, Y/N → direct event).
   - If remapped to a button activation, `ConfirmMenu_Event()` is called directly.

4. **Result Delivery**:
   - `ConfirmMenu_Event()` determines the result boolean (YES=qtrue, NO=qfalse).
   - Calls `UI_PopMenu()` to remove the confirmation from the stack.
   - Invokes `s_confirm.action(result)` to notify the original caller.

## Learning Notes

### Idiomatic Patterns for Game UI (1999–2005 Era)

1. **QVM Sandbox Perspective**: This file never touches memory management (`malloc`/`free`), file I/O, or platform APIs. All interactions with the outside world go through `trap_*` syscalls. The Q3A engine treats QVM code as untrusted; this module is completely sandboxed.

2. **Virtual Screen Coordinates**: Game UIs of this era do not adapt to arbitrary screen sizes or DPI. Instead, they work in a fixed 640×480 virtual space and let the renderer scale it. Modern engines use responsive/reactive layouts; Q3A uses **fixed proportional scaling**.

3. **Modal Simplicity**: Q3A's UI has no modal stack or dialog nesting. A confirmation is either the topmost menu (active) or not. This is simpler than a full UI framework (e.g., Win32 message boxes), but it avoids the complexity of managing layered modality.

4. **Callback Over Coroutines**: The confirmation result is delivered via a callback function pointer, not a return value or coroutine yield. This is pre-async/await, and it's idiomatic for event-driven C code.

5. **Proportional Typography**: Rather than pixel-perfect layout, the UI calculates positions based on rendered string widths. This is robust to font changes and scaling without needing a layout engine.

### Modern Engine Comparisons

- **Modern UI frameworks** (e.g., Unity uGUI, Unreal Slate): Use data-binding, immediate-mode rendering, or retained-mode scene graphs with automatic layout. Q3A is purely imperative and frame-driven.
- **Modal management**: A modern UI framework might use a modal stack (`pushModal()`, `popModal()`), allowing nested dialogs. Q3A's single-modal-at-a-time design is simpler but less flexible.
- **Input routing**: Modern frameworks often use event bubbling or focus capture. Q3A's menu framework uses a linear menu item list with explicit focus management.

## Potential Issues

1. **Typo in function comment** (line 123): `MessaheMenu_Draw` should be `MessageMenu_Draw`. Non-functional but a documentation error.

2. **Single Modal Limitation**: The singleton `s_confirm` design precludes nested confirmations. If a callback invoked by `s_confirm.action()` itself calls `UI_ConfirmMenu()`, the first dialog's state will be overwritten. This is unlikely in practice (the callback usually dismisses the menu and returns to a prior menu), but it's a subtle constraint.

3. **No Validation of Callback**: If `s_confirm.action` is NULL, the code silently does nothing after popping the menu. This is safe but means the caller must always provide a callback (or provide a no-op). A modern framework might enforce non-null via type system.

4. **Hard-Coded Layout**: Button positions and text Y-coordinates (264, 265, 280, 288) are magic numbers. No centralized layout constants or configuration. This is normal for this era, but it makes the UI fragile to viewport size changes or accessibility requirements (e.g., larger text for visually impaired players).

---
