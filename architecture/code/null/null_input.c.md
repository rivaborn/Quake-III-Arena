# code/null/null_input.c

## File Purpose
Provides a no-op (null) implementation of the platform input subsystem for Quake III Arena. All functions are empty stubs, used when building a headless/dedicated server or a platform-agnostic null client where no actual input handling is needed.

## Core Responsibilities
- Stub out `IN_Init` so the engine's input initialization path can be called safely with no effect
- Stub out `IN_Frame` so the per-frame input polling path executes without error
- Stub out `IN_Shutdown` so the input teardown path completes cleanly
- Stub out `Sys_SendKeyEvents` so the OS key-event pump is a no-op

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### IN_Init
- Signature: `void IN_Init(void)`
- Purpose: Initialize the input subsystem. No-op in this null implementation.
- Inputs: None
- Outputs/Return: void
- Side effects: None
- Calls: None
- Notes: Real platform implementations (e.g., `win32/win_input.c`, `unix/linux_joystick.c`) register cvars, open device handles, and set up mouse/keyboard state here.

### IN_Frame
- Signature: `void IN_Frame(void)`
- Purpose: Per-frame input polling hook. No-op in this null implementation.
- Inputs: None
- Outputs/Return: void
- Side effects: None
- Calls: None
- Notes: Real implementations read joystick axes and accumulate mouse deltas into `cl.mouseDx`/`cl.mouseDy`.

### IN_Shutdown
- Signature: `void IN_Shutdown(void)`
- Purpose: Tear down the input subsystem. No-op in this null implementation.
- Inputs: None
- Outputs/Return: void
- Side effects: None
- Calls: None
- Notes: Real implementations release device handles and unregister callbacks.

### Sys_SendKeyEvents
- Signature: `void Sys_SendKeyEvents(void)`
- Purpose: Pump OS-level keyboard/mouse events into the engine key queue. No-op in this null implementation.
- Inputs: None
- Outputs/Return: void
- Side effects: None
- Calls: None
- Notes: Real implementations call `Key_Event` / `Com_QueueEvent` to feed events from the OS message loop or X11/evdev into the engine. Declared as a `Sys_*` function, meaning it is a platform abstraction boundary.

## Control Flow Notes
`IN_Init` is called during client startup (`CL_Init`). `IN_Frame` and `Sys_SendKeyEvents` are called each client frame in the main loop before command processing. `IN_Shutdown` is called during client teardown. In this null file all four are pass-through no-ops, so the engine proceeds through its normal call sites without branching.

## External Dependencies
- `../client/client.h` — pulls in the full client subsystem header (key types, `clientActive_t`, `kbutton_t`, input function declarations, etc.), though none of those symbols are actually used here.

**Defined elsewhere (symbols the real implementation would use):**
- `Key_Event` / `Com_QueueEvent` — engine key/event queue (defined in `cl_keys.c` / `common.c`)
- `cl.mouseDx`, `cl.mouseDy` — mouse delta accumulators in `clientActive_t` (defined in `cl_main.c`)
- Platform OS handles — not applicable in null build
