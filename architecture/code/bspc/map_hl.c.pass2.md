# code/bspc/map_hl.c — Enhanced Analysis

## Architectural Role
This file is a **format converter** within the offline BSPC tool pipeline, translating Half-Life BSP binaries into the engine's intermediate `bspbrush_t` representation. It bridges the gap between the HL BSP tree (stored in global `hl_d*` arrays loaded by `l_bsp_hl.h`) and the Q3/Q2 brush ecosystem used by the rest of BSPC—specifically, it feeds `HL_CreateMapBrushes` output to `HL_TextureBrushes` and ultimately `AAS_CreateMapBrushes` (from `aas_map.h`), which generates the final AAS navigation mesh.

## Key Cross-References

### Incoming (Callers)
- **`bspc.c`** (main BSPC entry point) calls `HL_LoadMapFromBSP` during the map-loading phase when a `.bsp` file's magic matches the Half-Life signature.
- Referenced globals set by `l_bsp_hl.h` (module that owns BSP file I/O): `hl_dnodes`, `hl_dleafs`, `hl_dplanes`, `hl_dmodels`, `hl_dfaces`, `hl_texinfo`, `hl_dedges`, `hl_dvertexes`, etc.

### Outgoing (Dependencies)
- **Brush/geometry utilities** (`qbsp.h` layer): `CopyBrush`, `BrushFromBounds`, `AllocBrush`, `FreeBrush`, `BrushVolume`, `TryMergeBrushes`, `SplitBrush`
- **Winding operations** (`qbsp.h`): `BaseWindingForPlane`, `ChopWindingInPlace`, `ClipWindingEpsilon`, `CopyWinding`, `FreeWinding`, `WindingArea`, `WindingIsTiny`, `WindingIsHuge`
- **Plane utilities** (`qbsp.h`): `FindFloatPlane`, `BrushMostlyOnSide`, `BoundBrush`
- **AAS integration** (`aas_map.h`): `AAS_CreateMapBrushes` called conditionally via `create_aas` cvar
- **Global arrays** written: `map_texinfo[]`, `mapbrushes[]`, `brushsides[]` (populated by `HL_BSPBrushToMapBrush`)
- **Global bounds** updated: `map_mins`, `map_maxs` via `AddPointToBounds`

## Design Patterns & Rationale

**Recursive Divide-and-Conquer (HL_SplitBrush → HL_CreateBrushes_r)**  
The file reconstructs solid geometry by inverting the BSP tree: starting with a massive brush covering the entire world, it recursively splits along each node's plane, assigning content types at leaf boundaries. This **iterative brushification** approach is necessary because Half-Life's BSP format does not store explicit brush geometry—only the spatial partitioning tree. The recursive descent `HL_CreateBrushes_r` mirrors the in-engine collision traversal in `qcommon/cm_*.c`.

**Tolerance for Degenerate Geometry**  
`HL_SplitBrush` explicitly comments "modified for Half-Life because there are quite a lot of tiny node leaves." It tolerates splits that produce only one valid half, logs warnings, and falls back to copying the input brush rather than failing—pragmatic for handling real-world HL maps with artifact leaves.

**Post-hoc Texture Assignment**  
`HL_TextureBrushes` solves a mismatch: the HL BSP stores per-face texture info, but the brushified geometry has arbitrary per-side windings. The solution is **geometric matching**: for each untextured side, find the face with maximum overlap area via `HL_FaceOnWinding`, then assign its texinfo. If multiple faces claim the same side (conflicting), split the brush to isolate regions—trading brush count for correctness.

**Content Type Classification**  
`HL_TextureContents` maps texture name prefixes (`!lava`, `!slime`, `@`, etc.) to Q2-style content enums. `HL_FixContentsTextures` ensures water/slime/lava brushes have a matching texture on all sides (for lightmap/shader purposes at runtime).

## Data Flow Through This File

1. **Load & Parse** (`HL_LoadMapFromBSP`): File I/O delegates to `l_bsp_hl.h`; entities parsed into `entities[]` array.
2. **Per-Model Pipeline** (loop over `entities` with `model` keys):
   - `HL_CreateBrushesFromBSP`: Recursively splits world-bounding brush along BSP tree → linked list of untextured `bspbrush_t` with `contents` set.
   - `HL_TextureBrushes`: Match HL faces to brush sides by overlap; split brushes if conflicting textures detected; populate `texinfo` indices.
   - `HL_FixContentsTextures`: Scan all sides; if content (water/slime/lava) mismatches texture, find matching texture in `map_texinfo[]`.
   - `HL_MergeBrushes` (optional, if `!nobrushmerge`): Iteratively fuse adjacent brushes of identical content type.
   - `HL_BSPBrushToMapBrush`: Convert final `bspbrush_t` list to `mapbrush_t` entries in global arrays; optionally invoke `AAS_CreateMapBrushes`.
3. **Output**: Global `mapbrushes[]` array populated; ready for next BSPC pipeline stage (AAS compilation, lighting, etc.).

## Learning Notes

- **No runtime involvement**: This is **offline compilation only**. The file has no in-game counterpart; contrast with `code/qcommon/cm_*.c` (runtime collision) which uses similar geometry primitives.
- **Format impedance mismatch**: The HL BSP tree encodes space partitioning, not brushes. This file **inverts** that representation—a common pattern when converting between spatial data structures.
- **Tolerance as a feature**: Logging warnings (`Log_Print`) but continuing on degenerate splits reflects BSPC's philosophy: compile everything possible; let the user inspect logs for issues. Modern engines might error hard.
- **Idiomatic Q3/Q2**: Content types (`CONTENTS_WATER`, `CONTENTS_LAVA`) and texinfo indexing (`texinfo_t`) follow the Q2/Q3 convention, not Half-Life's native format. This is an **adaptor layer**.
- **Greedy matching**: Texture assignment by maximum overlap is heuristic; ambiguous or pathological face layouts could yield incorrect results. No backtracking or optimization.

## Potential Issues

- **Texture assignment brittleness**: If no face overlaps a brush side, `HL_TextureBrushes` leaves it untextured; `HL_FixContentsTextures` patches water/slime/lava but not others. Orphaned sides may cause lightmap/shader issues downstream.
- **Brush explosion**: Dense HL BSPs with many tiny leaves could produce thousands of brushes; merging (`HL_MergeBrushes`) helps but may not eliminate redundancy if content types don't align.
- **Content type fixup assumption**: `HL_FixContentsTextures` assumes a matching texture exists in `map_texinfo[]`; if not found, it logs a warning but does not add one. This could cause a water brush to render as solid if no water texture was ever referenced.
