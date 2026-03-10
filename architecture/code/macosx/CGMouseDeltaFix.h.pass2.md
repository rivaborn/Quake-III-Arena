# code/macosx/CGMouseDeltaFix.h — Enhanced Analysis

## Architectural Role

This file provides a macOS-specific shim for raw mouse delta querying, positioned within the **platform input layer** (`code/macosx/`). It abstracts away a known CoreGraphics quirk or API instability on early macOS versions, preventing drift/accumulation bugs in the mouse tracking subsystem. The module integrates directly into the **client** subsystem's per-frame input polling loop and feeds raw mouse movement data to the aim/look control system that eventually propagates into the cgame VM for client-side prediction and 3D camera updates.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/macosx/macosx_input.m`** — calls `CGFix_Initialize()` during input subsystem setup and `CGFix_GetLastMouseDelta()` each frame to acquire raw mouse movement for the `usercmd_t` assembly pipeline
- **Client input processing** (`code/client/cl_input.c`) — indirectly receives mouse delta data via the platform layer's `IN_Frame()`

### Outgoing (what this file depends on)
- **`<ApplicationServices/ApplicationServices.h>`** — provides `CGMouseDelta` typedef (likely `int32_t`), `CGGetLastMouseDelta()` API, and CoreGraphics mouse tracking primitives; macOS-only framework
- **Implementation** (`code/macosx/CGMouseDeltaFix.m`, Objective-C) — contains the actual bug workaround logic (caching, event taps, initial mouse position normalization, or API call wrapping)

## Design Patterns & Rationale

**Platform abstraction layer**: Rather than scattering macOS-specific CoreGraphics calls throughout the input code, Q3A wraps them in a thin, versioned API. This allows:
- **Isolation of platform quirks** — the workaround logic is confined to one `.m` file
- **Testability** — the public interface is stable even if the underlying fix evolves
- **Code reuse** — other macOS input code can depend on a consistent, guaranteed-correct delta stream

**Initialization + polling pattern**: Mirroring subsystems like the renderer (`GLimp_Init` / `GLimp_EndFrame`), this module follows a two-phase lifecycle:
1. `CGFix_Initialize()` — one-time setup (likely installing an event tap or caching an initial sample)
2. `CGFix_GetLastMouseDelta()` — per-frame query (accumulates and resets deltas)

This pattern ensures no data is lost and the subsystem is ready before polling begins.

## Data Flow Through This File

```
User mouse movement (hardware)
        ↓
  CoreGraphics event queue
        ↓
CGFix_GetLastMouseDelta() [out-params: dx, dy]
        ↓
  Client input frame: cl_input.c → usercmd_t assembly
        ↓
  Network transmission (delta-compressed usercmd_t)
        ↓
  Server: Pmove execution (trace, collision, step)
        ↓
  Client prediction (cgame): camera update, aim correction
```

The deltas are typically accumulated over one frame (16–33 ms at 60–30 fps), then reset on the next call to maintain "last delta" semantics.

## Learning Notes

**Platform quirk workarounds (2005 era)**: Early macOS versions (10.2–10.3) had inconsistent or unreliable `CGGetLastMouseDelta()` behavior—deltas could be dropped, accumulated incorrectly, or reset unexpectedly under certain window/focus conditions. Rather than retrying or polling the API multiple times, the fix likely uses an **event tap** (`CGEventTapCreate`) or **absolute position tracking** to maintain a delta accumulator independent of CoreGraphics' internal state. This is idiomatic for cross-platform game engines of this era.

**Contrast with modern engines**: Modern engines (Unreal, Unity, Godot) abstract input through higher-level frameworks (raw input devices, event queues) that hide OS quirks more completely. Q3A's approach—a thin shim per platform—is lean but requires platform-specific knowledge to maintain.

**Connection to CoreGraphics renderer**: The same framework (`ApplicationServices`) powers the renderer module (`macosx_glimp.m`), which handles window creation and gamma correction. Both platform modules can share framebuffer and input event management, reducing duplication.

## Potential Issues

*None clearly inferable*. The small, single-responsibility design minimizes risk. However:
- If `CGFix_Initialize()` is not called before `CGFix_GetLastMouseDelta()`, undefined behavior (likely crash or garbage delta values) will result.
- The underlying CoreGraphics quirk is not documented in this header; future maintainers may not understand why the fix exists and inadvertently revert to direct `CGGetLastMouseDelta()` calls.
