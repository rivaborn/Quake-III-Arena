# code/bspc/map_sin.c — Enhanced Analysis

## Architectural Role

This file is a **format adapter** in the bspc offline compilation pipeline, converting Sin engine BSP files into BSPC's internal `mapbrush_t` representation—a crucial preprocessing stage before AAS (Area Awareness System) generation. It sits at the boundary between raw BSP data layout (opaque structs from `sinfiles.h`) and the AAS subsystem's dependency on semantic brush content classifications (DETAIL, TRANSLUCENT, FENCE, etc.). Unlike the commented-out text-map-file functions, the active `#ifdef ME` code path drives the entire BSP→AAS→`.aas` file compilation sequence.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — main entry point; calls `Sin_LoadMapFromBSP` as one of several format-specific loaders
- **`code/bspc/be_aas_bspc.c`** — AAS compilation host; receives populated `mapbrushes[]` after this file's conversion completes
- **`code/bspc/aas_map.c`** — post-brush-conversion stage; calls `AAS_CreateMapBrushes` to convert `mapbrush_t` into AAS collision geometry

### Outgoing (what this file depends on)
- **`code/bspc/qbsp.h`** — core type definitions (`mapbrush_t`, `side_t`, global arrays `mapbrushes[]`, `brushsides[]`, `nummapbrushes`)
- **`code/bspc/l_bsp_sin.h` + `sinfiles.h`** — raw Sin BSP data arrays (`sin_dbrushes`, `sin_dleafs`, `sin_dbrushsides`, `sin_texinfo`, `sin_numtexinfo`)
- **`code/bspc/aas_map.h`** — `AAS_CreateMapBrushes` (called per-brush after conversion)
- **Utility functions** — `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels` (defined elsewhere in bspc tool, not engine runtime)

## Design Patterns & Rationale

### 1. **Iterative BSP Tree Traversal (Defensive Stack)**
The `Sin_SetBrushModelNumbers` function uses a manual `nodestack[]` array instead of recursion. This pattern appears throughout offline tools to avoid stack exhaustion on pathologically deep BSP trees (common in large maps). The `Sin_InitNodeStack`, `Sin_PushNodeStack`, `Sin_PopNodeStack` helpers implement a simple push-down automaton.

**Rationale:** Offline tools must handle worst-case BSP trees without crashing; recursion depth is unpredictable at build time.

### 2. **Multi-Format Adapter Pattern**
The file is one of many (`map_q3.c`, `map_hl.c`, etc.) implementing format-specific BSP readers. Each defines `*_LoadMapFromBSP`, `*_ParseBSPEntity`, `*_BrushContents` in an overloaded namespace. This allows `bspc.c` to switch formats via runtime enum (`loadedmaptype`) without recompilation.

**Rationale:** Q3 engine supported shipping maps in multiple game formats; bspc needed to handle all variants in a single tool.

### 3. **Content Flag Demultiplexing**
The `Sin_BrushContents` function is a semantic **gate** that translates raw texinfo surface flags (`SURF_TRANS33`, `SURF_TRANS66`, `SURF_HINT`, `SURF_SKIP`) into engine-level content bits (`CONTENTS_DETAIL`, `CONTENTS_TRANSLUCENT`, `CONTENTS_FENCE`). This is **not** a simple bitwise copy; it involves:
- Aggregation (summing translucence across sides, ORing flags)
- Conflict resolution (DETAIL vs. non-DETAIL sides)
- Remapping (FENCE → WINDOW + DUMMYFENCE + DETAIL)
- Conditional application (fulldetail cvar, clipping content auto-promotion to SOLID)

**Rationale:** AAS navigation must understand "walkable," "detail," "translucent" semantics to build correct reachability links. Surface-level shader flags are insufficient; brushes need semantic tagging.

### 4. **Sin-Specific Branching via Preprocessor**
Throughout the file, `#ifdef SIN` guards handle Sin engine differences:
- Translucence is a **float** (cumulative) in Sin vs. flags (bitwise) in Q3
- Sin supports detailed entity-key parsing (`ParseSurfaceFile`, `surfacefile`, `MergeRefs`)
- Sin remaps certain flag combinations (FENCE handling)

**Rationale:** By 2005, the engine codebase was heavily polymorphic to support multiple game releases. Preprocessing compile-time variants minimizes runtime branching in a tool that will only load one format per invocation.

## Data Flow Through This File

```
Sin BSP file (on disk)
         ↓
Sin_LoadMapFromBSP()
    - Sin_LoadBSPFile() [extern, loads raw BSP lumps]
    - Sin_ParseEntities() [parses entity key-value pairs]
    - for each entity:
        Sin_ParseBSPEntity()
            - Sin_SetBrushModelNumbers() [DFS BSP tree, map leaf→brush]
            - Sin_ParseBSPBrushes() [iterate brushes in entity's leaf list]
                - Sin_BSPBrushToMapBrush()
                    - Sin_BrushContents() [aggregate texinfo → semantic flags]
                    - MakeBrushWindings() [convert plane equations to windings]
                    - AAS_CreateMapBrushes() [feed to AAS preprocessor]
                    ↓ (appends to mapbrushes[], brushsides[])
    - Sin_CreateMapTexinfo() [copy texinfo[0..N] from BSP]
         ↓
mapbrushes[0..nummapbrushes-1]  (internal representation)
brushsides[0..nummapbrushsides-1]
         ↓
AAS compilation pipeline (aas_create.c, etc.)
         ↓
.aas binary (on disk)
```

**Key state transitions:**
- Brush loads unpopulated; content flags computed on-the-fly from texinfo
- Leaf-to-brush mapping established during BSP tree walk (for func_group, func_moving_door support)
- Model numbers assigned to track which brushes belong to which entities (for later portal/trigger merging)

## Learning Notes

### Quake III Engine Archaeology
1. **Multi-game Binary:** The Q3 codebase was designed to ship one engine binary supporting multiple game formats (Q1, Q2, SiN, HL, Star Wars Jedi Knight II). The `#ifdef SIN` guards are artifacts of this polymorphic era—modern engines use data-driven format plugins instead.

2. **Offline Compilation Burden:** The bspc tool's existence highlights a key limitation of the Q3 architecture: **AAS generation requires specialized offline processing**. Modern engines often compute navigation meshes dynamically at runtime or in-editor; Q3/SiN separated these concerns entirely, requiring developers to run bspc after every map change.

3. **Content Flag Inflation:** The elaborate `Sin_BrushContents` function shows the progression of shader complexity. Early engines (Q1) had simple texture properties. By Q3/SiN, textures carried translucence levels, detail flags, and special semantics (FENCE, CLIP, NONSOLID). The BSP → AAS bridge must **reinterpret** surface-level metadata as volume-level semantic boundaries for navigation.

4. **Brush as Geometric Primitive:** In Q3, brushes are the only runtime collision primitives (outside of skeletal models). Triangles, voxels, and heightfields did not exist. This explains the heavy reliance on brush winding generation and plane-based collision—everything AAS cares about is already baked into brush boundaries.

### Modern Divergence
- **ECS/Component Data:** Modern engines separate geometry from collision from walkability. Q3 tightly couples them via BSP/brush encoding.
- **Runtime Navigation Mesh:** Navmesh-based systems (Recast, Unreal Recast, Unity NavMesh) compute walkability dynamically. Q3 required offline bspc preprocessing, making iteration slower.
- **Data-Driven Serialization:** Modern tools use JSON, YAML, or protobuf for multi-format support. Q3 used preprocessor `#ifdef`, losing type safety and complicating maintenance.

### Key Insight: Semantic Translation Layer
This file embodies a fundamental compiler design pattern: **semantic bridging**. It translates from the **syntactic** realm (raw BSP file layout) to the **semantic** realm (content types the AAS system understands). The separation is clean:
- **Input contract:** `sin_dbrush_t`, `sin_dbrushside_t`, `sin_texinfo_t` (opaque; only the loader understands the binary format)
- **Output contract:** `mapbrush_t`, `side_t` (canonical; all downstream consumers depend on this)
- **Transformation logic:** Content flag rewriting, winding generation, entity assignment

This is analogous to compiler front-ends (parser) → intermediate representation (IR) → backend. Most of the file's code exists precisely because Sin's BSP encoding does not directly map to AAS's semantic model.

## Potential Issues

1. **Translucence Accumulation Bug (Sin):** In `Sin_BrushContents`, translucence is summed across sides as `trans += sin_texinfo[s->texinfo].translucence`. If a brush has, say, 5 translucent sides + 1 opaque side, `trans > 0` triggers `CONTENTS_Q2TRANSLUCENT` for the entire brush. This is likely intentional (any translucent face → translucent brush), but it differs from Q3's flag-based union logic and could be surprising to map authors.

2. **Uninitialized `newref` (Sin):** The `textureref_t newref` struct is memset to zero in `ParseBrush` (commented out), but in the active BSP path, `Sin_BSPBrushToMapBrush` calls `Sin_BrushContents` which reads `sin_texinfo[…].translucence` directly—there's no equivalent `newref` initialization visible in the active code path. This suggests either `newref` is unused in the BSP path or there's dead code.

3. **No Validation of BSP Data:** The file assumes `sin_dbrush_t.firstside` and `numsides` are valid indices into `sin_dbrushsides[]`. No bounds checking occurs. A malformed BSP could cause buffer overruns.

4. **Global State Coupling:** `brushmodelnumbers[]` and `dbrushleafnums[]` must be pre-sized to `MAX_MAPFILE_BRUSHES`. If Sin maps exceed this constant, they silently fail. No assert or error is thrown.

---

**Summary:** This file is a **format-specific data transformer** in a complex offline pipeline. Its role is humble but essential: it reinterprets raw BSP binary formats into a canonical intermediate representation that downstream AAS compilation stages understand. The prevalence of `#ifdef SIN` and content flag rewriting reveals the non-trivial semantic gap between "how maps are stored on disk" and "what the pathfinding system needs to know."
