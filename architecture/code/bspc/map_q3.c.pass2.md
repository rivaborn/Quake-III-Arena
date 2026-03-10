# code/bspc/map_q3.c — Enhanced Analysis

## Architectural Role

This file is the **BSP→map adapter layer** that bridges the compiled Quake III BSP file format (native `q3_dbrush_t` structures in lumps) with BSPC's internal `mapbrush_t` representation. It serves as a convergence point for two distinct offline compilation pipelines: standard BSP processing (via `qbsp.c`) and AAS navigation geometry generation (via `aas_create.c`). The file essentially converts read-only BSP geometry into a mutable form suitable for further analysis or synthesis.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/map.c`** — `LoadMapFromBSP` calls `Q3_LoadMapFromBSP` as the BSP-format entry point (symmetrical to `Q1_LoadMapFromBSP`, `Q2_LoadMapFromBSP` for other BSP versions)
- **`code/bspc/bspc.c`** — Main tool driver; `create_aas` flag controls whether this module feeds the AAS or BSP pipeline
- Indirectly called by **q3map** and **bspc** tools; no runtime engine dependence

### Outgoing (what this file depends on)
- **`code/qcommon/cm_patch.h`** — `CM_GeneratePatchCollide` tessellates curved surfaces into collision facets
- **`code/bspc/aas_map.c`** — `AAS_CreateMapBrushes` processes each brush for AAS geometry when `create_aas==true`; takes ownership of brush data
- **`code/qcommon/cm_public.h`** — `FindFloatPlane` registers planes into the shared map-plane hash table
- **`code/bspc/map.c`** — `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels`, `AddBrushBevels` (geometric processing primitives)
- **`code/bspc/l_bsp_q3.h`** — Source of all raw BSP lump data; `Q3_LoadBSPFile`, `Q3_ParseEntities` populate the globals this file reads

## Design Patterns & Rationale

**Adapter / Converter Pattern**: Each function hierarchy (`Q3_BSPBrushToMapBrush` → `Q3_ParseBSPBrushes` → `Q3_ParseBSPEntity`) adapts one level of the BSP hierarchy into the map hierarchy. The nesting is intentional: entities own brushes own sides own planes.

**Router/Strategy Pattern**: The `create_aas` global flag acts as a compile-time switch. When true, brushes bypass `MakeBrushWindings` and go directly to `AAS_CreateMapBrushes` (which does its own tessellation). When false, brushes follow the normal BSP path (bevels, splitting, etc.). This enables **code reuse**: the same Q3 BSP loading logic feeds both tools without duplication.

**Validation-on-Load**: Functions like `Q3_BSPBrushToMapBrush` detect and log corruptions (duplicate planes, mixed content types, hint brushes with contents) rather than silently dropping data. This defensive posture is typical of offline tools processing potentially malformed user-generated content.

**Content-Type Priority Resolver**: `Q3_BrushContents` implements a cascading priority order (DONOTENTER > liquid > PLAYERCLIP > SOLID) to resolve ambiguity when a brush's sides have mixed content flags. This is **not** a simple OR; it respects game semantics.

## Data Flow Through This File

```
Raw BSP globals (q3_dbrushes, q3_drawSurfaces, etc.)
    ↓
Q3_LoadMapFromBSP (top-level orchestrator)
    ├─ Q3_LoadBSPFile (populates lumps, handled by l_bsp_q3.c)
    ├─ Q3_ParseEntities (populates entities[])
    ├─ Q3_ParseBSPEntity (per-entity loop)
    │   └─ Q3_ParseBSPBrushes (per-brush loop)
    │       └─ Q3_BSPBrushToMapBrush (side-by-side conversion)
    │           ├─ FindFloatPlane (plane registration)
    │           ├─ AAS_CreateMapBrushes (if create_aas) → geometry synthesis
    │           └─ MakeBrushWindings (if !create_aas) → BSP processing
    │
    └─ AAS_CreateCurveBrushes (patch tessellation pass)
        ├─ CM_GeneratePatchCollide (per-patch facet generation)
        └─ Route to AAS or BSP depending on create_aas
    ↓
mapbrushes[] array (mutable, ready for BSP tree building or AAS compilation)
map_mins/map_maxs (world bounds)
```

**Transformation notes:**
- **Planes**: Every BSP dplane is deduplicated via `FindFloatPlane` (may extend the global mapplanes array).
- **Shaders**: Content/surface flags are looked up from `q3_dshaders[]` lump; used to determine brush type (solid, water, clip, etc.).
- **Patches**: Non-solid patches are skipped; solid patches are tessellated into synthetic axis-aligned convex brush proxies to enable consistent collision/AAS geometry.

## Learning Notes

**For a developer studying this engine:**
1. **BSP as optimization**: The Q3 BSP already includes bevels, planes are validated, duplicate geometry is rare. This file **does not** recompute bevels (commented-out code). This differs from older ID engines where the BSP was less carefully constructed.
2. **Patch collision is non-trivial**: `AAS_CreateCurveBrushes` shows that Bézier patches are *not* natively walkable; they must be discretized into facet-proxy brushes first. This is a key insight into how Q3 achieves smooth curved surfaces while keeping navigation graphs efficient.
3. **Content-flag semantics are game-specific**: The priority logic in `Q3_BrushContents` is *not* general; it encodes Q3A game rules (ladder brushes are semantically different from solids). A modern engine would likely use a bit-vector or enum instead of OR-reduction.
4. **Offline tools are more verbose**: This file logs extensively (`Log_Write`, `Log_Print`) because offline compilation is interactive; a runtime engine would not afford this overhead.
5. **Global state is acceptable here**: Unlike the runtime engine (which uses per-VM `dataMask` sandboxing), BSPC uses flat globals (`nummapbrushes`, `brushsides[]`, `nodestack[]`) because it's single-threaded and batch-oriented.

**Idiomatic to the Q3 era (late 1999):**
- Extensive use of pre-allocated flat arrays (`mapbrushes[MAX_MAPFILE_BRUSHES]`) instead of linked lists or dynamic vectors
- Plane dedupe via `FindFloatPlane` is a common pattern (also seen in q3map)
- The `qboolean` return type (always `true` in `Q3_ParseBSPEntity`) suggests copy-paste from a generic entity loader

## Potential Issues

1. **Unbounded plane growth**: `FindFloatPlane` can silently extend `mapplanes[]` without checking `MAX_MAPFILE_PLANES`. If a malformed BSP has degenerate geometry, the map compiler could crash. No defensive bound-check visible.
2. **Patch bounds not validated**: `AAS_CreateCurveBrushes` checks `c > MAX_PATCH_VERTS` but not whether the tesselated brushes exceed `MAX_MAPFILE_BRUSHES`. A very dense patch could overflow.
3. **Silent side loss**: Duplicate/mirrored planes are detected and logged, but the loop `if (k != b->numsides) continue;` silently drops the side without incrementing `nummapbrushsides`. If a brush loses too many sides, `MakeBrushWindings` may fail silently later.
4. **content/surf flag namespace**: Assumes `q3_dshaders[i].contentFlags` and `.surfaceFlags` are always valid; no bounds check on `shaderNum < 0` beyond a quick `if`. If the BSP is corrupted, `q3_dshaders[negative_index]` could read garbage.

These are minor; BSPC is mature offline code, but the architecture assumes the BSP is well-formed (as produced by q3map).
