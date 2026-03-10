# code/bspc/gldraw.c — Enhanced Analysis

## Architectural Role

This file is a **standalone offline debugging utility** within the BSPC (BSP Compiler) toolchain, entirely separate from the runtime engine. It provides two independent visualization pathways for CSG and collision geometry during offline map compilation:
- **Local OpenGL rendering** via Windows `glaux` for immediate visual feedback during tool development
- **TCP socket-based remote GL server protocol** for headless or networked debugging scenarios

Neither pathway affects the runtime engine; this is purely a compile-time development aid.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/csg.c`, `code/bspc/brushbsp.c`, `code/bspc/aas_*.c`** — call `DrawWinding()`, `DrawAuxWinding()`, and color-setter functions to visualize intermediate geometry during BSP splitting and AAS subdivision operations
- **Global `drawflag`** — read by all draw functions; set externally (likely via command-line or config in `bspc.c`)
- **Global `draw_mins`, `draw_maxs`** — populated by caller modules to define the world AABB for camera framing

### Outgoing (what this file depends on)
- **OpenGL 1.x API** (`<GL/gl.h>`, `<GL/glu.h>`, `<GL/glaux.h>`) — Windows-only immediate-mode rendering
- **Winsock2** (`WSAStartup`, `socket`, `connect`, `send`, `closesocket`, `htonl`) — TCP/IP transport
- **`Error()`** — defined elsewhere in BSPC (likely `code/bspc/l_cmd.c` or common utilities); called on Winsock initialization failure
- **`winding_t` type** — pulled from `code/bspc/l_poly.h` via `qbsp.h`

## Design Patterns & Rationale

**Guard-on-disable pattern**: Every public function begins with `if (!drawflag) return;`. This allows the tool to completely elide visualization overhead when disabled, avoiding frame-rate impact during batch compilation.

**Dual-transport independence**: The local GL path and remote TCP path are entirely decoupled. A caller chooses which to invoke:
- Local: `Draw_ClearWindow()`, `DrawWinding()` — synchronous, immediate visual output
- Remote: `GLS_BeginScene()`, `GLS_Winding()`, `GLS_EndScene()` — asynchronous network transport

This separation reflects two different use cases: interactive debugging (local) vs. automation/CI pipelines (remote).

**Lazy initialization**: The local GL window is created on first `Draw_ClearWindow()` call (guarded by static `init`), deferring OS window creation until actually needed.

## Data Flow Through This File

**Local path:**
1. CSG/AAS modules populate `draw_mins`, `draw_maxs`, set `drawflag = true`
2. `Draw_ClearWindow()` lazily opens a 512×512 GL window, sets up projection/view to frame the AABB
3. Geometry enters via `DrawWinding(winding_t *)` or `DrawAuxWinding(winding_t *)`
4. Each call immediately issues GL vertex/color commands and flushes to the GPU
5. Output: rendered polygon overlay on screen

**Remote path:**
1. `GLS_BeginScene()` initializes Winsock (one-time), opens TCP connection to `127.0.0.1:25001`
2. Geometry enters via `GLS_Winding(winding_t *w, int code)`
3. Data serialized to 1024-byte stack buffer: `[numpoints (int), code (int), points (float×3×n)]`
4. Raw bytes sent over socket
5. `GLS_EndScene()` closes connection
6. Output: transmitted to remote GL server process listening on port 25001

## Learning Notes

This file exemplifies **mid-2000s offline tool design patterns**:
- **Immediate-mode GL** (pre-retained-mode era): all rendering via `glBegin/glVertex/glEnd`
- **Windows-centric**: hardcoded includes (`windows.h`, `glaux.h`) with no cross-platform consideration — this is a developer tool, not production code
- **Network protocol simplicity**: raw binary serialization over TCP, no framing/checksum/error recovery — acceptable for a debug channel on `127.0.0.1`
- **No abstraction layer**: GL calls directly embedded, not wrapped through a renderer interface (unlike the runtime `renderer/` module's `qgl_*` layer)

In modern engines, this would be replaced by:
- Runtime imgui/debug overlay system
- Structured logging to remote debugger
- In-editor visualization within the map compiler UI

## Potential Issues

1. **Buffer overflow in `GLS_Winding`**: Fixed 1024-byte stack buffer has no bounds checking. A winding with ≥85 points (each point = 12 bytes) + 8-byte header exceeds the buffer. This silently corrupts the stack.

2. **Network byte-order bug**: `GLS_BeginScene()` assigns `address.sin_port = GLSERV_PORT` (25001) directly without `htons()`. On big-endian architectures, this produces the wrong port number. (Practically benign on x86/x64, latent on PPC.)

3. **No send error handling**: `send()` call in `GLS_Winding()` ignores return value. Partial sends or EPIPE errors are silently discarded.

4. **Unvalidated socket reuse**: `GLS_Winding()` issues `send()` on `draw_socket` without verifying it's still connected or valid. A stale socket from a failed prior connection would cause undefined behavior.

5. **Windows-only hardcoding**: `#include <windows.h>` at top; will not compile on Linux/Mac even if BSPC is ported.

---

**Why these issues persist**: This is **offline development tooling**, not user-facing code. Robustness against network failures or buffer overflows is lower priority than development velocity. The tool assumes a trusted, local GL server process and windings within reasonable size bounds.
