# code/client/cl_keys.c — Enhanced Analysis

## Architectural Role

This file implements the **input dispatch and key binding subsystem** sitting at the OS→engine boundary. It receives raw platform key events (via `CL_KeyEvent`/`CL_CharEvent` from `win_input.c`, `linux_joystick.c`, etc.) and routes them to multiple engine subsystems: the console, UI VM, cgame VM, or the server as bound commands. The global `keys[]` array maintains per-key state (down, repeats, binding string), making input visible across the entire engine. Essentially, this file is the **semantic layer above raw OS keycodes**—translating key names, managing bindings, and multiplexing events.

## Key Cross-References

### Incoming (who depends on this)
- **Platform layer** (`win_input.c`, `unix/linux_joystick.c`) → calls `CL_KeyEvent()`, `CL_CharEvent()` for every keystroke
- **Client main loop** (`cl_main.c`) → calls `Key_ClearStates()` on disconnect/mode transitions
- **Console** (`cl_console.c`) → renders via `Field_Draw()`; processes input through `Field_*` functions; uses `g_consoleField` global
- **UI/cgame VMs** → syscall wrappers like `CL_Key_SetBinding`, `CL_Key_GetBinding` dispatched by `VM_Call()`
- **Chat system** → uses `chatField` and `Message_Key()` for in-game messaging

### Outgoing (what this depends on)
- **qcommon**: `Cmd_AddCommand`, `Cbuf_AddText`, `Cvar_Set`, `FS_Printf` (config save), `Com_Error`, `Z_Free`
- **Client subsystems**: `Con_ToggleConsole_f`, `CL_AddReliableCommand`, `CL_Disconnect_f`, `SCR_Draw*` (rendering), `S_StopAllSounds`
- **VM system**: `VM_Call(cgvm/uivm, ...)` for event dispatch
- **Globals**: `cls.keyCatchers` (bitmask router), `cls.realtime` (blink timing)
- **Platform**: `Sys_GetClipboardData()` for paste

## Design Patterns & Rationale

**Event dispatcher** — Raw OS keycodes → `CL_KeyEvent()` → routes to console/UI/cgame/game via `cls.keyCatchers` bitmask. This separates OS-specific input from engine semantics and allows multiple subsystems to be "active" simultaneously.

**Dual input streams** — Key-down events (non-printable: arrows, modifiers) handled by `Field_KeyDownEvent()`; character events (printable text) by `Field_CharEvent()`. This mirrors actual keyboard hardware and avoids conflating structural input (cursor moves) with semantic input (text).

**String-based command binding** — `keys[keynum].binding` holds command strings; `+button` pairs (down/up) interpreted specially for sub-frame server accuracy. Inherited from early Quake; enables runtime rebinding and mod customization.

**Field-reusable abstraction** — `field_t` struct centralizes scrolling, cursor tracking, and buffer management for console, chat, and UI input. Reduces duplication but creates tight coupling between console/chat logic and this file.

**History ring buffer** — `historyEditLines[COMMAND_HISTORY]` with `nextHistoryLine`/`historyLine` indices. Standard shell behavior; circular structure avoids dynamic allocation.

**Tradeoff:** Global state (`keys[]`, `g_consoleField`) rather than OOP—matches Q3's era, simplifies VM syscalls, but less modular. Input dispatch is bitmask-based, not hierarchical—simpler but less flexible for nested menus.

## Data Flow Through This File

```
OS Keystroke
  ↓
CL_KeyEvent(keynum, down, time)
  ├─ Update keys[keynum].down, .repeats counter
  ├─ [Down?] Check for +button binding → send keynum+time to server/usercmd
  ├─ [Down?] Route via cls.keyCatchers:
  │   ├─ KEYCATCH_CONSOLE → Console_Key() → modifies g_consoleField → Cbuf_AddText()
  │   ├─ KEYCATCH_MESSAGE → Message_Key() → modifies chatField → CL_AddReliableCommand()
  │   ├─ KEYCATCH_UI → VM_Call(uivm, UI_KEY_EVENT)
  │   └─ (None) → execute binding or queue to server
  └─ [Up?] Always → CL_AddKeyUpCommands() for -button pairs

CL_CharEvent(ch)
  ├─ Route via cls.keyCatchers:
  │   ├─ KEYCATCH_CONSOLE → Field_CharEvent(&g_consoleField)
  │   ├─ KEYCATCH_MESSAGE → Field_CharEvent(&chatField)
  │   └─ KEYCATCH_UI → VM_Call(uivm, UI_CHAR_EVENT)
  └─ cgame typically ignores char events

Field_CharEvent(field, ch)
  ├─ Handle ctrl shortcuts (ctrl-v paste, ctrl-h backspace, etc.)
  └─ Insert/overstrike ch; advance cursor; auto-scroll if at right edge
```

**Per-frame rendering:** `Field_VariableSizeDraw()` computes scroll offset to keep cursor visible, calls `SCR_Draw*` to render.

## Learning Notes

**Quake III idioms** — Global `keys[]` array, string-based binding system, and console as first-class input mode are era-specific (2000s FPS conventions). Modern engines use event queues, input action maps, and abstract input from text. The `+button` binding pattern is unique to Quake; Half-Life used similar schemes.

**Dual input streams** — Separating key-down (structural) from char-input (semantic) is still good practice; modern frameworks (SDL, GLFW) follow this.

**Text field without framework** — Pre-GUI toolkit era—basic ASCII, no IME, no clipboard length validation. Modern text input is much more complex.

A developer studying this learns input routing, field management, command binding, and how to expose input state to VMs safely via syscalls.

## Potential Issues

1. **Clipboard overflow** (HIGH) — `Field_Paste()` does not validate clipboard length before feeding to `Field_CharEvent()`. A malicious clipboard could overflow the field buffer and corrupt heap.

2. **Field buffer size assumption** (MEDIUM) — `Field_CharEvent()` uses `memmove()` without validating that the caller's buffer is actually `MAX_EDIT_LINE` bytes. Silent corruption if misused.

3. **History ring boundary** (LOW) — `Console_Key()` history navigation logic doesn't explicitly validate `historyLine` bounds when wrapping; could access uninitialized fields in edge cases.

4. **Key state persistence across errors** (LOW) — If an error occurs mid-keystroke, stuck keys are possible since `Key_ClearStates()` may not be called in all error paths.
