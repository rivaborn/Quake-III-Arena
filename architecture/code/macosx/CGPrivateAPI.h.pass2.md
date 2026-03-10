# code/macosx/CGPrivateAPI.h — Enhanced Analysis

## Architectural Role

This header enables **raw system-level event interception on macOS** outside the normal AppKit event dispatch chain. The macOS platform layer uses it to register notifications for low-level mouse movement events, allowing Quake III to capture raw mouse delta data even when the window lacks focus or the cursor is captured—a critical capability for FPS games. It bridges the gap between the public NSEvent model (which respects window focus) and the engine's need for continuous input.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/macosx/macosx_input.m`** — Likely contains the platform input event loop; initializes `CGSRegisterNotifyProc` callbacks at startup and unpacks `CGSEventRecord` payloads to extract mouse deltas
- **`code/macosx/CGMouseDeltaFix.m`** — Companion file (visible in git status) probably wraps this API to isolate platform-specific workarounds
- Indirectly: **`code/client/client.h`** and **`code/client/cl_input.c`** consume the processed input via standard `usercmd_t` assembly

### Outgoing (what this file depends on)
- **Apple CoreGraphics private framework** — `CGSRegisterNotifyProc` function (resolved at runtime via `dlsym`, not linked)
- **`<CoreGraphics/CoreGraphics.h>`** — `CGPoint` type used in `_CGSEventRecord`; implicitly included
- **No engine dependencies** — header-only declarations; zero coupling to qcommon/renderer/game subsystems

## Design Patterns & Rationale

**1. Private API via Runtime Linking**
- `CGSRegisterNotifyProcType` is a function-pointer typedef, not a direct function call
- Engine must dynamically load the symbol to avoid build-time link errors if Apple removes the private function
- **Rationale:** Graceful degradation; binary runs on macOS versions that don't expose this API

**2. Callback-Driven Architecture**
- `CGSNotifyProcPtr` is the notification callback signature — asynchronous event delivery
- **Rationale:** CGS (CoreGraphics Server) is a low-level system daemon; callbacks are its native dispatch model

**3. Union-Based Event Payload (`CGSEventRecordData`)**
- All 9 event types (mouse, move, key, tracking, process, scroll, tablet, proximity, compound) share a union to save memory
- Each variant consumes only its required bytes; total size remains fixed (~40–44 bytes)
- **Rationale:** Matches the low-level CGS internal design; minimizes kernel-to-userspace IPC size

**4. Opaque Notification Data**
- `CGSNotificationData` is a `void*`; `CGSByteCount` specifies the length
- Callback receives raw bytes and must interpret based on `CGSNotificationType`
- **Rationale:** Loose coupling to kernel-provided event details; allows Apple to evolve payload without breaking ABI

## Data Flow Through This File

```
[macOS Kernel / CGS Daemon]
       ↓ (at startup)
Engine calls CGSRegisterNotifyProc() with:
  • callback function pointer (CGSNotifyProcPtr)
  • notification type (e.g., kCGSEventNotificationMouseMoved)
  • user arg (opaque context)
       ↓ (during gameplay)
[CGS Event Generation]
       ↓
Callback invoked: CGSNotifyProc(type, data, length, arg)
       ↓
Engine extracts CGSEventRecord from data → unpacks mouse deltas
       ↓
[code/client/cl_input.c] assembles usercmd_t, feeds frame loop
```

## Learning Notes

**Idiomatic to this era (2005 macOS ports):**
- No Cocoa-level event filtering; direct kernel/CGS hookup for precision
- Union-based event encoding reflects C-level systems programming (memory efficiency over type safety)
- Dynamic symbol resolution (`dlsym`-style) was the standard workaround for private API before App Store sandboxing

**How modern engines differ:**
- Modern macOS ports use **`IOKit` HID layer** (public) or **Metal event loops** (higher-level)
- Swift-era frameworks (Appkit in 2023+) provide better event capture APIs without private hooks
- Modern input systems often use **event queues** rather than per-frame callbacks

**Connection to engine concepts:**
- This is a **platform abstraction boundary** — the engine's input subsystem (`code/client/cl_input.c`) never directly knows about CGS; macOS input layer translates `CGSEventRecord` → engine's canonical input representation
- Exemplifies **temporal decoupling** — events arrive asynchronously via callback, buffered until frame processing

## Potential Issues

1. **Private API Fragility**: `CGSRegisterNotifyProc` is undocumented. Apple changed or removed private APIs frequently between OS X 10.4–10.7 (the era this code targets). No version guards are visible in this header.

2. **Incomplete Notification Data Handling**: `CGSNotificationData` (void*) lacks any built-in validation. Misinterpreting the payload layout by changing OS versions could cause misaligned memory reads.

3. **No Error Handling Path**: The typedef for `CGSRegisterNotifyProcType` returns `CGSError`, but this header defines the type only—the actual error handling must live in the caller (`macosx_input.m`). If dynamic symbol lookup fails silently, input may degrade without warning.

4. **Thread Safety Ambiguous**: No documentation on whether `CGSNotifyProcPtr` callbacks fire on main thread or a system thread, which is critical for safe memory access in the engine.
