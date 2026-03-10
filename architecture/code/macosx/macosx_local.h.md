# code/macosx/macosx_local.h

## File Purpose
This is the macOS platform-specific shared header for Quake III Arena, declaring the OpenGL window/display state, macOS-specific system function prototypes, and accessor macros for managing the OpenGL context across the macOS rendering and input subsystems.

## Core Responsibilities
- Declares `glwstate_t`, the central macOS OpenGL window state structure
- Exposes the global `glw_state` instance for use across macOS platform files
- Provides `OSX_*` macros for safe GL context get/set/clear operations
- Declares input system entry points (`macosx_input.m`)
- Declares system event and display utility functions (`macosx_sys.m`)
- Declares GL visibility/pause functions (`macosx_glimp.m`)
- Handles C/Objective-C/C++ compatibility via `#ifdef __cplusplus` guards

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `glwgamma_t` | struct | Stores per-display gamma ramp tables (red/green/blue arrays + display ID) |
| `glwstate_t` | struct | Top-level macOS GL window state: display handles, mode dictionaries, gamma tables, NSOpenGLContext, CGL context, window pointer, log file, frame counters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `glw_state` | `glwstate_t` | global (extern) | Singleton holding all macOS GL/display/window state; defined in `macosx_glimp.m` |
| `Sys_IsHidden` | `qboolean` | global (extern) | Tracks whether the application window is currently hidden; defined in `macosx_glimp.m` |
| `glThreadStampList` | `OTStampList` | global (extern, conditional) | OmniTimer stamp list for GL thread profiling; only present when `OMNI_TIMER` is defined |

## Key Functions / Methods

### Sys_InitInput / Sys_ShutdownInput
- Signature: `void Sys_InitInput(void)` / `void Sys_ShutdownInput(void)`
- Purpose: Initialize and tear down the macOS input system
- Inputs: None
- Outputs/Return: void
- Side effects: Allocates/frees OS-level input resources
- Calls: Defined in `macosx_input.m`
- Notes: Must be paired; called during engine init/shutdown

### Sys_SetMouseInputRect
- Signature: `void Sys_SetMouseInputRect(CGRect newRect)`
- Purpose: Constrains raw mouse delta input to a specific screen rectangle (used for windowed vs fullscreen mouse capture)
- Inputs: `newRect` — CG rectangle for mouse confinement
- Outputs/Return: void
- Side effects: Modifies OS mouse capture region
- Calls: Defined in `macosx_input.m`

### Sys_DisplayToUse
- Signature: `CGDirectDisplayID Sys_DisplayToUse(void)`
- Purpose: Returns the `CGDirectDisplayID` of the display the game should render to
- Inputs: None
- Outputs/Return: `CGDirectDisplayID`
- Calls: Defined in `macosx_input.m`

### Sys_QueEvent
- Signature: `void Sys_QueEvent(int time, sysEventType_t type, int value, int value2, int ptrLength, void *ptr)`
- Purpose: Enqueues a system event (key, mouse, etc.) into the engine event queue
- Inputs: timestamp, event type, two integer values, optional pointer payload
- Outputs/Return: void
- Side effects: Writes to the engine's event ring buffer
- Calls: Defined in `macosx_sys.m`

### Sys_Hide / Sys_Unhide
- Signature: `qboolean Sys_Hide(void)` / `qboolean Sys_Unhide(void)`
- Purpose: Hide or restore the game window (e.g., on app deactivation); updates `Sys_IsHidden`
- Inputs: None
- Outputs/Return: `qboolean` success flag
- Side effects: Modifies `Sys_IsHidden`, interacts with NSWindow/display

### Sys_PauseGL / Sys_ResumeGL
- Signature: `void Sys_PauseGL(void)` / `void Sys_ResumeGL(void)`
- Purpose: Pause/resume GL rendering (increments/decrements `glw_state.glPauseCount`)
- Inputs: None
- Side effects: Modifies `glw_state.glPauseCount`; defined in `macosx_glimp.m`

## Control Flow Notes
This header is included by all macOS platform `.m` and `.c` files. It sits at init/shutdown boundaries: `Sys_InitInput`/`Sys_ShutdownInput` are called during `Sys_Init`/`Sys_Shutdown`. `glw_state` is populated during renderer initialization (`macosx_glimp.m`) and the `OSX_*` macros are invoked around every GL context switch. `Sys_QueEvent` is called from the Cocoa event loop each frame.

## External Dependencies
- `qcommon.h` — engine core types (`qboolean`, `sysEventType_t`, `fileHandle_t`, etc.)
- `<ApplicationServices/ApplicationServices.h>` — `CGRect`, `CGDirectDisplayID`, `CGGammaValue`, etc.
- `<OpenGL/CGLTypes.h>` — `CGLContextObj`
- `<Foundation/NSGeometry.h>` (Obj-C only) — `NSRect`
- `macosx_timers.h` — `OTStampList` / `OmniTimer` profiling API (conditional on `OMNI_TIMER`)
- `NSOpenGLContext`, `NSWindow`, `NSEvent` — Cocoa objects (forward-declared or void-typed for C++ compatibility)
- `glw_state`, `Sys_IsHidden`, `glThreadStampList` — defined in `macosx_glimp.m` / `macosx_sys.m`
