# code/bspc/nodraw.c — Enhanced Analysis

## Architectural Role

This file provides **link-time polymorphic stubs** for the BSPC compiler's debug visualization layer, enabling headless (non-interactive) offline compilation of BSP→AAS geometry. BSPC is a *standalone preprocessing tool* (not part of the runtime engine); it produces `.aas` files consumed by the runtime botlib subsystem. By linking `nodraw.c` instead of the full `gldraw.c`, the BSPC build avoids OpenGL/platform dependencies while retaining the identical symbol interface—a pre-DLL-era build configuration pattern.

## Key Cross-References

### Incoming (who depends on this)
- **`code/bspc/bspc.c`** — main BSPC entry point; links either `nodraw.c` (headless) or `gldraw.c` (interactive)
- **BSPC BSP→AAS pipeline** — any code in `code/bspc/` that calls `Draw_ClearWindow()`, `GLS_BeginScene()`, `GLS_Winding()`, `GLS_EndScene()` expects these symbols at link time
- **Build system** — Makefile or build script selects which `.o` to link (nodraw or gldraw) based on build variant

### Outgoing (what this depends on)
- **`code/bspc/qbsp.h`** — provides `winding_t`, `vec3_t`, `qboolean` type definitions and extern declarations for `draw_mins`, `draw_maxs`, `drawflag`
- **No other symbol dependencies** — this file is deliberately isolated; no calls to other BSPC modules

## Design Patterns & Rationale

**Link-Time Polymorphism (Pre-DLL Era)**  
Two build targets with *identical* public interfaces but different implementations:
- `gldraw.c` (real): submits polygons to OpenGL for real-time visual debugging during interactive BSPC runs
- `nodraw.c` (stub): discards all graphics calls for batch/CI builds

Rationale: Avoids `#ifdef`/`#if 0` preprocessor cruft; keeps code paths identical; enables two separate executables (`bspc-interactive` vs `bspc-headless`) from one source tree without conditional logic.

**Null Object Pattern**  
All functions are valid no-ops: they accept the same parameters, return void, and have no side effects. Caller code never needs to check "is this stub or real?" because the interface is uniform.

## Data Flow Through This File

1. **Inbound:** Global variables `draw_mins`/`draw_maxs`/`drawflag` are written by BSPC compilation passes to record debug geometry bounds; these globals have *no effect* in the stub variant.
2. **Winding geometry:** `GLS_Winding(winding_t *w, int code)` would be called with `winding_t` polygons from `code/bspc/aas_*.c` reachability/area creation. The `code` parameter likely encodes edge type, traversability, or color hint. With this stub, the geometry is discarded immediately.
3. **No persistence:** Frame lifecycle (`GLS_BeginScene` → multiple `GLS_Winding` calls → `GLS_EndScene`) is invoked but has no observable output.

## Learning Notes

- **Quake III era (2005) modularity:** No vtable pointers, no runtime dispatcher; polymorphism achieved purely at link time. Modern engines use function pointers or abstract interfaces; Q3 used file-level build variants.
- **Winding_t polygon format:** The `winding_t` structure (defined in `code/bspc/l_poly.h`) is a variable-length array of vertices. The debug system would use this to visualize AAS areas, reachability edges, and cluster topology.
- **GLSERV_PORT (25001):** Defined but unused in the stub; likely a protocol constant in the real `gldraw.c` for a socket-based GL server (remote visualization over network). Indicates BSPC originally supported *remote* graphics rendering—a sophisticated debug harness for headless machines.
- **Game engine contrast:** Unlike the runtime engine's renderer (which abstracts GL via `qgl_*` function pointers), BSPC solves the graphics problem entirely by substitution, not indirection.

## Potential Issues

- **Uninitialized globals:** `draw_mins`, `draw_maxs`, `drawflag` are defined but never initialized in this file. Caller (`qbsp.h` extern users) must zero them. If BSPC code relies on these being zeroed on startup, an undetected logic bug could hide in interactive builds (where `gldraw.c` might zero them) but not headless builds.
- **No input validation:** These stubs discard parameters without checking. If BSPC passes a corrupted `winding_t*` or invalid `code` value, no error is raised. The bug would only manifest in interactive mode (as a crash in `gldraw.c`) or not at all.
- **Silent discarding:** Developers adding new debug visualization to BSPC might accidentally call graphics functions expecting them to no-op, unaware they're being silently discarded in headless builds (leading to missing debug output in batch compiles).
