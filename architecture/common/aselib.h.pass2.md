# common/aselib.h — Enhanced Analysis

## Architectural Role

This header bridges the **offline tool pipeline** (q3map, bspc) to 3ds Max-exported mesh data. ASE is a text-based scene format, allowing level designers to export complex static/skeletal geometry from Max directly into Quake III maps via `misc_model` entities and bot navigation patches. It sits in `common/`, emphasizing that the entire ASE system is **build-time only**—zero runtime engine overhead. The loader unifies geometry import across both the BSP compiler (for lightmap-aware placement) and AAS compiler (for collision-aware bot nav).

## Key Cross-References

### Incoming (who depends on this)
- **q3map/** — Calls `ASE_Load()` during `misc_model` entity processing to import vertex/face data for lightmap baking and static collision
- **code/bspc/** — Calls `ASE_Load()` to consume skeletal mesh geometry during AAS reachability computation (e.g., for moving platforms modeled as mesh instances)
- Build tools that embed or link `common/aselib.c` directly (no DLL boundary; tight coupling by design)

### Outgoing (what this imports)
- **common/cmdlib.h** — `qboolean`, `qprintf`, error/warning macros; file I/O helpers (`fopen`, `fread`, etc.)
- **common/mathlib.h** — `vec3_t`, `vec_t`, vector math for geometry transformation; likely used in animation frame lerp or model-space→world-space conversion
- **common/polyset.h** — `polyset_t` struct: the **primary output type**, wrapping a named triangle set with material info; the ASE parser directly populates `polyset_t` arrays

## Design Patterns & Rationale

**Opaque State Holder + Query API:**  
The header exposes no structs or globals—all state resides in static file-scope variables in `aselib.c`. This is a deliberate **information-hiding pattern**: callers cannot accidentally read half-parsed state or corrupt internal pointers. The downside: **only one ASE file can be loaded at a time** (no per-load context struct). This is acceptable because build tools are single-threaded, sequential, and often operate on one model per invocation.

**Stateful Lifecycle:**  
The pattern `ASE_Load()` → `ASE_GetNumSurfaces()` → loop `ASE_Get*()` → `ASE_Free()` mirrors a **resource handle** without explicit handles. This is simpler than passing a context pointer everywhere, but less flexible than modern engines (e.g., Unreal's `UAssetLoader` which returns a loaded object directly). It reflects 1990s-2000s tool design: serial resource loading with global scope.

**Out-Parameter for Frame Counts:**  
`ASE_GetSurfaceAnimation(..., int *numFrames, ...)` uses a pointer to return the **actual** frame count after filtering (`skipFrameStart`, `skipFrameEnd`, `maxFrames`). This decouples the caller from needing to pre-allocate correctly—a practical choice for build tools where memory is plentiful but correctness matters.

## Data Flow Through This File

1. **Input**: ASE text file (from 3ds Max) on disk
2. **Parsing** (inside `aselib.c`): `ASE_Load()` opens, tokenizes, builds in-memory surface/frame tables
   - `ASE_GetToken()` lexes whitespace and quoted strings
   - `ASE_ParseBracedBlock()` handles nested scope (Max's block syntax)
   - `ASE_Process()` dispatches to geometry/material/animation handlers
3. **Storage**: File-static arrays in `aselib.c` holding:
   - Surface count and name lookup table
   - Per-surface frame arrays (one `polyset_t` per animation frame)
4. **Output**: Callers retrieve pointers via `ASE_Get*()`, iterate, use geometry
5. **Cleanup**: `ASE_Free()` deallocates file-static storage; ready for next file

## Learning Notes

**Era-Specific Design:**
- **No C++ classes, no RAII:** The resource lifecycle is manual and easy to forget (leak if caller doesn't call `ASE_Free()`). Modern engines use smart pointers or owned resources. This reflects Q3A's C-only, multi-platform portability mindset.
- **Static globals vs. context struct:** Simpler calling convention, but precludes async loading or parallel tool chains. Educational: shows the tradeoff between simplicity and flexibility.
- **Text format parsing:** ASE is human-readable, debuggable, but slow. Modern tools use binary formats (FBX, GLTF, Unreal Skeletal Mesh) with binary parsers. ASE's textual nature made it ideal for Max→Q3 interop circa 1999.

**Idiomatic to Q3A Toolchain:**
- The `polyset_t` output format is shared with other geometry importers (`l3dslib.c` for 3DS, `trilib.c` for TRI). A unified output type simplifies downstream processing.
- Frame filtering (`skipFrameStart`, `maxFrameEnd`) is a pragmatic design choice: Max animations often include setup/cleanup frames or unwanted intro/outro; the tool lets artists trim without re-exporting.

## Potential Issues

1. **Single-File Limitation:** No explicit reentrance guard. Calling `ASE_Load("file1.ase")` then `ASE_Load("file2.ase")` without an intervening `ASE_Free()` will leak file1's allocations. Modern practice: add a `qboolean` flag or explicit assertion.

2. **Memory Management Assumptions:** The header declares function signatures but offers no detail on who owns returned pointers. The first-pass notes that `ASE_GetSurfaceAnimation()` "likely allocates" the frame array—if true, its lifetime is managed opaquely and freed only by `ASE_Free()`. A caller holding a pointer to a frame after `ASE_Free()` invokes undefined behavior. No defensive null-checks visible at this API boundary.

3. **No Error Propagation:** `ASE_Load()` returns `void`. Parse errors are likely logged (via `cmdlib.h` functions) but not reported to the caller as a success/failure code. If a malformed ASE file is loaded, subsequent `ASE_GetSurfaceAnimation()` calls may return garbage or crash silently.

---

**Determinism & Reproducibility:** Build tools depend on deterministic output; the stateless query API (no randomization, no I/O during queries) supports this well. However, the single-file constraint means tool pipelines must serialize ASE load/process/free cycles—no parallelism.
