# code/bspc/aas_gsubdiv.h — Enhanced Analysis

## Architectural Role

This header bridges the raw AAS area creation phase with geometry optimization and finalization. It exposes the **gravity-aware and ladder-specific spatial decomposition interfaces** that prepare areas for bot pathfinding. These functions execute as offline compilation passes within BSPC's AAS build pipeline—they are never invoked at runtime. They sit logically after initial area generation (`aas_create`) and before edge/face merging (`aas_edgemelting`, `aas_facemerging`) and area consolidation (`aas_areamerging`), ensuring navigation areas respect both movement physics boundaries (gravity) and traversal type isolation (ladders).

## Key Cross-References

### Incoming (who depends on this file)
- **Called from:** `code/bspc/` main AAS compilation orchestrator (likely `aas_create.c` or a top-level BSP→AAS converter such as `be_aas_bspc.c`)
- **Uses these functions as:** Subdivision post-processing steps applied to `tmpaasworld` after initial area extraction and before storage/optimization

### Outgoing (what this file depends on)
- **Calls (inferred from cross-ref):** 
  - Lower-level geometry operations: `AAS_SplitArea`, `AAS_SplitFace`, `AAS_SplitWinding` (defined in the paired `.c` file)
  - Plane/surface testing: `AAS_FindBestAreaSplitPlane`, `AAS_TestSplitPlane`
  - Area state updates: `AAS_RefreshLadderSubdividedTree_r`
- **Reads/writes:** Global `tmpaasworld` (the mutable AAS world under construction)

## Design Patterns & Rationale

**Recursive Subdivision (`*_r` naming):** Both functions follow a divide-and-conquer recursive pattern, splitting areas recursively along axis-aligned or heuristic planes. This is efficient for offline compilation and produces well-balanced spatial hierarchies for subsequent routing passes.

**Physics-Aware Decomposition:** Rather than generic geometric subdivision (e.g., BSP balancing), the two passes are semantically specific:
- **Gravitational subdivision** isolates regions where gravity behavior differs—separating walkable floor from void/pit regions so bots don't pathfind across infinite-fall boundaries.
- **Ladder subdivision** isolates vertical-only traversal zones, ensuring reachability analysis can distinguish ladder-only movement from general locomotion.

**Why this structure?** Early separation of these concerns prevents later routing passes from generating nonsensical "reachable" links (e.g., a jump that connects a ladder area to a disconnected island). The cost of subdivision at compile-time is negligible; the benefit to runtime pathfinding correctness is substantial.

## Data Flow Through This File

1. **Input:** Global `tmpaasworld` populated with raw areas (from `AAS_Create*`)
2. **Processing:** 
   - `AAS_GravitationalSubdivision()` recursively examines each area; if gravity properties change across a plane, splits the area and recurses on children.
   - `AAS_LadderSubdivision()` similarly identifies ladder-only surfaces and isolates them into dedicated areas.
3. **Output:** Modified `tmpaasworld` with finer-grained spatial partitioning, ready for clustering and reachability computation.

## Learning Notes

**Idiomatic to Q3's Era:** The explicit two-pass subdivision for gravity and ladders reflects 2000-era level design: Quake III maps feature prominent floating platforms, pits, and rope/ladder climbs. Modern engines often use:
- Navmesh layers or discrete navigation volumes per traversal type
- Physics capsule prediction at pathfinding query time (lazy validation) rather than compile-time decomposition

**Conceptual Lesson:** This file demonstrates **offline semantic enrichment**—using domain knowledge (gravity physics, ladder surface properties) to preprocess geometry, reducing the runtime pathfinding burden. The AAS system trades build time (one-shot) for lower memory footprint and faster runtime queries.

**Connection to Game Engine Concepts:** Similar in spirit to ECS preprocessing (component pre-computation) or baking in modern physics engines. Spatial decomposition for physics-aware pathfinding is a staple of autonomous agent systems.

## Potential Issues

- **Global state mutation without error recovery:** Both functions mutate `tmpaasworld` in place; if the compile process is interrupted or fails partway, there is no rollback. (Acceptable for BSPC since compilation is atomic—either full success or restart from scratch.)
- **Implicit ordering dependency:** These must execute *before* reachability computation and *after* raw area creation; no guard assertions in the header. Calling order is enforced only by the orchestrating code in the BSPC tool.
- **No size/iteration limits:** Recursive subdivision depth is not capped; pathological geometry could cause stack overflow, though in practice Q3 maps are well-bounded.
