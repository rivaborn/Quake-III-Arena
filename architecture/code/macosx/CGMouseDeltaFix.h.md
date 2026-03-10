# code/macosx/CGMouseDeltaFix.h

## File Purpose
This header declares a small macOS-specific shim that wraps CoreGraphics mouse delta querying. It provides a stable interface for retrieving raw mouse movement deltas, likely working around a platform bug or behavioral inconsistency in the `CGGetLastMouseDelta` API on early macOS versions.

## Core Responsibilities
- Declare initialization routine for the mouse delta fix subsystem
- Declare the mouse delta query function used by the macOS input layer
- Import the `ApplicationServices` framework to expose `CGMouseDelta` and related CG types

## Key Types / Data Structures
None defined in this file; relies on `CGMouseDelta` (a `int32_t` typedef) from `<ApplicationServices/ApplicationServices.h>`.

## Global / File-Static State
None.

## Key Functions / Methods

### CGFix_Initialize
- **Signature:** `void CGFix_Initialize()`
- **Purpose:** Sets up any state required by the mouse delta fix (e.g., installing an event tap or caching an initial mouse position).
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Not inferable from this file; implementation is in `CGMouseDeltaFix.m`.
- **Calls:** Not inferable from this file.
- **Notes:** Must be called before `CGFix_GetLastMouseDelta`; likely called during macOS input subsystem initialization.

### CGFix_GetLastMouseDelta
- **Signature:** `void CGFix_GetLastMouseDelta(CGMouseDelta *dx, CGMouseDelta *dy)`
- **Purpose:** Returns the accumulated raw mouse movement delta since the last call, as a drop-in replacement for or wrapper around `CGGetLastMouseDelta`.
- **Inputs:** `dx` — out-pointer for horizontal delta; `dy` — out-pointer for vertical delta.
- **Outputs/Return:** Values written through `dx` and `dy` pointers.
- **Side effects:** Likely resets the internal delta accumulator on each call (consistent with "last delta" semantics).
- **Calls:** Not inferable from this file.
- **Notes:** The "fix" naming strongly implies this addresses a known macOS bug (possibly delta accumulation being dropped or misreported under certain conditions).

## Control Flow Notes
Called from the macOS input polling path (`macosx_input.m`). `CGFix_Initialize` fits into the platform input **init** phase; `CGFix_GetLastMouseDelta` is called each **frame/input poll** to feed raw mouse movement into Quake's look/aim system.

## External Dependencies
- `<ApplicationServices/ApplicationServices.h>` — provides `CGMouseDelta`, CoreGraphics types; macOS-only framework
- Implementation body: `code/macosx/CGMouseDeltaFix.m` (Objective-C)
- Consumers: `code/macosx/macosx_input.m` (defined elsewhere)
