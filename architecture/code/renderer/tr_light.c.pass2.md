# code/renderer/tr_light.c — Enhanced Analysis

## Architectural Role

This file implements the **static light grid sampling + dynamic light accumulation** subsystem of the renderer's front-end pipeline. It bridges the world's precomputed trilinear light grid (built offline) with per-entity lighting calculations, producing `ambientLight`, `directedLight`, and `lightDir` vectors that the shader backend uses during surface shading. Combined with dlight bitmask distribution into surface structures, it enables the renderer to selectively shade surfaces under dynamic light influence without quadratic per-surface/per-dlight iteration.

## Key Cross-References

### Incoming (who depends on this file)
- **`R_DlightBmodel`** is called from the front-end surface collection phase, likely from `R_AddBrushModelSurfaces` in `tr_bsp.c` or `tr_world.c`, before surfaces are added to the render queue
- **`R_SetupEntityLighting`** is called per-entity during scene traversal (before surfaces are rendered), integrating with the main entity loop in `tr_main.c`
- **`R_LightForPoint`** is a public API for arbitrary-point light sampling; likely used by cgame/game VMs or level tools
- The **`dlightBits` surface field** populated by `R_DlightBmodel` is read downstream in the shader backend (`tr_shade.c`, `tr_shade_calc.c`) to limit dlight iteration per surface

### Outgoing (what this file depends on)
- **`tr.world->lightGridData`** — precomputed light grid built by the map compiler (`q3map`), stored in the BSP world; structured as 8-byte cells (3 ambient RGB, 3 directed RGB, 2 encoded angles)
- **`tr.sinTable`, `FUNCTABLE_SIZE`, `FUNCTABLE_MASK`** — precomputed sine table initialized in `tr_init.c` for fast trigonometry in light direction decoding
- **`tr.identityLight`, `tr.sunDirection`** — global lighting parameters; fallback sun direction for worlds without light grids
- **`tr.or`, `tr.currentEntity`, `tr.smpFrame`** — renderer state: current entity orientation, entity being processed, SMP frame index
- **`r_ambientScale`, `r_directedScale`, `r_debugLight`** cvars — registered in `tr_init.c`; provide artist tuning and debug control
- **Math utilities** (`VectorSubtract`, `DotProduct`, `VectorNormalize`, etc.) — shared math library (likely `q_math.c`)
- **`myftol`** — platform-specific fast float-to-int (x86 inline asm or equivalent); used for lighting packing

## Design Patterns & Rationale

### Trilinear Interpolation (Classic Real-Time Graphics)
`R_SetupEntityLightingGrid` uses a **3D trilinear sampling** pattern across the 8 neighbors of the discretized grid cell containing the entity. The loop factor `(1<<j)` selects which corner is sampled; `frac[]` holds the blend weights. This is a standard offline → online handoff: the map compiler burns light values into a coarse lattice; the renderer interpolates smoothly at runtime.

**Why?** Dramatic memory savings and cache efficiency vs. per-texel light maps. A typical large Q3 map's light grid might occupy 10–50 MB vs. 200+ MB for per-surface light maps.

### Dlight Bitmask Distribution  
Rather than storing an array of dlight indices per surface, `R_DlightBmodel` uses a single 32-bit mask (`dlightBits`). The shader backend then loops `for (i=0; i<32; i++) if (mask & (1<<i)) { apply dlight i }`. This avoids pointer chasing and per-surface allocation.

**Why?** Limits theoretical dlights to 32 (practical upper bound in Q3 anyway due to network bandwidth), enables O(1) lookup and bulk rejection in the backend.

### SMP Double-Buffering via `smpFrame`  
The `dlightBits` storage includes `dlightBits[tr.smpFrame]` — a two-slot buffer. The front-end writes to one index while the back-end reads the other. This avoids race conditions if front-end and back-end run on separate threads.

**Why?** Q3 supports optional **SMP rendering**: front-end traversal and back-end GL execution on separate cores. Synchronization is minimal (sleep/wake), not per-surface locking.

### Guard Variable (`lightingCalculated`)  
`R_SetupEntityLighting` checks and sets `ent->lightingCalculated` to skip redundant work if an entity appears in multiple PVS cells or frame sections.

**Why?** Entities can be referenced from multiple view frustum sections; prevent N-way recomputation of identical ambient/directed values.

### Graceful Degradation for No-Light Maps  
Both `R_SetupEntityLightingGrid` and `R_LightForPoint` check for NULL `tr.world->lightGridData`. When absent, lighting falls back to a fixed `identityLight * 150` + sun direction.

**Why?** Supports debug/testing maps compiled with `-nolight` flag; the renderer remains functional without crashing.

## Data Flow Through This File

```
[Input]
  Entity origin (or lightingOrigin) 
  → Discretize into lightGrid cell coordinates
  → Fetch 8 surrounding grid cells
  → Trilinear blend → ambientLight, directedLight, lightDir
  
[Parallel Path]
  Active dlights array (from refdef)
  → Test AABB overlap with bmodel
  → Build bitmask of overlapping dlights
  → Stamp into each surface's dlightBits[smpFrame]

[Output]
  Entity::ambientLight, directedLight, lightDir (RGB + scalar)
  Entity::ambientLightInt (packed byte representation for fast lookup)
  Per-surface dlightBits (read by shader backend during shading)
```

The light grid direction encoding (lat/lng angles) is **decoded on-the-fly** using `tr.sinTable` lookups, then accumulated as a weighted normal vector. This avoids storing 3 floats per grid cell; 2 bytes encode direction with ~5° precision.

## Learning Notes

### Idiomatic Patterns in Real-Time Rendering (mid-2000s era)
1. **Precomputed Light Grids:** Offline computation of volumetric lighting at coarse resolution, enabling smooth interpolation at runtime. Modern engines use **lightmaps** (per-surface) or **Radiance Cascades** / **Signed Distance Fields**. Q3's trilinear grid was a middle ground: memory-efficient, smooth, artist-tunable.
2. **Dlight as Bit Mask:** Elegant fixed-size constraint. Modern deferred renderers handle many lights via per-pixel lists or bindless textures; Q3 was bandwidth-constrained.
3. **SMP Double-Buffering:** Explicit synchronization pattern for separated front/back threads, circa ~2005. Modern engines use atomic queues or lockless data structures; Q3's model was simpler and sufficient.

### Contrast with Modern Engines
- **ECS engines** would separate lighting as a standalone system queried by shade passes, with no entity-owned storage.
- **Forward+ / Deferred pipelines** would cull lights per tile/cluster; no per-surface bitmask needed.
- **GPU-resident lighting** would store all light sources in a buffer and shade entirely on GPU.
- **Real-time global illumination** (Radiance Cascades, SVOGI) would replace the static grid with dynamic updates.

### Connection to Game Engine Concepts
This file exemplifies **lighting decoupling**: static (grid) and dynamic (dlights) lighting are computed independently, then combined. The grid is **view-independent** (sampled per-entity, not per-pixel), making it cheap but coarse. Dlights are **per-light** (accumulated into a vector), not per-texel, enabling hundreds of dynamic lights without memory explosion.

## Potential Issues

### Minor: Assert vs. Null Check Inconsistency
- `R_SetupEntityLightingGrid` asserts `tr.world->lightGridData` with a comment "bk010103 - NULL with -nolight maps"
- `R_LightForPoint` explicitly checks for NULL and returns `qfalse`

**Implication:** The assert would crash in debug builds with `-nolight` maps, but the public API correctly handles it. The intended flow is that the caller checks `RDF_NOWORLDMODEL` in `R_SetupEntityLighting` before calling the grid function. Not a bug, but the assert is defensive rather than truly necessary. (The comment suggests a historical crash; the check was added retroactively.)

### Edge Case: `totalFactor < 0.99`  
If all 8 samples in a grid cell are in walls (zero ambient/directed), `totalFactor` becomes 0, and the renormalization is skipped. The entity retains default black lighting. This is correct, but the threshold `< 0.99` is somewhat arbitrary—it's just "if we skipped any samples, renormalize."

---

## Summary

`tr_light.c` is a **lighting I/O layer**: it consumes the BSP world's precomputed light grid and accumulates dynamic lights, producing per-entity lighting vectors consumed by the shader backend. Its design (trilinear sampling, bitmask distribution, SMP-safe buffering) reflects constraints of mid-2000s real-time rendering: limited bandwidth, limited texture memory, and the need for smooth lighting from coarse data. The code is clean, well-commented, and pedagogically valuable for understanding offline→online lighting handoffs in classical renderers.
