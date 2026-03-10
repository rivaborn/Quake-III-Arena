# code/ui/keycodes.h — Enhanced Analysis

## Architectural Role

This file defines the **input abstraction boundary** that unifies all hardware inputs into a single numeric keycode space. It bridges the platform-specific input layer (OS events, hardware enumeration) to the engine's high-level input and UI pipelines. By assigning unique `keyNum_t` values to keyboard, mouse, gamepad, and auxiliary inputs, Q3A achieves universal dispatch through a single `KeyEvent()` function, allowing the client and UI subsystems to remain agnostic to the underlying input mechanism.

## Key Cross-References

### Incoming (who depends on this file)
- **Client input layer** (`code/client/cl_keys.c`, `cl_input.c`): translates platform-specific key events into `keyNum_t`, manages `kbutton_t` state machines, and assembles `usercmd_t` button bitfields
- **UI/menu systems** (`code/q3_ui/ui_atoms.c`, `ui_main.c`, `code/ui/ui_atoms.c`): interpret `keyNum_t` for menu navigation, widget focus, and key binding storage
- **Platform input layers** (`code/win32/win_input.c`, `code/unix/linux_joystick.c`): perform the final OS→`keyNum_t` mapping from raw DirectInput/X11/joystick events
- **Console and debug systems**: used for rebindable command keys and developer input

### Outgoing (what this file depends on)
- **None**: This is a pure header with no external dependencies or includes. It is entirely self-contained.

## Design Patterns & Rationale

**1. Hardware Abstraction via Enum**
- Rather than expose raw OS keycodes (which differ across Windows/Linux/macOS and vary by input device), Q3A normalizes all inputs into a single `keyNum_t` space. This lets client code ignore input source entirely.
- **Tradeoff**: Fixed-size enumeration is easier to work with than dynamic key registries (used in modern engines), but creates a hard ceiling of 256 values.

**2. ASCII Compatibility**
- ASCII keys are anchored at their literal values: `K_TAB=9`, `K_ENTER=13`, `K_ESCAPE=27`, `K_SPACE=32`. This allows legacy code to treat keys as characters when convenient and avoids redundant mapping tables.
- Extended keys (modifiers, function keys, arrows) begin at value 128, leaving a clean separation.

**3. Flag-Based Event Polymorphism**
- `K_CHAR_FLAG` (1024) is OR'd into a `keyNum_t` at the call site to signal a **character event** rather than a **raw key press**. This allows menu systems to distinguish "user typed 'Q'" (text input for a name field) from "user pressed the key bound to reload_weapon" (game action), both dispatched through the same `KeyEvent()` path.
- **Rationale**: Avoids duplicating the key-handling callchain; a single switch statement can route both event types by testing the flag bit.

**4. Unified Input Dispatch**
- Every input (keyboard, mouse buttons, scroll wheel, 32 joystick buttons, 16 auxiliary) maps to a single enumeration. This allows `KeyEvent(keyNum_t code)` to be the universal entry point for all input into the UI and client systems, simplifying dispatch logic.

## Data Flow Through This File

1. **Platform → keyNum_t**
   - Windows: `win_input.c` receives `WM_KEYDOWN` / DirectInput events → maps to `K_*` constants
   - Linux: `linux_joystick.c` / X11 key handlers → map to `K_*` constants
   - macOS: `macosx_input.m` → map to `K_*` constants

2. **keyNum_t → Client Input State**
   - `cl_keys.c` receives `keyNum_t` from `KeyEvent()`, updates `kbutton_t` structs
   - Per-frame: `kbutton_t` state aggregates into `usercmd_t` button bitfields (impulse, attack, use, etc.)
   - Sent to server as part of snapshot delta

3. **keyNum_t → UI Dispatch**
   - UI VMs receive `keyNum_t` (optionally OR'd with `K_CHAR_FLAG`) via `trap_Key_SetCatcher` / `trap_Key_GetCatcher` syscalls
   - Menu handlers inspect the code: `if (key == K_ESCAPE) UI_PopMenu()` etc.
   - Character events (flag set) routed to text field handlers for name input, console input, chat

4. **UI Output → Game Action**
   - Menu selection triggers console commands (`exec config.cfg`, `map dm1`) dispatched via `cbuf` (client command buffer)
   - Menu-bound console variables affect gameplay (`cg_fov`, `sensitivity`, etc.)

## Learning Notes

**Engine Design Principle: Hardware Abstraction**
- Q3A demonstrates a foundational pattern: insulate gameplay and UI logic from platform-specific input enumeration. By the time code reaches `cl_keys.c` or the UI VM, all inputs are normalized, making the engine highly portable.

**Idiomatic to the Q3A Era (vs. Modern Engines)**
- **Old way**: Fixed-size enum of all possible inputs, hard ceiling at 256. Easy to reason about, low overhead, but brittle.
- **Modern way**: String-based or dynamic key registries (Unreal's `FInputActionValue`, Unity's new Input System). Extensible but higher complexity and indirect dispatch.

**Elegant Constraint Satisfaction**
- The ASCII alignment (`K_TAB=9`, etc.) is a clever trick: avoids redundant translation tables and allows some code to treat keys as characters. Modern C++ would use string interning or type-safe wrappers; Q3A uses raw integers.

**Event Polymorphism via Bitmask**
- Using `K_CHAR_FLAG` to distinguish event types within a single parameter is a pre-OOP pattern. It works but is hard to extend (e.g., adding mouse movement events would require yet another overlay bit). Modern engines use variant types or separate dispatch callbacks.

## Potential Issues

**Hard Keycode Ceiling**
- The enum + comment assertion ("K_LAST_KEY must be < 256") suggests space is tight. With 32 joystick buttons + 16 auxiliaries + mouse (5) + numpad (12) + function keys (15) + modifiers (4), the pool is nearly saturated. Any new input device (e.g., VR controllers, extra mouse buttons) would overflow the space. A workaround would require reassigning or removing existing codes.

**K_CHAR_FLAG Scope Ambiguity**
- The flag (1024) is larger than the max keycode (< 256), meaning character events occupy a disjoint value space. This works but is implicit and fragile: if code accidentally compares `if (key < K_CHAR_FLAG)` to distinguish char events, it will silently fail if a future refactoring expands the keycode range.

**No Mutual Exclusivity of Input Types**
- A single physical key maps to exactly one `keyNum_t`, but modern input systems allow **modifiers** (shift+click, ctrl+key). Q3A handles this via separate modifier keys (`K_SHIFT`, `K_CTRL`) that are pressed independently. Detecting chord combinations (`Ctrl+S`) requires explicit state checking in client code rather than being baked into the keycode itself. This is workable but shifts complexity to callers.
