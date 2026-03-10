# code/bspc/aas_map.h — Enhanced Analysis

## Architectural Role

This header defines the boundary between **BSP geometry parsing** and **AAS area generation** within BSPC's offline compilation pipeline. It exposes the critical first step in converting a BSP map's raw brush geometry into the navigational topology that botlib will use at runtime. The function consumes map entities' brush definitions (from BSP headers) and translates them into AAS area-space geometry—a transformation required before clustering, reachability analysis, and path-finding can proceed.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/bspc/aas_create.c`** — Master AAS creation orchestrator; iterates all map brushes and calls `AAS_CreateMapBrushes` to seed the area generation pipeline
- **BSPC compiler pipeline** — Called during map-to-AAS phase within the offline tool workflow (likely from `code/bspc/be_aas_bspc.c` or `bspc.c`)

### Outgoing (what this file depends on)

- **`code/bspc/aas_map.c`** — Implementation file; defines helper functions (`AAS_ExpandMapBrush`, `AAS_PositionBrush`, `AAS_PositionFuncRotatingBrush`, `AAS_SetTexinfo`, `AAS_TransformPlane`, `AAS_ValidEntity`, `AAS_AlwaysTriggered`)
- **`mapbrush_t` / `entity_t`** — BSP geometry types from `code/bspc/qbsp.h` / `code/bspc/map.h`
- **Downstream AAS pipeline** — Results feed into `aas_create.c` (area subdivision), `aas_gsubdiv.c` (gravitational refinement), and `aas_file.c` (serialization)

## Design Patterns & Rationale

**Offline Compiler Pattern:**  
This header is part of BSPC's **offline-only** subsystem (unlike botlib's `be_aas_*.c` which runs at runtime). The choice to keep map brush conversion in a separate module reflects the principle that compilation-time geometry transformations are logically distinct from runtime navigation queries. The compiler "bakes in" geometric decisions (like bevel planes) once, avoiding repeated computation.

**BSP→AAS Translation Pattern:**  
The function bridges two geometric models:
- **BSP model** (`mapbrush_t`): Face-per-plane representation suitable for rendering and collision
- **AAS model** (internal structures): Area/portal/face/plane representation optimized for pathfinding

This translation is non-trivial—it must handle entity-specific brushes (movers, trigger volumes) differently from static geometry.

**Bevel Plane Heuristic:**  
The `addbevels` parameter encodes a **compiler trade-off**: bevel planes smooth bot collision along brush edges but increase AAS data size. This is a **compile-time optimization decision** not exposed at runtime, illustrating how offline tools can precompute navigational metadata.

## Data Flow Through This File

```
Input:  mapbrush_t (BSP brush geometry)
        entity_t (owning entity, e.g., mover flags, trigger type)
        addbevels (boolean: add edge-smoothing planes)
           ↓
        [AAS_ExpandMapBrush - expands brush boundaries]
        [AAS_PositionBrush - applies entity transformation]
        [AAS_TransformPlane - converts planes to AAS space]
        [AAS_SetTexinfo - assigns surface properties]
           ↓
Output: AAS brush data written to global aasworld singleton
        (faces, planes, edges recorded for later area generation)
```

Key state transition: **Raw BSP geometry → Entity-transformed AAS planes → Area subdivision input**

## Learning Notes

**What this reveals about the engine:**
1. **Two-phase AAS compilation:** BSP geometry is first converted to AAS geometry (this file's role), then areas and reachability are computed in subsequent phases. This separation allows each phase to focus on distinct concerns.

2. **Entity-specific geometry handling:** The `mapent` parameter hints at runtime variations—some brushes belong to movers (`func_plat`, `func_rotating`) or triggers, requiring coordinate-space transformations. A naive compiler might skip these; BSPC explicitly positions and transforms them.

3. **Offline vs. runtime split:** Unlike modern engines that might compute navigation online, Quake III's architecture aggressively precomputes everything into `.aas` binary files. This file exemplifies that philosophy—geometry baking happens once, loading happens per-session.

4. **Idiomatic to this era:** In contrast to modern engines using ECS or skeletal data-driven navigation, this represents the **explicit-geometry-model** approach: each brush is individually processed, transformed, and stored. Modern equivalents might use volumetric fields or voxel grids computed on-the-fly.

## Potential Issues

**Not directly inferable, but worth noting:**

- **Edge cases in entity transformation:** Rotating brushes (`AAS_PositionFuncRotatingBrush`) may not handle all rotation matrices correctly; complex entity hierarchies could be missed.
- **Bevel plane quality:** The `addbevels` flag is likely a boolean or simple count, potentially insufficient for high-curvature geometry (e.g., curved ramps).
- **No validation visible:** The header doesn't show error-checking for invalid brushes (e.g., self-intersecting, degenerate faces). This likely happens in `aas_map.c`'s helpers.
