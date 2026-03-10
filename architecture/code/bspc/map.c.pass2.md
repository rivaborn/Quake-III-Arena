# code/bspc/map.c — Enhanced Analysis

## Architectural Role

This file serves as the **BSP-to-MAP converter** in the standalone BSPC tool pipeline, which is decoupled from the runtime engine. It acts as a unified format handler, translating BSP files from multiple Quake-era engines (Q1/Q2/Q3/HL/SIN) into a normalized intermediate `.map` text representation. This is a **tool-time transformation layer**, not part of the runtime `qcommon/cm_*` collision subsystem—BSPC is a separate offline compiler that generates AAS files for bot navigation using reusable botlib infrastructure.

## Key Cross-References

### Incoming (who depends on this file)

- **BSPC main (`code/bspc/bspc.c`)**: Calls `LoadMapFromBSP`, `WriteMapFile`, and accesses global `mapbrushes[]`, `mapplanes[]` arrays
- **AAS compilation pipeline (`code/bspc/aas_map.c`, `aas_create.c`)**: Consumes `mapbrushes[]` to generate AAS areas, reusing the brush geometry built here
- **Q3Map (separate tool, `q3map/map.c`)**: Implements identical `AddBrushBevels` and winding logic independently (code duplication, not shared)

### Outgoing (what this file depends on)

- **Format-specific loaders** (`l_bsp_q3.h`, `l_bsp_q2.h`, `l_bsp_q1.h`, `l_bsp_hl.h`, `l_bsp_sin.h`): Pluggable strategy dispatched via `LoadMapFromBSP` based on magic bytes
- **Winding utilities** (`l_mem.h`): `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding` for brush side clipping
- **AAS subsystem** (`aas_store.h`, `aas_cfg.h`): Only for constants (`AAS_MAX_BBOXES`); botlib AAS compilation is downstream
- **No qcommon dependency**: Unlike runtime collision (`qcommon/cm_*.c`), this tool has no ties to `qcommon.h` or networking

## Design Patterns & Rationale

### Format Abstraction via Dispatch
`LoadMapFromBSP` reads a minimal `idheader_t` (ident + version) and dispatches to the appropriate `Q{1,2,3}_AllocMaxBSP / LoadMapFromBSP / FreeMaxBSP` triplet. This avoids a monolithic parser and allows future format additions without core recompilation. **Tradeoff**: Code duplication (each format needs its own loader); gain is modularity and independence (swap loaders without rebuilding).

### Tool-Scoped Global State
All map arrays (`mapbrushes`, `mapplanes`, `planehash`) are file-static globals, not exported to other modules. This is acceptable for a standalone tool that processes one map at a time, unlike the runtime engine's stricter encapsulation. **Rationale**: Offline tools historically prioritized simplicity over architectural purity; a single global namespace per tool is common in 1990s–2000s compilers.

### Hash-Based Plane Deduplication
`FindFloatPlane` uses a hash table (`planehash[1024]`) to avoid O(n²) plane searches as brushes are added. The hash key is `(int)fabs(dist) / 8 & (PLANE_HASHES-1)`, and neighbors (±1 bins) are searched for epsilon-match tolerance. **Rationale**: Planes are expensive to store and compute; dedup is essential for large maps with many similar brushes.

### Bevel Addition Strategy
`AddBrushBevels` pre-emptively adds axial planes (6 max) and edge-derived slanted planes to each brush so the AAS compiler can sweep-expand the brush against AABB bounds without gaps. This is a preprocessing step unique to offline tools; runtime collision geometry doesn't need bevels. **Why separate from initial load**: Bevels depend on computed windings, which depend on all plane assignments being complete.

## Data Flow Through This File

1. **Input:** Raw BSP file (binary, format-specific encoding) via `LoadMapFromBSP`
2. **Dispatch:** Magic bytes → format detector → call appropriate `Q{1,2,3}_LoadMapFromBSP` 
3. **Loading:** Format-specific loader populates `mapbrushes[]`, `brushsides[]`, `mapplanes[]`, and raw texture pointers
4. **Geometry construction:**
   - `MakeBrushWindings`: Clip each brush side winding against all planes → compute AABB
   - `AddBrushBevels`: Insert missing axial/edge planes to fill gaps
   - `MarkBrushBevels`: Tag degenerate sides as bevels (post-hoc cleanup)
5. **Output:** 
   - `.map` text file (via `WriteMapFile` → `WriteMapFileSafe` → `WriteMapBrush`)
   - Global arrays fed to AAS pipeline (downstream in `code/bspc/aas_map.c`)

## Learning Notes

- **Era-specific idiom**: Late 1990s offline compiler design preferred global state and monolithic dispatch over OOP/modular patterns. Modern engines (Unreal, Unity) would use a map format abstraction interface and a per-format loader class.
- **No runtime collision tie-in**: This file never appears in the runtime engine's call path. `qcommon/cm_load.c` reimplements map loading and collision for runtime; the two are independent. This was a pragmatic split (tools ≠ runtime).
- **Shared algorithm, separate implementation**: Both BSPC (`map.c:AddBrushBevels`) and q3map (`q3map/map.c:AddBrushBevels`) implement the same bevel-addition algorithm. No shared code library. This reflects the era's tolerance for duplication and each tool's independence.
- **Winding clip-and-clip-again**: `MakeBrushWindings` computes windings by clipping an infinite plane against all other brush planes—a classic BSP preprocessing pattern (Tinn Foley's 1991 work on PVS). Modern engines may use different collision representations.
- **Plane signbits**: The `PlaneSignBits` encoding (3-bit bitmask of normal sign) is a Q3A idiom for fast plane-box rejection tests. It appears nowhere in this file's logic but is stored for downstream use (AAS, runtime collision).

## Potential Issues

- **Winding memory lifecycle**: `MakeBrushWindings` allocates windings into `side_t::winding` but relies on `ResetMapLoading` to free them all at once. If a single brush's winding fails to allocate mid-load, partial leaks are possible (no per-side error recovery).
- **Plane dedup epsilon**: `DIST_EPSILON = 0.02` and `NORMAL_EPSILON = 0.0001` are hardcoded. Different BSP formats (especially Q1, which uses integer coordinates) may hit floating-point precision cliffs with these values.
- **No format validation post-load**: After dispatch to a format-specific loader, no checks verify that `nummapplanes` and `nummapbrushes` are sensible. A corrupted BSP could overflow the static arrays silently.
