# code/bspc/map_q1.c — Enhanced Analysis

## Architectural Role
This file is a **translation bridge** in the offline BSPC compilation pipeline, converting Quake 1 and Half-Life BSP files into the engine's internal brush representation for AAS (bot navigation) and map export. It sits at the input stage of the tool, consuming Q1 BSP lumps and reconstructing solid/liquid geometry via recursive BSP tree decomposition before forwarding to the AAS processor (`aas_map.c`) or standard map export. Unlike the runtime engine (which only understands Q3 BSPs), BSPC's multi-format support was essential for legacy content migration during Q3A's early competitive scene.

## Key Cross-References
### Incoming (who calls this file)
- **code/bspc/bspc.c** — Main tool entry point; calls `Q1_LoadMapFromBSP` when input file is identified as Q1/HL BSP
- **code/bspc/map.c** — Shared brush utilities (`AddBrushBevels`, `MakeBrushWindings`, `TryMergeBrushes`) reused for both Q1 and Q3 processing

### Outgoing (what this file depends on)
- **code/bspc/aas_map.c** → `AAS_CreateMapBrushes` — Core AAS integration; all non-culled brushes are handed here for bot-world mesh generation
- **code/bspc/l_bsp_q1.c** — Parsed Q1 BSP lumps (`q1_dleafs[]`, `q1_dnodes[]`, `q1_dfaces[]`, `q1_dplanes[]`); all spatial and texture data sources
- **code/bspc/map.c** — Brush primitives and utilities (`BrushFromBounds`, `AllocBrush`, `FreeBrush`, `BoundBrush`, `BrushVolume`)
- **Global arrays** — Writes into `mapbrushes[]`, `mapbrushsides[]`, `map_texinfo[]` (shared with Q3 pipeline); updates `map_mins`/`map_maxs`

## Design Patterns & Rationale

**Recursive BSP Decomposition** (`Q1_CreateBrushes_r`)
- Classic spatial-tree algorithm: split a bounding box at each node plane, recursively recurse on both halves, accumulate leaf fragments with their content types
- Rationale: Direct translation of the BSP tree structure itself; no need to parse face lists or manually reconstruct topology
- Trades CPU (many splits) for simplicity (no triangle/edge assembly)

**Greedy Face-Overlap Texturing** (`Q1_TextureBrushes`)
- For each brush side, find the BSP face with largest overlapping area on the same plane; assign that face's `texinfo`
- Rationale: Q1/HL face lists are sparse and unordered; brute-force overlap is pragmatic when face count is low (< 10k)
- Weakness: **inherently ambiguous** if multiple faces have similar overlap; no semantic hint to disambiguate

**Content Type Enforcement** (`Q1_FixContentsTextures`)
- Ensures that liquid brushes (CONTENTS_WATER/SLIME/LAVA) have matching texture prefixes (`*`) on all sides
- Rationale: Q1 BSP may have geometry/texture mismatches due to map compiler quirks; engine runtime expects consistency
- Pragmatic fix rather than validation error

**Degenerate Elimination**
- Systematic checks post-split: volume < 1, bounds outside ±4096, fewer than 3 sides → discard
- Rationale: BSP decomposition can produce tiny slivers from numerical error or edge cases; silently culling avoids downstream crashes
- Logged as "clip brushes" (`q1_numclipbrushes`) for user awareness

## Data Flow Through This File

```
Input: Q1 BSP file (q1_dleafs[], q1_dnodes[], q1_dfaces[], q1_dplanes[])
  ↓
Q1_LoadMapFromBSP (per-level)
  ├─ Q1_LoadBSPFile (from l_bsp_q1.c)
  ├─ Q1_ParseEntities (extract entity keyvalues)
  └─ For each entity with valid modelnum:
       Q1_CreateMapBrushes (per-entity orchestrator)
         ├─ Q1_CreateBrushes_r (recursive BSP walk → geometry)
         │   └─ Q1_SplitBrush (split at each node plane)
         ├─ Q1_TextureBrushes (assign texinfo via face overlap)
         ├─ Q1_FixContentsTextures (ensure liquid consistency)
         ├─ Q1_MergeBrushes (optional consolidation)
         └─ Q1_BSPBrushToMapBrush (convert to mapbrush_t)
              ├─ AAS_CreateMapBrushes (if create_aas flag)
              └─ MakeBrushWindings + AddBrushBevels (if standard export)
  ↓
Output: mapbrushes[], mapbrushsides[] (and optionally .aas file)
```

**Content transformation:**
- **Q1_CreateBrushes_r** preserves topology (plane equations, winding order) but discards texture/face identity
- **Q1_TextureBrushes** reintroduces texture identity by matching against original face list  
- **Merge phase** combines same-content adjacent brushes (reduces 10k+ fragments to ~1k) before AAS processing
- **Export phase** either feeds to AAS (for bot compilation) or standard map pipeline (geometry, windings, bevels)

## Learning Notes

**Multi-Format Translation in Game Engines**
- Q3A inherited Q1/Q2 support from Quake 2's tool ecosystem. This file exemplifies the **translator** pattern — reading foreign format, reconstructing via a canonical intermediate (internal brush representation), then exporting.
- Modern engines (Unreal, Unity) avoid this complexity by **committing early** to a single canonical format (USD, GLTF, etc.).

**Recursive Spatial Decomposition is Elegant but Expensive**
- The BSP tree walk is O(depth × sides_per_brush). A well-balanced Q1 map might generate 10k–50k brush fragments before merging.
- Each split allocates new brush structs and copies windings — high memory churn. The `FreeBrush`/`AllocBrush` pattern is a mitigation, but modern tools use **streaming geometry** or **deferred conversion**.

**Texture Matching as an Ambiguous Problem**
- The file's largest sub-problem is: "Given a planar brush side, which Q1 BSP face should texture it?" The greedy area-overlap heuristic is pragmatic but:
  - Fails if faces are coplanar but non-overlapping (seams)
  - Fails if texture coordinates don't align (rotation/scale issues)
  - Modern editors use **explicit UV painting** at authoring time; remapping is less common.

**Epsilon Tolerance Tuning for Format Quirks**
- Note the `epsilon = 0` in `Q1_SplitBrush` vs. `PLANESIDE_EPSILON` elsewhere. This was likely tuned via bug reports from Half-Life community maps with unusual (non-axis-aligned, very thin) geometry.
- Such **format-specific magic constants** are a maintenance burden; modern tools parameterize them.

**Connection to Modern ECS/Bot Navigation**
- AAS (Area Awareness System) is a **precomputed navigation mesh** — distinct from modern runtime navmesh systems (Recast, etc.). This file's output feeds directly into AAS binary serialization.
- Modern bot frameworks generate AAS-equivalent data from final geometry at tool time, not from BSP reverse-engineering; this file's complexity would be unnecessary.

## Potential Issues

**Texture Ambiguity & Silent Discarding**
- `Q1_TextureBrushes` silently skips brush sides with no matching face (increments `q1_numclipbrushes` but continues). This can hide legitimate geometry mismatches or misbehaving texture inference. **Recommendation:** Log warnings for every discarded side so mappers can audit.

**Floating-Point Precision After Many Splits**
- Recursive plane clipping via `ChopWindingInPlace` accumulates FP error. After 20+ splits, winding vertices may diverge from true plane positions. The `0.1` bounds-check epsilon mitigates large errors but doesn't eliminate drift. **Risk:** Misaligned brush topology leading to rendering seams or AAS failures.

**Memory Overhead in Fragment Generation**
- Worst case: a brush split n times generates 2^n fragments (exponential), then the merge phase tries to consolidate them. Early stopping (e.g., prune empty/tiny fragments during recursion) would reduce peak memory. Currently, degenerate elimination happens *post-split*, not *in-tree*.

**Loss of BSP Leaf Information**
- Once a brush is split from the BSP tree, the leaf content (`Q1_CONTENTS_*`) is burned into `brush->side` and never revisited. If texturing logic later determines the brush should be culled, that leaf info is lost. **Edge case:** Potential for cracks if texturing and content logic disagree.
</tml:thinking>

# code/bspc/map_q1.c — Enhanced Analysis

## Architectural Role
This file is a **translation bridge** in the offline BSPC compilation pipeline, converting Quake 1 and Half-Life BSP files into the engine's internal brush representation for AAS (bot navigation) and map export. It sits at the input stage of the tool, consuming Q1 BSP lumps and reconstructing solid/liquid geometry via recursive BSP tree decomposition before forwarding to the AAS processor (`aas_map.c`) or standard map export. Unlike the runtime engine (which only understands Q3 BSPs), BSPC's multi-format support was essential for legacy content migration during Q3A's early competitive scene.

## Key Cross-References
### Incoming (who calls this file)
- **code/bspc/bspc.c** — Main tool entry point; calls `Q1_LoadMapFromBSP` when input file is identified as Q1/HL BSP
- **code/bspc/map.c** — Shared brush utilities (`AddBrushBevels`, `MakeBrushWindings`, `TryMergeBrushes`) reused for both Q1 and Q3 processing

### Outgoing (what this file depends on)
- **code/bspc/aas_map.c** → `AAS_CreateMapBrushes` — Core AAS integration; all non-culled brushes are handed here for bot-world mesh generation
- **code/bspc/l_bsp_q1.c** — Parsed Q1 BSP lumps (`q1_dleafs[]`, `q1_dnodes[]`, `q1_dfaces[]`, `q1_dplanes[]`); all spatial and texture data sources
- **code/bspc/map.c** — Brush primitives and utilities (`BrushFromBounds`, `AllocBrush`, `FreeBrush`, `BoundBrush`, `BrushVolume`)
- **Global arrays** — Writes into `mapbrushes[]`, `mapbrushsides[]`, `map_texinfo[]` (shared with Q3 pipeline); updates `map_mins`/`map_maxs`

## Design Patterns & Rationale

**Recursive BSP Decomposition** (`Q1_CreateBrushes_r`)
- Classic spatial-tree algorithm: split a bounding box at each node plane, recursively recurse on both halves, accumulate leaf fragments with their content types
- Rationale: Direct translation of the BSP tree structure itself; no need to parse face lists or manually reconstruct topology
- Trades CPU (many splits) for simplicity (no triangle/edge assembly)

**Greedy Face-Overlap Texturing** (`Q1_TextureBrushes`)
- For each brush side, find the BSP face with largest overlapping area on the same plane; assign that face's `texinfo`
- Rationale: Q1/HL face lists are sparse and unordered; brute-force overlap is pragmatic when face count is low (< 10k)
- Weakness: **inherently ambiguous** if multiple faces have similar overlap; no semantic hint to disambiguate

**Content Type Enforcement** (`Q1_FixContentsTextures`)
- Ensures liquid brushes (CONTENTS_WATER/SLIME/LAVA) have matching texture prefixes (`*`) on all sides
- Rationale: Q1 BSP may have geometry/texture mismatches due to map compiler quirks; engine runtime expects consistency

**Degenerate Elimination**
- Systematic checks post-split: volume < 1, bounds outside ±4096, fewer than 3 sides → discard
- Rationale: BSP decomposition can produce tiny slivers from numerical error or edge cases; logged as "clip brushes" for user awareness

## Data Flow Through This File

```
Q1 BSP Input → Q1_LoadMapFromBSP → per-entity loop:
  ├─ Q1_CreateBrushes_r (BSP walk, split at planes) 
  ├─ Q1_TextureBrushes (match BSP faces to sides)
  ├─ Q1_FixContentsTextures (enforce liquid consistency)
  ├─ Q1_MergeBrushes (optional consolidation)
  └─ Q1_BSPBrushToMapBrush
      └─ AAS_CreateMapBrushes or MakeBrushWindings/AddBrushBevels
Output → mapbrushes[], mapbrushsides[], (optionally .aas file)
```

**Key transformations:** Spatial tree → flat brush list → textured brushes → consolidated brushes → export format.

## Learning Notes

**Multi-Format Translation in Game Engines**
- Q3A inherited Q1/Q2 support from Quake 2's tool ecosystem. This file exemplifies the **translator** pattern — reading foreign format, reconstructing via canonical intermediate (internal brush), exporting to target.
- Modern engines (Unreal, Unity) avoid this by committing early to a single format (USD, GLTF); translators are now external tools, not runtime code.

**Recursive Spatial Decomposition: Elegant but Expensive**
- O(tree_depth × sides_per_brush); a well-balanced Q1 map generates 10k–50k fragments before merging
- Each split allocates new brush structs and clones windings — high memory churn. Modern tools use streaming geometry or deferred conversion.

**Texture Matching as Ambiguous Problem**
- "Which Q1 face textures this planar brush side?" is solved by greedy area-overlap, but fails on coplanar non-overlapping faces (seams) or rotated UVs.
- Modern editors use explicit UV painting at authoring time; remapping at tool time is increasingly rare.

**Epsilon Tuning for Format Quirks**
- Note `epsilon = 0` in `Q1_SplitBrush` vs. `PLANESIDE_EPSILON` elsewhere — tuned for Half-Life's non-axis-aligned thin geometry.
- Such format-specific magic constants are maintenance burden; modern tools parameterize them.

## Potential Issues

**Silent Brush Side Discarding**
- `Q1_TextureBrushes` skips sides with no matching face (increments `q1_numclipbrushes` but continues). Can hide geometry mismatches. **Recommendation:** Log per-side warnings.

**Floating-Point Drift After Many Splits**
- Recursive `ChopWindingInPlace` accumulates FP error; after 20+ splits, winding vertices may diverge from plane positions, causing rendering seams or AAS failures.

**Exponential Fragment Generation**
- Worst case: brush split n times generates 2^n fragments, then merge phase consolidates. Early culling during recursion would reduce peak memory vs. post-split elimination.
