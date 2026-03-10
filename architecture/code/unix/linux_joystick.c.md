# code/unix/linux_joystick.c

## File Purpose
Implements Linux-specific joystick input handling for Quake III Arena, translating Linux kernel joystick events (`/dev/jsN`) into the engine's internal key event system. It bridges the Linux joystick driver's event model to Quake's polling-style input pipeline.

## Core Responsibilities
- Open and initialize the first available joystick device (`/dev/js0`–`/dev/js3`)
- Drain the joystick event queue each frame
- Dispatch button press/release events directly as `SE_KEY` events
- Convert axis values to a bitmask and synthesize key press/release events for axis transitions
- Map 16 axes to virtual key codes (`joy_keys[]`)

## Key Types / Data Structures
None defined in this file; uses `struct js_event` from `<linux/joystick.h>`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `joy_keys` | `int[16]` | global | Maps axis direction bits to Q3 key codes (arrow keys + `K_JOY16`–`K_JOY27`) |
| `joy_fd` | `static int` | static | File descriptor for the open joystick device; `-1` = not active |
| `in_joystick` | `extern cvar_t *` | extern (defined in `linux_glimp.c`) | CVar enabling joystick input |
| `in_joystickDebug` | `extern cvar_t *` | extern | CVar for joystick debug output |
| `joy_threshold` | `extern cvar_t *` | extern | Dead-zone threshold for axis-to-key translation |

## Key Functions / Methods

### IN_StartupJoystick
- **Signature:** `void IN_StartupJoystick( void )`
- **Purpose:** Detects and opens the first available joystick device; queries and logs its capabilities.
- **Inputs:** None (reads `in_joystick` CVar)
- **Outputs/Return:** void
- **Side effects:** Sets `joy_fd`; writes to console via `Com_Printf`; flushes init events from the device queue via `read()`
- **Calls:** `open`, `read`, `ioctl` (`JSIOCGAXES`, `JSIOCGBUTTONS`, `JSIOCGNAME`), `strncpy`, `Com_Printf`
- **Notes:** Tries `/dev/js0`–`/dev/js3` in order; stops at the first successful open. Init events (flagged `JS_EVENT_INIT`) are discarded before returning.

### IN_JoyMove
- **Signature:** `void IN_JoyMove( void )`
- **Purpose:** Per-frame joystick poll — drains the non-blocking event queue, updates axis state, and fires key events for button and axis transitions.
- **Inputs:** None (reads `joy_fd`, `joy_threshold` CVar)
- **Outputs/Return:** void
- **Side effects:** Updates `static axes_state[16]` and `static old_axes`; calls `Sys_QueEvent` for each button and axis state change
- **Calls:** `read`, `Sys_QueEvent`, `Com_Printf`
- **Notes:** Axis values are normalized to `[-1, 1]` (divided by 32767). Each axis maps to two bits in the `axes` bitmask (negative direction = even bit, positive = odd bit). Key events are only fired on state *transitions* (old vs. new bitmask). Button events use `K_JOY1 + event.number` directly.

## Control Flow Notes
- `IN_StartupJoystick` is called once during input subsystem initialization (`IN_Init`, defined in `linux_glimp.c`).
- `IN_JoyMove` is called every frame from `IN_Frame` (also in `linux_glimp.c`), which is driven by the main client loop.
- No rendering or server involvement; purely an input-layer concern.

## External Dependencies
- `<linux/joystick.h>` — `struct js_event`, `JS_EVENT_BUTTON`, `JS_EVENT_AXIS`, `JS_EVENT_INIT`, `JSIOCG*` ioctls
- `<fcntl.h>`, `<sys/ioctl.h>`, `<unistd.h>`, `<sys/types.h>` — POSIX I/O
- `../client/client.h` — `cvar_t`, `Com_Printf`, key code constants (`K_LEFTARROW`, `K_JOY1`, etc.)
- `linux_local.h` — `Sys_QueEvent`, `sysEventType_t` (`SE_KEY`)
- `Sys_QueEvent` — defined in `unix_main.c` (not this file)
- `in_joystick`, `in_joystickDebug`, `joy_threshold` — defined/registered in `linux_glimp.c`
