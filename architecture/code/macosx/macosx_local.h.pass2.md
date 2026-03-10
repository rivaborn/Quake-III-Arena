# code/macosx/macosx_local.h — Enhanced Analysis

## Architectural Role

This header is the **macOS platform abstraction boundary** between the renderer and OS-level graphics/input services. It sits at a critical juncture: the renderer (`code/renderer/`) is built as a swappable DLL that calls platform-provided `GLimp_*` entry points, and this header declares the macOS-specific implementation of that interface plus the internal state those functions manipulate. By consolidating all macOS GL/display/input declarations here, the codebase achieves platform isolation—the core engine and game VMs never include macOS-specific headers; only the platform layer files (`macosx_*.m`, `macosx_*.c`) include this one.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer subsystem** (`code/renderer/tr_init.c`, `tr_main.c`, `tr_backend.c`): Calls `GLimp_*` functions defined in `macosx_glimp.m`; reads/writes `glw_state` for context and swap-buffer management.
- **Client engine** (`code/client/cl_main.c`, `cl_input.c`, `cl_scrn.c`): Calls `Sys_InitInput`/`Sys_ShutdownInput` and `Sys_QueEvent` during init/shutdown and per-frame event loop.
- **All macOS platform files** (`macosx_input.m`, `macosx_sys.m`, `macosx_glimp.m`, `macosx_snddma.m`, etc.): Include this header to declare their shared state and cross-file function signatures.

### Outgoing (what this file depends on)
- **`qcommon.h`**: Pulls in core types (`qboolean`, `sysEventType_t`, `fileHandle_t`).
- **CoreGraphics / ApplicationServices** (`CGDirectDisplayID`, `CGRect`, `CGGammaValue`): Direct OS dependencies for display enumeration, gamma tables, and mouse tracking.
- **OpenGL/CGL** (`CGLContextObj`, OpenGL headers): Low-level display context object from Core OpenGL.
- **Cocoa/Foundation** (via `#import`): `NSOpenGLContext`, `NSWindow`, `NSEvent` (forward-declared or void-typed for C++ safety).
- **`macosx_timers.h`**: Conditional OmniTimer profiling integration for thread-level GL performance analysis.

## Design Patterns & Rationale

### 1. **Language Interoperability via Opaque Void Types**
   - The `#ifdef __cplusplus` guards and void-typedef pattern (`typedef void NSDictionary`) allow a single header to be included by:
     - Pure C files (e.g., `macosx_sys.m` calling functions)
     - C++ source (future third-party code linking against the engine)
     - Objective-C files (which import real Cocoa classes)
   - This is a pragmatic workaround circa 2005—modern solutions would use `#pragma once` and proper forward declarations in different headers. The cost is zero type safety for Objective-C class pointers in C code paths.

### 2. **Macro-Based Context Lifecycle Management**
   - `OSX_SetGLContext`, `OSX_GLContextSetCurrent`, `OSX_GLContextClearCurrent` macros encapsulate the dual representation required by OpenGL on macOS: both `NSOpenGLContext` (Cocoa wrapper) and `CGLContextObj` (low-level CGL object) must be kept in sync.
   - This pattern avoids scattered context-switch code across multiple `.m` files and centralizes invariants: whenever context changes, both the NS and CGL representations are updated atomically.
   - The `_ctx_is_current` boolean is a cache of the actual GL state to avoid expensive Cocoa messaging on every check.

### 3. **Singleton State Container**
   - `glwstate_t glw_state` holds all GL/display/window state as a single global. This design reflects mid-2000s game engine patterns (no ECS, single-instance renderer), but it creates **hard constraints**:
     - Only one rendering context per process
     - No multi-window or context-sharing capabilities
     - Thread-unsafe without external synchronization
   - The `glPauseCount` incremented by `Sys_PauseGL`/`Sys_ResumeGL` is a **ref-counted pause mechanism**—allowing nested pause/resume calls without state confusion.

### 4. **Platform Abstraction Inversion**
   - The pattern is **not** "macOS implementations export functions"; it's "the renderer imports through a platform interface, and macOS defines that interface."
   - Renderer code never `#include "macosx_local.h"`; instead, renderer calls generic `GLimp_Init`, `GLimp_EndFrame`, etc. declared in `qcommon.h` or `tr_public.h`.
   - This header is private to the macOS platform layer—no cross-platform coupling.

## Data Flow Through This File

1. **Initialization**: `Sys_InitInput()` is called at engine startup. Input system reads `CGDirectDisplayID` from `Sys_DisplayToUse()` to determine mouse confinement area via `Sys_SetMouseInputRect()`.

2. **Renderer Context Setup**: `macosx_glimp.m` populates `glw_state` during `GLimp_Init`:
   - Display enumeration and mode (fullscreen vs windowed) stored in `desktopMode` / `gameMode` (NSDictionary)
   - Gamma ramp tables in `originalDisplayGammaTables` and working copies `inGameTable` / `tempTable`
   - NSOpenGLContext and CGL context pointers stored for future context switches
   - Window reference stored for resize/visibility changes

3. **Per-Frame Event Loop**: Cocoa event dispatcher calls into `Sys_QueEvent()` with keyboard/mouse/window events. These are enqueued into the engine's event ring buffer, later dispatched to cgame/UI VMs.

4. **GL Context Switching**: Before rendering, code calls `OSX_GLContextSetCurrent()` → renderer draws → swaps buffers. If the window minimizes, `Sys_Pause/ResumeGL` increments/decrements `glPauseCount`, signaling the renderer to skip frame rendering.

5. **Shutdown**: `Sys_ShutdownInput()` tears down input resources; renderer clears GL context via `OSX_GLContextClearCurrent()`.

## Learning Notes

### Idiomatic to Quake III's Era (2005)
- **Global singletons for subsystem state**: Modern engines use dependency injection or scene/world objects; Q3A uses globals like `glw_state`, `clientActive`, `svs`.
- **Macros for boilerplate**: The `OSX_*` macro pattern avoids function-call overhead for frequent operations. Modern compilers and inline functions have made this less critical.
- **C/Asm VM interfaces**: The syscall dispatch pattern (cgame/game/UI VMs call back into engine via trap_*) is unusual for 2025, where engines typically embed scripting or use plugin ABIs.
- **Cocoa/Carbon platform split**: This code targets pre-Intel Macs (PowerPC support implied by vm_ppc.c); modern Xcode dropped 32-bit and PPC support years ago.

### Cross-Cutting Architectural Lessons
1. **Platform abstraction works through inversion**: The renderer doesn't know about macOS; macOS implements generic `GLimp_*` signatures. This enabled id to ship renderer.dll on Windows/Linux/Mac from 1999–2005 with minimal forking.

2. **Display state complexity**: Gamma ramps, mode switching, multi-monitor support, and window lifecycle are non-trivial on real OSes. Even a "simple" game engine needs 100+ lines of per-platform boilerplate just to manage this state.

3. **C/Objective-C boundary management**: The void-typedef pattern is brittle; a production codebase would define a thin C wrapper API in `.m` files and keep this header pure C. The dual NSOpenGLContext/CGLContextObj pattern is essential because Cocoa doesn't expose all CGL features.

4. **Threading hints**: `glPauseCount` and the `_ctx_is_current` cache suggest that `Sys_PauseGL` can be called from one thread (e.g., Cocoa event dispatcher on window minimize) while the renderer runs on another. This is why a simple boolean wouldn't work—a race condition could cause the renderer to skip a frame incorrectly.

## Potential Issues

1. **Type-Safety Loss in C/C++ Code**: 
   - Declaring `NSDictionary` and `NSOpenGLContext` as `void` in C++ mode strips all compile-time type checking. If a C file accidentally passes a wrong pointer, the compiler silently accepts it.
   - **Mitigated slightly** by the fact that this is a private header; bugs here don't propagate to game code.

2. **Non-Thread-Safe Global State**:
   - `glw_state` is accessed without locks. If `Sys_PauseGL` is called from the Cocoa event thread while the renderer thread reads `glPauseCount`, a race condition could allow the renderer to start/stop mid-frame.
   - **In practice**: Q3A does not use truly independent threads on macOS (SMP is Windows-only per architecture docs), so this risk is mitigated—but the counter pattern suggests thread-awareness that isn't fully formalized.

3. **Forward Declaration Brittleness**:
   - `@class NSEvent, NSOpenGLContext, NSWindow;` is a forward declaration that works only in Objective-C files. In C files including this header, these become opaque void pointers with zero type info. A bug in a C file casting the wrong type through `void*` is undetectable.

4. **Gamma Table Lifecycle**:
   - `originalDisplayGammaTables` is a pointer to dynamically allocated `glwgamma_t` array, but the header doesn't declare allocation/deallocation functions. It's implicit that `macosx_glimp.m` manages this, but callers have no way to verify the pointer is valid.
