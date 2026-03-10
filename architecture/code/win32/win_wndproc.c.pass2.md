# code/win32/win_wndproc.c — Enhanced Analysis

## Architectural Role

This file implements the **OS-to-engine event translation bridge** for the Windows platform layer. It acts as the critical junction where all raw OS messages are converted into the engine's event model (`Sys_QueEvent`), enabling the high-level engine code to remain platform-agnostic. The file also manages game-mode behavior enforcement (Alt-Tab suppression, mouse capture toggle) and window lifecycle state synchronization with persisted cvars, making it essential to the platform abstraction layer's contract.

## Key Cross-References

### Incoming (who depends on this file)
- **Windows OS**: Calls `MainWndProc` via the message pump after the window is registered (likely by `win_main.c` or `win_glimp.c`)
- **Platform initialization** (`win_main.c`): Registers this procedure during window class setup on `WM_CREATE`
- **Main game loop** (in `qcommon/common.c` via `Com_Frame`): Indirectly consumes events queued by this file through `Sys_QueEvent`
- **Input subsystem** (`win_input.c`): Reads `in_mouse` and `in_logitechbug` cvars defined/tested here; receives `IN_MouseEvent`, `IN_Activate` calls
- **Sound subsystem** (`win_snd.c`): Receives focus notifications via `SNDDMA_Activate` on `WM_ACTIVATE`

### Outgoing (what this file depends on)
- **Event queue** (`Sys_QueEvent` in `win_main.c`): All input events are queued asynchronously for main-loop processing
- **Client subsystem** (`client/client.h`): `Key_ClearStates`, `IN_Activate`, `IN_MouseEvent`, `cls.keyCatchers` (for console visibility)
- **Core engine** (`qcommon/`): `Cvar_Get`, `Cvar_SetValue`, `Cbuf_ExecuteText`, `Cbuf_AddText` (deferred console commands)
- **Win32 platform** (`win_local.h`): Reads/writes `g_wv` singleton (shared window/system state); calls `SNDDMA_Activate`
- **External cvars**: `r_fullscreen`, `vid_xpos`, `vid_ypos` (renderer, input modules)

## Design Patterns & Rationale

### 1. **Asynchronous Event Queuing**
Rather than calling input/client subsystems directly (except for focus state), this file queues events via `Sys_QueEvent` for deferred processing in the main loop. This decouples the OS's arbitrary message-pump thread from the game's simulation and rendering threads, eliminating cross-thread input state corruption.

### 2. **Platform-Specific Oddity Absorption**
The file absorbs Windows-era quirks:
- **Dual mouse wheel paths**: Both MSH_MOUSEWHEEL (Win95/NT3.51) and WM_MOUSEWHEEL (Win98+) are handled, with a fallback to the legacy message if DI input isn't used.
- **Logitech driver workaround**: The `flip` toggle (lines 241–249) treats successive wheel events as alternating press/release to compensate for Logitech's double-reporting bug—a pragmatic band-aid that keeps high-level code oblivious.
- **Keyboard layout neutrality via scan-code mapping**: The `s_scantokey[128]` table and `MapKey()` function separate Win32 scan codes from Quake key semantics; the extended-key bit (line 24) distinguishes numpad from cursor keys, a detail Windows buries in `lParam`.

### 3. **Game-Mode Enforcement**
Win32 alt-tab suppression is applied asymmetrically:
- **WinNT path** (line 53): Registers a `RegisterHotKey` hotkey to consume Alt+Tab OS-side.
- **Win9x path** (line 58): Sets `SPI_SCREENSAVERRUNNING` to trick the OS into thinking the screen saver is active, which disables Alt+Tab as a side effect.
This two-tier approach reflects OS-specific input control mechanisms circa 2005.

### 4. **Cvar-Based Window State Persistence**
On every `WM_MOVE` (lines 319–332), window position is stored in `vid_xpos`/`vid_ypos` cvars (with `CVAR_ARCHIVE` flag). This ensures the window position survives restart. The `AdjustWindowRect` call accounts for window chrome, a subtle detail that prevents drift between logical and screen coordinates.

### 5. **Focus-Driven Input Capture Toggling**
`VID_AppActivate` (lines 95–107) ties mouse capture to application focus: on `WA_INACTIVE`, mouse is released for system use; on regain, it's recaptured. This is standard game behavior but shows tight coupling between window activation and input handling.

## Data Flow Through This File

```
OS Message Pump
    ↓ (WM_* messages)
MainWndProc switch table
    ├─ Keyboard (WM_KEYDOWN/UP, WM_SYSKEYDOWN/UP)
    │   └─→ MapKey(lParam) → Sys_QueEvent(SE_KEY, keynum) → Main loop
    ├─ Mouse Wheel (WM_MOUSEWHEEL or MSH_MOUSEWHEEL)
    │   └─→ Sys_QueEvent(SE_KEY, K_MWHEELUP/DOWN) → Main loop
    ├─ Mouse Movement/Buttons (WM_LBUTTONDOWN, etc.)
    │   └─→ IN_MouseEvent(state) → Input subsystem
    ├─ Focus (WM_ACTIVATE)
    │   ├─→ VID_AppActivate(…)
    │   │   ├─→ Key_ClearStates()
    │   │   └─→ IN_Activate(bool)
    │   └─→ SNDDMA_Activate()
    ├─ Window Move (WM_MOVE)
    │   └─→ Cvar_SetValue("vid_xpos"/"vid_ypos") → Persisted on shutdown
    ├─ Fullscreen Toggle (Alt+Enter via WM_SYSKEYDOWN)
    │   └─→ Cbuf_AddText("vid_restart\n") → Renderer restart
    ├─ Close (WM_CLOSE)
    │   └─→ Cbuf_ExecuteText(EXEC_APPEND, "quit") → Shutdown
    └─ Create (WM_CREATE)
        ├─→ Cvar_Get() for vid_xpos, vid_ypos, r_fullscreen
        ├─→ RegisterWindowMessage(MSH_MOUSEWHEEL) for legacy wheel
        └─→ WIN_DisableAltTab() / WIN_EnableAltTab() based on fullscreen mode
```

**Key insight**: Input events flow *asynchronously* into the queue (`Sys_QueEvent`), while focus and structural changes (window position, fullscreen mode) are *immediately* synced to cvars or subsystem state. This two-tier approach balances responsiveness (focus) with determinism (queued input).

## Learning Notes

### Idiomatic to Early-2000s Game Engine Design
1. **Scan-code vs. keynum distinction** — Modern engines often avoid this complexity by using key enums directly, but this approach is portable across keyboard layouts if the table were configurable.
2. **Cvar-driven renderer parameters** — The file stores window position in cvars (`CVAR_ARCHIVE`), letting the renderer/config system own persistence. This is indirect but decouples I/O from input.
3. **No input batching** — Each OS message produces an immediate event queue entry. Modern engines batch input over a frame to reduce queue contention.

### Connections to Game Engine Concepts
- **Platform abstraction layer**: This file is the concrete Win32 implementation of what would be an abstract `PlatformInput` interface in modern engines.
- **Event-driven input model**: Contrasts with polling (e.g., DirectInput in `win_input.c`), allowing synchronous events (Alt+Enter, close) and asynchronous ones (typing) to coexist.
- **Focus-aware input capture**: Similar to modern exclusive fullscreen vs. windowed mode trade-offs; losing focus relinquishes mouse to the OS, a necessity in cooperative multitasking.

## Potential Issues

1. **Keyboard layout inflexibility** (code/win32/win_wndproc.c:122–170): The hardcoded `s_scantokey[128]` table assumes QWERTY. Non-English keyboard layouts (AZERTY, Dvorak) will produce incorrect key mappings. A proper fix would query the OS layout and remap at runtime—feasible but not done here.

2. **No multi-threaded input validation** (code/win32/win_wndproc.c:95–107): `g_wv.activeApp` is written here and potentially read by the renderer thread (see architecture context: "Optional SMP: front-end and back-end run on separate threads"). No synchronization primitive (mutex, atomic) guards this. Could cause stale focus state in SMP mode, though this is mitigated if the renderer only *reads* it per-frame.

3. **WM_DISPLAYCHANGE disabled** (code/win32/win_wndproc.c:271–277): The code to handle desktop resolution changes is `#if 0`'d out with a comment about `com_insideVidInit`. This means the game won't respond if the user changes resolution while playing—they must manually restart the renderer or quit. A missed UX improvement.

4. **No filtering of spurious Alt+Tab events** (code/win32/win_wndproc.c:53–59): The `RegisterHotKey` approach on WinNT consumes Alt+Tab but doesn't prevent the OS from still sending `WM_ACTIVATE` if another window somehow gets focus. Edge case, but could cause subtle state divergence.
