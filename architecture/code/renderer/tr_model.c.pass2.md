# code/renderer/tr_model.c — Enhanced Analysis

## Architectural Role

This file bridges **asset loading** and **scene composition**. It manages the model registry—a critical junction where the renderer's lifecycle (`RE_BeginRegistration` → `RE_RegisterModel` → runtime queries) intersects with shader system (`R_FindShader` registrations during load) and hunk allocator. MD3 and MD4 support coexist within a discriminated union (`model_t.type`), allowing the engine to handle both static vertex-animated meshes and skeletal bone-weighted models without separate code paths for culling/bounds.

## Key Cross-References

### Incoming (who depends on this file)
- **cgame VM** (`code/cgame/cg_ents.c`, `cg_players.c`): calls `R_LerpTag` per-frame for attachment points (weapon muzzles, item pickups, nameplates); calls `R_ModelBounds` for local entity culling
- **Renderer init** (`code/renderer/tr_init.c` via `R_ModelInit`): seeds the null/bad model at slot 0 during renderer startup
- **Engine public API** (`refexport_t`): `RE_RegisterModel` and `RE_BeginRegistration` exposed to game module during map load
- **Server** (`code/server/sv_game.c`): indirectly reads model bounds via cgame queries for entity visibility culling

### Outgoing (what this file depends on)
- **Shader system** (`code/renderer/tr_shader.c:R_FindShader`): called once per surface during load to pre-register materials; shaders must exist or defaults are used
- **Hunk allocator** (`code/qcommon/common.c`): all model data (`md3[]` pointers, `md4` pointer) allocated from `h_low` dual-ended buffer; ties model lifetime to `RE_BeginRegistration`→`RE_EndRegistration` cycle
- **Filesystem** (`code/qcommon/files.c:FS_ReadFile`): LOD variant lookup with suffixes (`_1.md3`, `_2.md3`); physical file I/O is the only disk access
- **Utility layer** (`code/game/q_shared.c`): `Q_strncpyz`, `Q_strlwr`; (`code/game/q_math.c`): `VectorNormalize`, axis functions
- **Global renderer state** (`tr_local.h`): reads/writes `tr.models[]` array, `tr.numModels` counter; calls `R_SyncRenderThread` for MT safety

## Design Patterns & Rationale

**Handle Indirection**: Model handles (1-based indices) decouple virtual filesystem names from in-memory pointers, enabling fast registry lookup and asset hot-reload without invalidating caller references.

**Registry + Failure Caching**: Rejected models are marked `MOD_BAD` but retained in `tr.models[]`—trades 4 bytes per failed asset to avoid O(n) filesystem rescans. Idiomatic for late-1990s asset pipelines with slow disk I/O.

**Byte-Swapping by Macro**: The `LL()` macro (presumably `LittleLong`) assumes files are always stored in little-endian; loader swaps on big-endian platforms. Reflects era when SGI/PPC servers co-existed with x86 client base.

**Dual Format Union**: MD3 (multi-LOD, per-frame vertex animation) and MD4 (skeletal, LOD-local, bone-weighted) coexist in a discriminated union via `model_t.type`. Allows single rendering pipeline to handle both without branching at draw time—instead, surface identifiers (`SF_MD3`, `SF_MD4`) route to format-specific `tr_surface.c` handlers.

**LOD Mirroring**: If only LOD 2 loads, it's copied to LODs 0 and 1. Allows runtime `r_lodbias` changes without forcing reload; modern engines compute LOD selection per-object based on screen coverage instead.

## Data Flow Through This File

1. **Load Phase** (`RE_RegisterModel`):
   - Name lookup in registry; cache hit returns handle immediately
   - Allocate empty `model_t` slot
   - Iterate LOD suffixes from high→low (`_2.md3`, `_1.md3`, base)
   - Per-LOD: `FS_ReadFile` → format detect (`ident` field) → dispatch to `R_LoadMD{3,4}`
   - Each loader: hunk-allocates, byte-swaps in-place, calls `R_FindShader` for surfaces, validates limits
   - Missing low LODs duplicate from next-higher LOD

2. **Query Phase** (`R_LerpTag`, `R_ModelBounds`):
   - `R_GetModelByHandle` → array lookup
   - `R_LerpTag`: linearly interpolate tag (origin + 3×axis) between frames; independent per-axis normalization
   - `R_ModelBounds`: extract AABB from frame 0 (or brush model bounds)

3. **Rendering Phase** (not in this file):
   - `tr_main.c` calls `R_GetModelByHandle` for each entity surface
   - `tr_mesh.c` dispatches on `surface.ident` to either MD3 or MD4 vertex unpacking

## Learning Notes

**Idiomatic to 1999–2005 Quake**:
- Per-frame vertex animation (MD3) was standard before GPUs had shader ubiquity. Modern engines use skeletal animation with GPU skinning.
- Tag attachment (named points with rotation matrix) predates inverse-kinematics and constraint systems; still elegant for props and weapon muzzles.
- Stateless `R_LerpTag` (no skeleton state) and `R_ModelBounds` (frame 0 only) suggest client doesn't cache interpolation results—recalculated every frame. No performance cost given mid-2000s hardware.
- LOD suffix convention (`_1.md3`) mirrors Q2 toolchain; no per-LOD threshold configuration in code—hardcoded `r_lodbias` cvar would select dynamically (see commented-out code).

**Modern Contrast**:
- Skeletal-only with GPU vertex weighting (MD4 underutilized in this codebase)
- LOD selection driven by bounding-sphere distance to frustum, not static assignment
- Runtime shader specialization (e.g., low-end vs. high-end shaders) instead of asset duplication

## Potential Issues

1. **MD4 Frame Pointer Bug** (`R_LoadMD4`, line ~412): `frame` is declared but never initialized before the loop that assigns it inside the loop body. This is harmless because the loop always assigns before use, but it's suspicious C—should initialize to `NULL` or compute the first pointer before the loop.

2. **Unchecked File Offset Bounds**: Both `R_LoadMD3` and `R_LoadMD4` assume file offsets (`ofsFrames`, `ofsSurfaces`, etc.) are valid. Corrupted files with out-of-bounds offsets could read past the buffer. A check like `if (offset + size > fileSize)` before pointer arithmetic would harden this.

3. **MD4 Vertex Stride Assumption**: Vertices are manually advanced as `&v->weights[v->numWeights]`, assuming `weights` is packed. If a vertex has 0 weights, the pointer may not advance correctly depending on structure packing. Not explicitly validated.

4. **LOD Shallow Copy**: LOD mirroring copies only the pointer (`mod->md3[lod] = mod->md3[lod+1]`), not the data. If hunk is ever compacted or a higher LOD is freed, all mirrored slots become dangling. Unlikely given `RE_BeginRegistration` bounds hunk lifetime, but fragile.

5. **No Duplicate Asset Detection**: If the same model is registered multiple times via different path names (e.g., `models/foo.md3` vs. `Models/Foo.md3` on case-sensitive FS), each gets a separate slot. Case-insensitive hash-table lookup would prevent this, but `strcmp` is case-sensitive.
