# common/polyset.h — Enhanced Analysis

## Architectural Role

`polyset.h` serves as the shared data model for offline geometry processing pipelines that convert artist-authored 3D assets into engine-consumable binary formats. It bridges high-level model import (ASE, 3DS, etc.; handled by sibling files `aselib.c`, `l3dslib.c`) and downstream compilers (`q3map`, `q3radiant`). Critically, it has **zero runtime engine presence** — the engine never loads or consumes polyset data directly; all geometry is pre-baked into BSP and model files during the asset build phase.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/** — BSP compiler (geometry processing pipeline)
- **q3radiant/** — Level editor (ASE/3DS import, editing, export)
- **code/bspc/** — Offline AAS (AI pathing) compiler, which reuses botlib's geometry processing
- **aselib.c** / **l3dslib.c** — Model importers; load raw vertex/face data that `Polyset_*` functions restructure into triangle sets

### Outgoing (what this file depends on)
- **mathlib.h** — Provides `vec3_t` (3D position vectors)
- **q_shared.h** — Base types (implicit via mathlib)
- No runtime engine dependencies; purely compile-time and offline tool scope

## Design Patterns & Rationale

**Stateless Pipeline Pattern**: Each `Polyset_*` function is a pure transform with no global state. This allows modular composition:
```
load(file) → [split/collapse as needed] → snap → computeNormals → export
```

**Allocation Ownership**: Functions return newly allocated `polyset_t` arrays; caller owns lifetime. This allows chaining transforms without explicit intermediate freeing, though the header leaves memory management semantics ambiguous (e.g., does `Polyset_CollapseSets` free input?).

**Material Grouping**: `polyset_t.materialname` allows per-set shader binding during compilation, enabling tools to group triangles by surface properties before BSP/lightmap generation.

**Type Simplification**: Uses floats for UV (`st_t`) and normals despite engine's typical fixed-point conventions, since offline tools prioritize precision over size.

## Data Flow Through This File

1. **Import Phase**: `ASE_Load` or 3DS parser → raw vertex/face/material arrays
2. **Restructure**: `Polyset_LoadSets` allocates and populates `triangle_t` structures, grouping by material name → array of `polyset_t`
3. **Optimization**: `Polyset_SplitSets` (if per-set triangle count exceeds limit for lightmapping) or `Polyset_CollapseSets` (merge identical materials)
4. **Normalization**: `Polyset_SnapSets` (reduce floating-point noise to grid), `Polyset_ComputeNormals` (fill per-vertex normals)
5. **Export**: Tools serialize normalized polysets to BSP or MD3/MD4 model formats

## Learning Notes

**Offline-first design**: This codebase separates asset authoring from runtime strictly — no dynamic model loading, no in-game geometry modification. Contrast with modern engines (Unreal, Unity) that ship material/mesh editors at runtime.

**Material-centric organization**: Early 2000s Q3A design groups geometry by material shader, not by draw order or memory layout. This reflects the separate lighting and texture baking pipelines.

**Grid snapping** as a canonical technique: Floating-point geometry is snapped to a grid to eliminate micro-gaps that cause Z-fighting and PVS cracks during BSP compilation — a pattern still used in some map editors today.

**No explicit metadata**: Unlike modern glTF/FBX, polysets carry only positions, normals, UVs, and material names. Animation rigs, bones, and blend shapes are handled elsewhere (`splines/` for camera paths, MD3 skeleton data encoded separately).

## Potential Issues

- **`rgb_t` unused**: Declared but never instantiated in any struct. Likely a holdover from an earlier color-per-vertex design that was removed.
- **Snap precision opaque**: Grid size hardcoded in `Polyset_SnapSets` implementation (not visible here). Tools/mappers cannot control tolerance.
- **Memory ownership unclear**: Header does not specify whether collapse/split functions free their input allocations. Risk of double-free or leak if caller makes wrong assumption.
