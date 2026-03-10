# code/bspc/nodraw.c

## File Purpose
A null/stub implementation of the BSP compiler's OpenGL debug drawing interface. All functions are empty no-ops, serving as a build target for headless/server-side BSP compilation where no graphical debug visualization is needed.

## Core Responsibilities
- Provide link-time stubs for the GL scene visualization API declared in `qbsp.h` (gldraw.c section)
- Define the global drawing state variables `draw_mins`, `draw_maxs`, and `drawflag`
- Allow the BSPC tool to compile and link without a real GL debug renderer

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `draw_mins` | `vec3_t` | global | Minimum bounds of debug draw region (unused in this stub) |
| `draw_maxs` | `vec3_t` | global | Maximum bounds of debug draw region (unused in this stub) |
| `drawflag` | `qboolean` | global | Flag indicating whether debug drawing is active (unused in this stub) |

## Key Functions / Methods

### Draw_ClearWindow
- Signature: `void Draw_ClearWindow(void)`
- Purpose: Stub for clearing the debug GL window.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None
- Notes: No-op; real implementation would clear an OpenGL viewport.

### GLS_BeginScene
- Signature: `void GLS_BeginScene(void)`
- Purpose: Stub for beginning a GL scene frame.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None
- Notes: No-op; `GLSERV_PORT 25001` is defined but unused, suggesting a socket-based GL server was originally planned or exists in a sibling implementation.

### GLS_Winding
- Signature: `void GLS_Winding(winding_t *w, int code)`
- Purpose: Stub for submitting a winding polygon to the debug GL renderer.
- Inputs: `w` — winding to draw; `code` — rendering hint/color code
- Outputs/Return: None
- Side effects: None
- Calls: None
- Notes: No-op.

### GLS_EndScene
- Signature: `void GLS_EndScene(void)`
- Purpose: Stub for finalizing and flushing a GL scene frame.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None
- Notes: No-op.

## Control Flow Notes
This file plays no role in normal BSP compilation control flow. It is linked in place of a real `gldraw.c` implementation for non-interactive (headless) builds of the BSPC tool. The global variables `draw_mins`/`draw_maxs`/`drawflag` are declared here and declared `extern` in `qbsp.h`, so other BSP modules can set them without triggering a link error, but they have no effect.

## External Dependencies
- `qbsp.h` — pulls in all BSPC types (`winding_t`, `vec3_t`, `qboolean`, etc.) and the `extern` declarations for the symbols defined here
- `winding_t` — defined in `l_poly.h` (via `qbsp.h`); used only as a pointer parameter in stubs
- `GLSERV_PORT` (25001) — defined locally but never referenced; implies a GL server protocol exists elsewhere
