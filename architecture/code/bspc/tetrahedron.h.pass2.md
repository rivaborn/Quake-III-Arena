# code/bspc/tetrahedron.h — Enhanced Analysis

## Architectural Role

This header exposes a BSPC-specific visualization/debug utility for converting compiled AAS (Area Awareness System) geometry into tetrahedral mesh representation. It exists only in the offline toolchain (`code/bspc/`), not in the runtime engine, and serves developer/designer workflows for inspecting the spatial navigation structure that bots will traverse at runtime.

## Key Cross-References

### Incoming (who depends on this file)
- **Caller:** Likely `code/bspc/bspc.c` (main tool driver) or a tool-invoked command handler, as part of the optional post-compilation visualization pipeline
- **Pattern:** Similar to `AAS_DumpAASData` (in `code/bspc/aas_file.c`), which is a companion debug export function for AAS internals

### Outgoing (what this file depends on)
- **AAS infrastructure:** Reads from AAS file structures via `AAS_LoadAASFile`, `AAS_*` query functions (from `code/botlib/be_aas_*.c`)
- **File I/O:** Writes output geometry to disk; no includes visible in header, but implementation must call filesystem primitives (likely wrappers from `code/bspc/l_*.c` utility layer)
- **No direct link to botlib:** Consumes AAS data structures but does not link botlib; uses BSPC's own AAS recompilation pipeline

## Design Patterns & Rationale

- **Module namespace convention (`TH_`):** Mirrors BSPC's established prefix scheme (e.g., `AAS_*`, `MAP_*`, `CM_*`), making toolchain modules namespace-safe
- **Conversion utility pattern:** Transforms from one domain (AAS volumetric geometry) to another (tetrahedral mesh), enabling visualization without modifying the core AAS data
- **Offline-only scope:** Deliberately excluded from runtime engine to avoid bloat; only runs during map compilation when developers explicitly request debug output
- **Void return + side effects:** Follows BSPC's patterns of silent success with file I/O; errors likely logged to console rather than returned

## Data Flow Through This File

1. **Input:** AAS filename (map identifier; e.g., `"maps/q3dm1.aas"`)
2. **Processing chain:**
   - Load compiled AAS file (areas, faces, vertices, reachability graph)
   - Iterate over areas/faces
   - Decompose convex areas into tetrahedra (volumetric primitive)
3. **Output:** Debug mesh file (format inferable only from implementation), likely `.th` or similar debug extension for use in level editor or external visualizer

## Learning Notes

- **BSPC design principle:** Build-time tools are separate binaries with own I/O layer; no runtime dependency injection
- **AAS debugging workflow:** Developers can inspect whether bot-nav geometry is correctly split, merged, and reachable by visualizing tetrahedra
- **Offline vs. runtime:** Contrasts with runtime `code/botlib/be_aas_debug.c` (which draws debug lines in-game); this header is compile-time offline analysis
- **Minimal public interface:** Single function with maximal side effects (read file, compute, write file)—typical for Unix-style build tools from the 1990s–2000s era

## Potential Issues

- **Raw pointer, no validation:** Function signature accepts `char *filename` with no null-check or path validation; caller is trusted not to pass invalid path
- **Silent failure modes:** `void` return type means errors (file not found, disk full, corrupt AAS) cannot be signaled to caller; implementation must log to stderr/console or ignore errors
- **No documented output format:** Consuming code would need to know tetrahedral format by convention or inspect `.h` of the implementation file
