# code/win32/win_input.c — Enhanced Analysis

## Architectural Role

This file is the **Win32-specific input translator** layer within the platform abstraction (`code/win32/`). It multiplexes three independent hardware APIs (Win32 raw mouse, DirectInput, Windows Multimedia joystick API, and MIDI) into a unified event stream via `Sys_QueEvent`. The file sits at a critical juncture: hardware ↔ platform layer ↔ abstract engine event queue (`sys_events_t`), enabling the engine core (`qcommon`, `client`) to remain input-API-agnostic.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/win32/win_main.c`** — Calls `IN_Init` (once at startup), `IN_Frame` (per-frame loop), `IN_Shutdown` (on exit); calls `IN_MouseEvent` from the Win32 window procedure (`MainWndProc`) for raw WM_MOUSEMOVE/button events when DirectInput is inactive.
- **`code/client/client.h`** — Via `cls` global (reads `cls.keyCatchers` to suppress mouse in console mode; shared via `#include`).
- **`code/qcommon/common.c`** — Indirectly; receives all queued `SE_KEY`, `SE_MOUSE` events from this layer into the global `sys_events` FIFO.

### Outgoing (what this file depends on)

- **`Sys_QueEvent`** (platform layer) — Primary output. Called once per input event (button, wheel tick, joystick direction change, MIDI note) with timestamped `sysEvent_t` structs.
- **`code/qcommon/{cvar,cmd,common}`** — `Cvar_Get`, `Cvar_Set`, `Cvar_VariableValue`, `Cvar_VariableString`, `Cmd_AddCommand`, `Com_Printf`, `Com_Memset`.
- **Win32 APIs** (dynamically loaded) — `dinput.dll` (DirectInput, loaded at runtime via `LoadLibrary`); `winmm.dll` (implicit, joystick/MIDI via system libs); Win32 native `GetCursorPos`, `SetCursorPos`, `ClipCursor`, `ShowCursor`, `SetCapture`, `ReleaseCapture`.
- **`code/win32/win_local.h`** — `g_wv` struct (`hWnd`, `hInstance`, `osversion`, `sysMsgTime`).

## Design Patterns & Rationale

### 1. **Fallback Chain for Mouse Input**
Three-tier hierarchy: DirectInput (preferred, exclusive, lowest-latency) → Win32 raw mouse (fallback) → error handling. `IN_StartupMouse` detects OS version, skips DirectInput on NT 4.0 (known bug #50), and sets `mouseStartupDelayed` to retry after window is created. This reflects mid-2000s Windows ecosystem fragmentation.

### 2. **Dynamic Loading of DirectInput DLL**
DirectInput (`dinput.dll`) is loaded at runtime, not statically linked. `hInstDI` caches the module handle; `pDirectInputCreate` is retrieved via `GetProcAddress`. This allows graceful degradation if the library is missing or incompatible on older systems.

### 3. **Custom Data Format (MYDATA)**
Rather than using `c_dfDIMouse` or `c_dfDIMouse2`, the file defines a custom `DIOBJECTDATAFORMAT` array to map DirectInput raw data into a struct with X/Y/Z axes + 4 buttons. This is more flexible for supporting variable hardware layouts but requires hand-coded offset/type mappings.

### 4. **Dual-Mode Input for DirectInput Mouse**
- **Buffered reads** (`IDirectInputDevice_GetDeviceData`) for low-frequency events (buttons, mouse wheel) with exact timestamps.
- **Snapshot reads** (`IDirectInputDevice_GetDeviceState`) for continuous motion (axis deltas).  
This hybrid approach avoids missing discrete events while capturing smooth motion data.

### 5. **Deferred Motion Events**
Mouse motion (`IN_MouseMove`) is NOT queued directly in `IN_DIMouse`; instead, raw deltas are returned to `IN_Frame`, which decides when and how to post motion events. This decouples polling (potentially high-frequency) from event scheduling.

## Data Flow Through This File

```
Hardware → API → Event Queue → Engine
  ↓         ↓         ↓          ↓
[Mouse]  [DI] → [Sys_QueEvent] → [sys_events FIFO]
[JOY]    [WinMM]  [SE_KEY]        (read by qcommon main loop)
[MIDI]   [WinMM]  [SE_MOUSE]
```

- **DirectInput path**: `IN_DIMouse` drains `GetDeviceData` buffer for buttons/wheel → `Sys_QueEvent(SE_KEY, K_MOUSE1, ...)`. Raw axis snapshot → return as `*mx, *my` to caller.
- **Win32 mouse path**: `IN_Win32Mouse` samples cursor position, recenters, computes delta → return to caller for motion event posting.
- **Joystick path**: `IN_JoyMove` polls `joyGetPosEx` once per frame, compares axes/buttons/POV against last state, fires `SE_KEY` events for transitions. Optionally emits `SE_MOUSE` for ball-axis continuous movement.
- **MIDI path**: System-thread callback (`MidiInProc`) intercepts `MIM_DATA` messages, decodes note on/off, directly calls `MIDI_NoteOn`/`MIDI_NoteOff` → `Sys_QueEvent(SE_KEY, K_AUX*)`.

## Learning Notes

### Engine-Era Conventions (2005)

- **No abstractions over platform APIs**: File directly uses Win32/DirectInput/WinMM calls rather than a cross-platform input abstraction layer (common in modern engines).
- **Manual state tracking**: `s_wmv`, `joy`, `s_midiInfo` are all file-static, initialized once, never cleaned up mid-session. No object-oriented encapsulation.
- **Zero-copy event design**: Events are created and immediately queued; no intermediate buffering or transformation pass.

### DirectInput as "Premium" Path

DirectInput was the high-performance input API on Windows XP-era gaming PCs, offering low-latency, buffered input with precise timestamping. Modern Windows games use XInput (Xbox controller abstraction) or raw pointer input. This file's three-API approach reflects the fragmentation of that era.

### MIDI Thread Safety Assumption

The `MidiInProc` callback runs on a system MIDI delivery thread, yet it calls `Sys_QueEvent` without locks. This works only if the main game loop is single-threaded and `Sys_QueEvent` is thread-safe (likely a simple ring-buffer enqueue with no shared mutable state except the write pointer). Modern engines would protect this with a mutex or lock-free queue.

### Mouse/Console Suppression Pattern

In `IN_Frame`, if `cls.keyCatchers & KEYCATCH_CONSOLE` is true (console is open) and `r_fullscreen == 0` (windowed mode), the mouse is deactivated. This is a UI-driven input suppression pattern: the input system doesn't know about menus/console; it asks the client layer "should I be active?"

## Potential Issues

1. **MIDI Callback Race Condition**: `MidiInProc` can be invoked concurrently with `IN_Frame` on different threads. If `Sys_QueEvent` is not internally thread-safe, this can corrupt the event queue. (Likely mitigated in practice if `Sys_QueEvent` uses interlocked writes, but not explicit in the code.)

2. **DirectInput Device Recovery**: If the DirectInput mouse device is lost (e.g., due to alt-tab, window destruction), `IN_ActivateDIMouse` attempts re-acquire and reinitializes via `IN_InitDIMouse` on failure. Between the re-init call and the next `IN_Frame`, input may stall or repeat. This is a known brittle transition (bugzilla #50 referenced in code comments).

3. **Frame-Rate Dependency of Motion Events**: Mouse motion is posted only once per `IN_Frame`, even if DirectInput buffered 10 wheel events in the same frame. High-frequency motion is inherently quantized to frame boundaries, which can cause hitches in fast-paced games on variable frame-time systems.

4. **No Input Timestamp Alignment**: DirectInput provides `DIDEVICEOBJECTDATA::dwTimeStamp` (system time), but `IN_Frame` motion events are queued with no explicit timestamp. This could cause motion to be attributed to slightly different game ticks than simultaneous button presses.
