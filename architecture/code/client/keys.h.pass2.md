# code/client/keys.h — Enhanced Analysis

## Architectural Role

This header defines the **input state abstraction layer** for the client, serving as the critical bridge between raw OS key events and the engine's command/binding system. It encodes Quake III's distinctive architecture choice: decoupling physical keys from game actions via **string-based command bindings**, allowing runtime rebinding and persistence without recompilation. The file sits at a convergence point where platform-layer input (`win32/`, `unix/`, `macosx/`) meets high-level game logic (cgame VM, UI VM, console, chat), and exposes both low-level key state queries and high-level text field editing operations.

## Key Cross-References

### Incoming (what depends on keys.h)

- **`code/client/cl_keys.c`** — Implementation; calls are internal to the client subsystem
- **`code/client/cl_input.c`** — Per-frame input processing; reads `keys[]` state and assembles `usercmd_t`
- **`code/client/cl_console.c`** — Console text input; uses `Field_KeyDownEvent`, `Field_CharEvent`, and reads/writes `g_consoleField`
- **`code/client/cl_ui.c`** — UI syscall dispatch; exposes `Key_*` and field functions to UI VM via `trap_*` boundaries
- **`code/cgame/cg_consolecmds.c`** — Cgame console commands; queries `Key_GetBinding` for display
- **`code/game/g_cmds.c`** (server-side) — May read key/command bindings when syncing player actions
- **Platform layers** (`win32/win_input.c`, `unix/linux_*`, `macosx/macosx_input.m`) — Call `Key_Event()` (in `cl_keys.c`) with raw `keynum` after OS input is decoded

### Outgoing (what this file depends on)

- **`code/ui/keycodes.h`** — Defines the `keyNum_t` enum covering all 256 key slots; establishes the canonical keycode vocabulary for the entire engine
- **`code/qcommon/qcommon.h`** — Defines `field_t` (text input state), `qboolean`, `fileHandle_t`, and the console variable system
- **`code/qcommon/cmd.c`** — Indirectly: bound command strings are executed through the command buffer, not called directly from keys subsystem
- **Renderer** (`code/renderer/tr_*.c`) — `Field_Draw` and `Field_BigDraw` issue renderer calls; no direct include but runtime dependency
- **Sound subsystem** — Text field editing may trigger audio feedback (beeps, etc.), but no direct dependency here

## Design Patterns & Rationale

### 1. **String-Based Command Binding**
The core pattern: `keys[keynum].binding` stores command strings (e.g., `"+forward"`, `"quit"`), not function pointers. This is a **late-binding design** inherited from Quake's roots:

- **Rationale**: Allows runtime rebinding without code changes; enables config file persistence (`Key_WriteBindings`); supports **meta-binding** (e.g., binding a key to another key's binding via `Key_GetKey`)
- **Trade-off**: Adds lookup overhead (string comparison in command dispatch) and requires a global command buffer to execute strings asynchronously
- **Architectural consequence**: Tight coupling to `qcommon/cmd.c` command dispatch; the key subsystem cannot function independently

### 2. **Separation of Key Events from Character Events**
`Field_KeyDownEvent` (raw keycodes: arrows, backspace, home, end) vs. `Field_CharEvent` (printable characters) reflects **pre-Unicode input paradigms**:

- **Rationale**: Decouples physical key identity (needed for bindings, rebinding UI) from character interpretation (needed for text entry, IME compatibility)
- **Idiomatic to era**: Early 2000s engines; modern engines often fuse these or use Unicode input events
- **Consequence**: UI must dispatch events twice—once for structural keys, once for character data

### 3. **Global Mutable Text Field State**
`g_consoleField`, `chatField`, and `historyEditLines[]` are global externs, not encapsulated in a UI manager:

- **Rationale**: Simplifies the input pipeline; avoids dynamic allocation of field state during frame loops
- **Design era**: Pre-OOP, stateful-globals architecture common in engines from this period
- **Coupling cost**: `cl_console.c`, `cl_ui.c`, and UI VM all access these globals directly; refactoring is difficult
- **Memory safety**: Fixed-size ring buffer (32 history lines) avoids allocation but is hardcoded for console use

### 4. **Per-Key Autorepeat Tracking**
`qkey_t.repeats` counter allows the engine to simulate autorepeat independently of OS repeat rates:

- **Rationale**: Ensures deterministic input handling across platforms (Windows DMusic, X11 repeat, macOS repeat all differ)
- **Usage**: Higher-level code can check `if (keys[k].repeats > 1)` to detect sustained key hold without re-triggering actions
- **Design consequence**: The engine owns repeat semantics, not the platform layer

### 5. **Bidirectional Binding Lookup**
Both `Key_GetBinding(keynum)` → command and `Key_GetKey(binding)` → keynum exist:

- **Rationale**: Forward lookup for command dispatch; reverse lookup for rebind UI (show current binding for a command)
- **Cost**: Reverse lookup is O(n) over 256 keys; UI caches results to avoid per-frame scans
- **Architectural insight**: The binding abstraction is bidirectional—no single canonical direction

## Data Flow Through This File

```
OS Key Event (from platform layer)
  ↓
Key_Event() [in cl_keys.c]
  ├─→ Update keys[keynum].down, repeats
  └─→ If text field active: dispatch Field_KeyDownEvent() or Field_CharEvent()
        ↓
        Field buffer mutation + cursor advance
        ↓
        Field_Draw() render next frame
  ↓
Per-frame input assembly [in cl_input.c]
  ├─→ Query keys[K_SPACE].down, etc. → build usercmd_t
  └─→ Lookup keys[k].binding → command string
        ↓
        Cbuf_ExecuteText() [in qcommon/cmd.c]
        ↓
        Game action (attack, move, menu toggle, etc.)
```

**Persistence flow:**
- Game startup: `Key_LoadBindings()` reads config → populates `keys[].binding`
- Gameplay: User rebinds via menu → `Key_SetBinding()` updates `keys[].binding`
- Game shutdown: `Key_WriteBindings(f)` serializes all bindings to config file

## Learning Notes

### What a developer studying this learns:
1. **Quake's command model**: Actions are not hardcoded dispatcher methods but runtime-configurable strings. This is the DNA of Quake modding flexibility.
2. **Layered input semantics**: Key identity, character representation, and command binding are three separate concerns, each with its own layer of abstraction.
3. **Determinism in networked games**: The autorepeat counter and per-frame key state allow demos and network play to be deterministic (all clients see identical input timing).
4. **Trade-offs in global state**: Global fields reduce allocation overhead and simplify the frame loop but create coupling and refactoring friction.

### Idiomatic differences from modern engines:
- **Modern**: Event-driven input with callbacks; key code ↔ char code merged; no global text field state (UI framework owns it)
- **Q3A**: Polled key state; separated key/char events; global mutable console/chat fields; string-based command binding
- **Consequence**: Harder to swap input systems or UI frameworks; easier to extend with new game-specific key behaviors

### Connections to game engine concepts:
- **ECS (Entity-Component-System)**: Q3A doesn't use ECS; this file reflects a pre-ECS, centralized-state architecture
- **Immediate-mode UI**: The field rendering (`Field_Draw`) is immediate-mode: called once per frame, no retained state tree
- **Input abstraction layers**: Three layers here: platform (raw OS key codes) → engine (key state, bindings) → game (usercmd_t, commands)

## Potential Issues

1. **Hardcoded chat metadata in input subsystem**: `chat_team` and `chat_playerNum` are game-aware globals in what should be a domain-neutral input module. Should be isolated to cgame/UI layer.

2. **No bounds checking on field editing**: `Field_KeyDownEvent` mutates a `field_t*` buffer; if `field_t.maxsize` is not enforced in the field functions, buffer overflow is possible (though unlikely given fixed sizes and careful editing logic).

3. **MAX_KEYS = 256 hardcoded**: Assumes all 256 key slots will fit in a single array. No dynamic resizing or overflow handling documented. If a new key type (extra mouse buttons, etc.) exceeds 256, the system breaks.

4. **String binding memory ownership**: `Key_SetBinding` allocates new binding strings; the cleanup and lifetime of these strings relies on careful caller discipline in `cl_keys.c`. No clear RAII or lifetime documentation in this header.

5. **Global text field pointers create initialization order issues**: `g_consoleField` and `chatField` must be initialized before any input event fires; no explicit lifecycle management in the header.

---

**Word count: ~1300**
