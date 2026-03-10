# code/macosx/CGMouseDeltaFix.h
## File Purpose
This header declares a small macOS-specific shim that wraps CoreGraphics mouse delta querying. It provides a stable interface for retrieving raw mouse movement deltas, likely working around a platform bug or behavioral inconsistency in the `CGGetLastMouseDelta` API on early macOS versions.

## Core Responsibilities
- Declare initialization routine for the mouse delta fix subsystem
- Declare the mouse delta query function used by the macOS input layer
- Import the `ApplicationServices` framework to expose `CGMouseDelta` and related CG types

## External Dependencies
- `<ApplicationServices/ApplicationServices.h>` — provides `CGMouseDelta`, CoreGraphics types; macOS-only framework
- Implementation body: `code/macosx/CGMouseDeltaFix.m` (Objective-C)
- Consumers: `code/macosx/macosx_input.m` (defined elsewhere)

# code/macosx/CGPrivateAPI.h
## File Purpose
Declares types, structures, and constants that mirror Apple's private CoreGraphics Server (CGS) API on macOS. This header enables Quake III's macOS port to hook into undocumented system-level event notification machinery, specifically to receive global mouse movement events outside of normal window focus.

## Core Responsibilities
- Define scalar primitive typedefs mirroring CGS internal integer/float types
- Declare the `CGSEventRecordData` union covering all macOS low-level event variants
- Declare the `CGSEventRecord` struct representing a complete raw system event
- Declare function pointer types for the private `CGSRegisterNotifyProc` notification registration API
- Define notification type constants for mouse-moved and mouse-dragged events

## External Dependencies
- `<CoreGraphics/CoreGraphics.h>` — implied; uses `CGPoint` without definition in this file
- `CGSRegisterNotifyProc` — **defined in a private Apple framework** (CoreGraphics private); not linked directly, expected to be resolved at runtime
- No standard C library headers included directly

# code/macosx/Q3Controller.h
## File Purpose
Declares the `Q3Controller` Objective-C class, which serves as the macOS application controller (NSObject subclass) for Quake III Arena. It acts as the AppKit-facing entry point that bridges the macOS application lifecycle into the engine's main loop.

## Core Responsibilities
- Declares the main application controller class for the macOS platform
- Exposes an Interface Builder outlet for a splash/banner panel
- Provides IBActions for clipboard paste and application termination requests
- Declares `quakeMain` as the engine entry point invoked from the macOS app

## External Dependencies
- `<AppKit/AppKit.h>` — AppKit framework (NSObject, NSPanel, IBOutlet, IBAction)
- `DEDICATED` — preprocessor macro defined externally to strip client-only UI code
- `Q3Controller.m` — implementation file (defined elsewhere)
- `Quake3.nib` — Interface Builder nib file that instantiates this controller and wires `bannerPanel` (defined elsewhere)

# code/macosx/macosx_display.h
## File Purpose
Public interface header for macOS display management in Quake III Arena. It declares functions for querying display modes, managing hardware gamma ramp tables, and fading/unfading displays during mode switches.

## Core Responsibilities
- Declare the display mode query function (`Sys_GetMatchingDisplayMode`)
- Declare gamma table storage and retrieval functions
- Declare per-display and all-display fade/unfade operations
- Declare display release cleanup

## External Dependencies
- `tr_local.h` — renderer types (`qboolean`, `glconfig_t`, etc.)
- `macosx_local.h` — `glwgamma_t`, `glwstate_t`, `CGDirectDisplayID`, `glw_state` global
- `ApplicationServices/ApplicationServices.h` (via `macosx_local.h`) — `CGDirectDisplayID`, Core Graphics display API
- Implementations defined in `macosx_display.m` (not visible here)

# code/macosx/macosx_glimp.h
## File Purpose
A minimal platform-specific header that sets up the OpenGL framework includes for the macOS renderer backend. It conditionally enables a CGL macro optimization path that bypasses per-call context lookups.

## Core Responsibilities
- Include the macOS OpenGL framework headers (`OpenGL/gl.h`, `OpenGL/glu.h`, `OpenGL/OpenGL.h`)
- Conditionally include `glext.h` if `GL_EXT_abgr` is not already defined
- Optionally enable `CGLMacro.h` mode to eliminate redundant CGL context lookups per GL call
- Expose the `cgl_ctx` alias into translation units that include this header under `USE_CGLMACROS`

## External Dependencies
- `<OpenGL/OpenGL.h>` — CGL and core GL types (Apple framework)
- `<OpenGL/gl.h>` — Standard OpenGL API (Apple framework)
- `<OpenGL/glu.h>` — OpenGL Utility Library (Apple framework)
- `<OpenGL/glext.h>` — GL extensions (Apple framework), guarded by `GL_EXT_abgr`
- `macosx_local.h` — Pulled in only under `USE_CGLMACROS`; provides `glw_state` (`glwstate_t`) and the `_cgl_ctx` field (`CGLContextObj`)
- `<OpenGL/CGLMacro.h>` — Apple CGL macro rewrite header, only under `USE_CGLMACROS`; defined elsewhere (Apple SDK)

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

## External Dependencies
- `qcommon.h` — engine core types (`qboolean`, `sysEventType_t`, `fileHandle_t`, etc.)
- `<ApplicationServices/ApplicationServices.h>` — `CGRect`, `CGDirectDisplayID`, `CGGammaValue`, etc.
- `<OpenGL/CGLTypes.h>` — `CGLContextObj`
- `<Foundation/NSGeometry.h>` (Obj-C only) — `NSRect`
- `macosx_timers.h` — `OTStampList` / `OmniTimer` profiling API (conditional on `OMNI_TIMER`)
- `NSOpenGLContext`, `NSWindow`, `NSEvent` — Cocoa objects (forward-declared or void-typed for C++ compatibility)
- `glw_state`, `Sys_IsHidden`, `glThreadStampList` — defined in `macosx_glimp.m` / `macosx_sys.m`

# code/macosx/macosx_qgl.h
## File Purpose
Autogenerated macOS-specific header that wraps every standard OpenGL 1.x function in a `qgl`-prefixed inline shim. Each shim optionally logs the call to a debug file and/or checks for GL errors after the call, then forwards to the real `gl*` function. A block of `#define` macros at the end redirects all bare `gl*` names to the error-message symbols, forcing callers to use the `qgl*` versions.

## Core Responsibilities
- Provide `qgl*` inline wrappers for every OpenGL 1.x core function (~200+ functions)
- Conditionally log GL call parameters to a debug file when `QGL_LOG_GL_CALLS` is defined
- Conditionally call `QGLCheckError()` after each GL call when `QGL_CHECK_GL_ERRORS` is defined
- Track nested `glBegin`/`glEnd` depth via `QGLBeginStarted` to suppress error checks inside a primitive block
- Poison all bare `gl*` symbols via `#define gl* CALL_THE_QGL_VERSION_OF_gl*` to enforce use of wrappers
- Provide `_glGetError()` as an unguarded bypass to avoid infinite recursion inside `QGLCheckError`

## External Dependencies
- Standard OpenGL headers (implicit via including code): all `gl*` functions, GL types
- `QGLCheckError` — defined in a companion `.m`/`.c` file (`macosx_qgl.m` or similar)
- `QGLDebugFile()` — returns a `FILE*` for debug output; defined elsewhere
- `QGLLogGLCalls`, `QGLBeginStarted` — extern globals, defined in companion source

# code/macosx/macosx_timers.h
## File Purpose
Conditional header that exposes macOS-specific OmniTimer profiling instrumentation for Quake III Arena's renderer and collision subsystems. When `OMNI_TIMER` is not defined, all macros collapse to no-ops, making the profiling entirely compile-time optional.

## Core Responsibilities
- Define `OTSTART`/`OTSTOP` macros for push/pop-style hierarchical timer nodes
- Declare extern `OTStackNode*` globals representing named profiling points across the renderer and collision paths
- Declare the `InitializeTimers()` initialization function
- Provide a zero-cost stub path (empty macros) when `OMNI_TIMER` is undefined

## External Dependencies
- `<OmniTimer/OmniTimer.h>` — macOS/OmniGroup framework providing `OTStackNode`, `OTStackPush`, `OTStackPop`; not present in the open-source release
- All `OTStackNode*` definitions live in a corresponding `.m` implementation file (not in this header)

