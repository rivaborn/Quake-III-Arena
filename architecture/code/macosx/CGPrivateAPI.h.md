# code/macosx/CGPrivateAPI.h

## File Purpose
Declares types, structures, and constants that mirror Apple's private CoreGraphics Server (CGS) API on macOS. This header enables Quake III's macOS port to hook into undocumented system-level event notification machinery, specifically to receive global mouse movement events outside of normal window focus.

## Core Responsibilities
- Define scalar primitive typedefs mirroring CGS internal integer/float types
- Declare the `CGSEventRecordData` union covering all macOS low-level event variants
- Declare the `CGSEventRecord` struct representing a complete raw system event
- Declare function pointer types for the private `CGSRegisterNotifyProc` notification registration API
- Define notification type constants for mouse-moved and mouse-dragged events

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CGSEventRecordData` | union | Variant payload for all event types: mouse, move, key, tracking, process, scroll wheel, tablet, proximity, compound |
| `_CGSEventRecord` / `CGSEventRecord` | struct | Complete system event record including location, timestamp, flags, window/connection IDs, and the data payload |
| `CGSNotifyProcPtr` | typedef (function pointer) | Callback signature for receiving CGS notifications |
| `CGSRegisterNotifyProcType` | typedef (function pointer) | Signature for the private `CGSRegisterNotifyProc` API, loaded at runtime to avoid link errors |

## Global / File-Static State
None.

## Key Functions / Methods
No function definitions — header only.

- **`CGSNotifyProcPtr`**: Callback invoked by the CGS notification system; receives type, opaque data pointer, data length, and a user arg.
- **`CGSRegisterNotifyProcType`**: Function pointer type for the private registration call; loaded dynamically (via `dlsym` or equivalent) so the binary does not hard-link the symbol.

## Control Flow Notes
This header is consumed by macOS-specific input or display code (likely `macosx_input.m` or `CGMouseDeltaFix.m`). At init time, the engine dynamically looks up `CGSRegisterNotifyProc` and registers a `CGSNotifyProcPtr` callback for the four mouse-movement notification constants. During the frame/event loop, macOS dispatches those callbacks outside the normal `NSEvent` path, giving the engine raw mouse delta data even when the cursor is captured or the window lacks focus.

## External Dependencies
- `<CoreGraphics/CoreGraphics.h>` — implied; uses `CGPoint` without definition in this file
- `CGSRegisterNotifyProc` — **defined in a private Apple framework** (CoreGraphics private); not linked directly, expected to be resolved at runtime
- No standard C library headers included directly
