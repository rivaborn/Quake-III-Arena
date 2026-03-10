# code/bspc/gldraw.c

## File Purpose
Provides debug visualization utilities for the BSPC (BSP Compiler) tool, offering two distinct rendering paths: a local OpenGL window via the Windows `glaux` library, and a TCP socket-based remote GL server protocol. It is Windows-only and intended for offline BSP compilation debugging, not runtime game rendering.

## Core Responsibilities
- Initialize and clear a local OpenGL debug window using the `glaux` auxiliary library
- Set current draw color for subsequent winding renders (red, grey, black)
- Draw `winding_t` polygons as filled + outlined primitives in the local GL window
- Establish a TCP connection to a local GL server (`GLS_BeginScene`)
- Serialize and transmit `winding_t` geometry over a socket to the remote GL server
- Close the remote GL server connection (`GLS_EndScene`)

## Key Types / Data Structures
None defined in this file; uses `winding_t` and `qboolean` from `qbsp.h` / `l_poly.h`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `drawflag` | `qboolean` | global | Master enable flag; all draw calls are no-ops when false |
| `draw_mins` | `vec3_t` | global | World-space AABB min, used to compute camera framing |
| `draw_maxs` | `vec3_t` | global | World-space AABB max, used to compute camera framing |
| `wins_init` | `qboolean` | file-static (global) | Guards one-time Winsock initialization |
| `draw_socket` | `int` | file-static (global) | Active TCP socket to the GL server; 0 = disconnected |

## Key Functions / Methods

### InitWindow
- **Signature:** `void InitWindow(void)`
- **Purpose:** Creates the `glaux` display window for local OpenGL debug rendering.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Opens a 512×512 RGB single-buffered OS window titled "qcsg".
- **Calls:** `auxInitDisplayMode`, `auxInitPosition`, `auxInitWindow`
- **Notes:** Called lazily from `Draw_ClearWindow` on first use.

### Draw_ClearWindow
- **Signature:** `void Draw_ClearWindow(void)`
- **Purpose:** Resets the local GL window each frame; sets up projection and view to frame the current BSP geometry.
- **Inputs:** None (reads `drawflag`, `draw_mins`, `draw_maxs`)
- **Outputs/Return:** None
- **Side effects:** Lazy-initializes the window; clears color buffer; sets GL projection/view matrices and render state.
- **Calls:** `InitWindow`, `glClearColor`, `glClear`, `glLoadIdentity`, `gluPerspective`, `gluLookAt`, `glColor3f`, `glPolygonMode`, `glDisable`, `glEnable`, `glBlendFunc`, `glFlush`
- **Notes:** Guard `if (!drawflag) return` is the first statement; an `#if 0` block contains an unused test polygon.

### DrawWinding
- **Signature:** `void DrawWinding(winding_t *w)`
- **Purpose:** Renders a winding polygon into the local GL window with a green semi-transparent fill and black outline.
- **Inputs:** `w` — pointer to winding to draw
- **Outputs/Return:** None
- **Side effects:** Issues immediate-mode GL draw calls; calls `glFlush`.
- **Calls:** `glColor4f`, `glBegin`, `glVertex3f`, `glEnd`, `glFlush`
- **Notes:** Uses `glVertex3f` explicitly (not `glVertex3fv`) because `vec3_t` fields may be `double` when `DOUBLEVEC_T` is defined.

### DrawAuxWinding
- **Signature:** `void DrawAuxWinding(winding_t *w)`
- **Purpose:** Identical to `DrawWinding` but fills red instead of green; used to distinguish a secondary/auxiliary winding.
- **Inputs:** `w` — pointer to winding to draw
- **Outputs/Return:** None
- **Side effects:** Same as `DrawWinding`.
- **Calls:** Same as `DrawWinding`.

### GLS_BeginScene
- **Signature:** `void GLS_BeginScene(void)`
- **Purpose:** Initializes Winsock (once) and opens a TCP connection to the local GL server on port 25001.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** One-time `WSAStartup`; creates and connects `draw_socket`; calls `Error` on Winsock init failure; silently sets `draw_socket = 0` on connect failure.
- **Calls:** `WSAStartup`, `socket`, `connect`, `closesocket`, `htonl`, `Error`
- **Notes:** Port is hardcoded as `GLSERV_PORT 25001`; `sin_port` is assigned without `htons()` — likely a latent byte-order bug.

### GLS_Winding
- **Signature:** `void GLS_Winding(winding_t *w, int code)`
- **Purpose:** Serializes a winding's point data and a code integer into a binary buffer and sends it to the GL server.
- **Inputs:** `w` — winding to transmit; `code` — caller-defined classification tag
- **Outputs/Return:** None
- **Side effects:** Writes to `draw_socket` via `send`; no-op if `draw_socket == 0`.
- **Calls:** `send`
- **Notes:** Buffer is stack-allocated at 1024 bytes; no bounds check — a winding with more than ~84 points (≥1024 bytes) would overflow.

### GLS_EndScene
- **Signature:** `void GLS_EndScene(void)`
- **Purpose:** Closes the TCP socket to the GL server and resets `draw_socket` to 0.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes `draw_socket`.
- **Calls:** `closesocket`

- **Notes (trivial helpers):** `Draw_SetRed`, `Draw_SetGrey`, `Draw_SetBlack` are one-liner color setters gated on `drawflag`.

## Control Flow Notes
This file is not part of any game frame loop. It is invoked during offline BSP compilation by other BSPC modules (e.g., `brushbsp.c`, `csg.c`) when debugging is enabled via `drawflag`. The two rendering paths (local GL window vs. TCP socket server) are independent; a caller chooses one by calling either `Draw_*` / `DrawWinding` functions or `GLS_BeginScene` / `GLS_Winding` / `GLS_EndScene`.

## External Dependencies
- `<windows.h>`, `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glaux.h>` — Windows-only; `glaux` is a legacy auxiliary library
- `qbsp.h` — pulls in `winding_t`, `vec3_t`, `vec_t`, `qboolean`, and `Error()`
- Winsock (`WSAStartup`, `socket`, `connect`, `send`, `closesocket`) — defined in Windows SDK, linked externally
- `Error()` — defined elsewhere in BSPC (`l_cmd.c`)
