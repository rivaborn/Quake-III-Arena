# code/bspc/textures.c — Enhanced Analysis

## Architectural Role

This file is a **texture-to-texinfo translation layer** within the BSPC offline map compilation pipeline. It sits at the boundary between brush surface specifications (loaded from `.map` files by parsers like `map_q2.c`, `map_q3.c`) and the final BSP lump arrays. The file orchestrates two parallel caching mechanisms: **miptex metadata** (loaded once per unique texture name via `FindMiptex`) and **computed texinfo records** (deduplicated during `TexinfoForBrushTexture`), feeding a globally accumulated `texinfo[]` array that is eventually written to the BSP file. This is strictly offline; zero runtime role.

## Key Cross-References

### Incoming (Callers)
- **Map parsers** (`code/bspc/map_q2.c`, `map_q1.c`, `map_q3.c`): Call `TexinfoForBrushTexture` when processing brush faces during map load, passing plane, brush texture params, and entity origin
- **Internal recursion**: `FindMiptex` calls itself to register animation texture chains; `TexinfoForBrushTexture` calls itself to resolve `nexttexinfo` pointers
- Entry point is not documented in cross-reference; likely called from `AAS_SetTexinfo` in `code/bspc/aas_map.c` or brush-side processing code

### Outgoing (Dependencies)
- **Global BSP lump arrays** (`l_bsp_q2.h`): Appends to `texinfo[]` (via pointer `tc` at line 214); reads/increments `numtexinfo`
- **Global texture cache** (`qbsp.h`): Reads/writes `textureref[]` and `nummiptex`
- **File I/O** (`l_qfiles.c`): `TryLoadFile` to load `.wal` texture metadata from `gamedir/textures/*.wal`
- **Memory utilities** (`l_mem.c`): `FreeMemory` to release loaded miptex structs
- **Math** (`l_math.h`): `DotProduct` (line 99), `VectorCopy` (lines 106–107)
- **Error handling** (`cmdlib.c`): `Error` (line 51) for fatal MAX_MAP_TEXTURES overflow

## Design Patterns & Rationale

### 1. **Dual-Level Caching**
   - **L1 Cache (`textureref[]`)**: Stores miptex **metadata only** (name, flags, contents, value, animname). Loaded once per unique texture name, populated on-demand by `FindMiptex`.
   - **L2 Cache (`texinfo[]`)**: Stores fully transformed **projection records**. Deduplicates identical texinfo entries to reduce BSP file size.
   - **Rationale**: Separates disk I/O (expensive, one-time) from geometric transformation (cheap, repeated). A map may reference 100 brush faces with the same texture but different rotations/scales; `textureref[tex_id]` is loaded once, but up to 100 `texinfo_t` records may be generated.

### 2. **Brute-Force Deduplication via goto**
   - Lines 209–222: Linear search of `texinfo[]` with nested loop early-exit using `goto skip`. Non-idiomatic by modern standards, but characteristic of late-1990s game engines (era of Quake 2/3 codebase). Avoids function call overhead of a separate comparison routine.
   - **Tradeoff**: O(n) search is acceptable because `numtexinfo` is typically 100–500 in practice; BSP compilation is offline and not latency-critical.

### 3. **Deterministic Planar Projection via Axis Table**
   - `baseaxis[18]` encodes 6 cardinal plane orientations (floor, ceiling, 4 walls) with 3 orthonormal vectors each (normal, S-axis, T-axis).
   - `TextureAxisFromPlane` uses **dot product with plane normal** to select the "closest" cardinal orientation, ensuring:
     - **Deterministic**: Same plane normal → same axis selection across builds
     - **Efficient**: Single dot product per axis (6 comparisons) vs. general matrix computation
     - **Art-friendly**: Artists expect walls to align with world axes; this respects that intent
   - **Limitation**: Only 6 orientations; skewed planes get nearest-fit, potentially causing visible texture skew

### 4. **Recursive Animation Chain Resolution**
   - Line 226: `FindMiptex(textureref[mt].animname)` ensures the entire animation texture sequence is registered
   - Each animated texture is automatically linked via `nexttexinfo` (line 230), forming a `texinfo` linked list
   - **Assumption**: Animation chains are acyclic (game assets enforce this; no cycle detection)

## Data Flow Through This File

```
Map Parser (map_q2.c, etc.)
  ↓ brush face with texture name, scale, rotate, shift, origin
TexinfoForBrushTexture()
  ├─→ FindMiptex(bt->name)  [L1 cache lookup / disk load]
  │    ├─→ TryLoadFile(.wal)
  │    ├─→ LittleLong() [endian swap]
  │    ├─→ FreeMemory()
  │    └─→ FindMiptex(animname) [recursive for animation chain]
  ├─→ TextureAxisFromPlane(plane) [deterministic axis selection]
  ├─→ DotProduct(), sin/cos [transform axes by rotation/scale/shift]
  ├─→ Linear search of texinfo[] for deduplication
  │    └─→ return existing index if found
  └─→ Append new texinfo_t to texinfo[], set nexttexinfo
       ↓
  BSP lump array (ready for writing to .bsp file)
```

**Key State Transitions:**
- `nummiptex`: 0 → N (unique textures encountered)
- `numtexinfo`: 0 → M (deduplicated transformed texinfo records)
- `textureref[]`: filled sparsely with metadata
- `texinfo[]`: filled sequentially with projection records

## Learning Notes

### What This File Teaches
1. **Offline vs. Runtime Layering**: This entire module is compilation infrastructure, not runtime. Similar texture loading happens in the renderer, but here it's done once at compile time for optimal disk layout.
2. **Idiomatic Late-1990s C**: Heavy use of `goto` for loop control, `strcpy` without bounds checking, global state for accumulators (`numtexinfo`). Reflects constraints and practices of the era (pre-C99, pre-security hardening).
3. **Deterministic Output**: Critical for reproducible builds and network consistency. The axis selection algorithm is deterministic; the deduplication is deterministic. Different tools (e.g., q3map vs. BSPC) must compute identical texinfo indices for cross-tool compatibility.

### Divergence from Modern Engines
- **Modern engines** typically use **hash-based UV atlassing** (pack multiple textures into a single large GPU texture), eliminating per-surface texinfo records
- **Per-surface projection** (as done here) is outdated; modern renderers use **vertex UV data** computed by the modeler
- **Animation chains** are now handled by material parameter swapping, not linked lump records
- **Planar projection** is a fallback; procedural UV generation is standard

### Connections to Engine Architecture
- **Offline compilation loop**: Map → BSPC (this file) → AAS compiler → BSP + AAS files
- **Runtime consumption**: Renderer (`tr_bsp.c`) reads `texinfo[]` during BSP load; applies projection to compute surface UV bounds for each poly
- **Game VM independence**: Texture logic is pure; game VM never directly touches texinfo (only indirectly via rendered surfaces)

## Potential Issues

1. **Stack Overflow in Recursive Animation Chains** (Line 226): If an artist creates a cycle in animated texture definitions (e.g., `anim_1 → anim_2 → anim_1`), `FindMiptex` would recurse infinitely. No cycle detection exists. Mitigation: Asset validation in the level editor.

2. **Silent Metadata Loss on Missing `.wal` File** (Lines 56–61): If a texture's `.wal` file is absent (e.g., typo in texture name), `TryLoadFile` returns -1, and `textureref[i]` is left with uninitialized (or zero-initialized) flags/contents/value. No warning is logged. The BSP file is generated, but the texture properties are undefined. Early compilers allowed this; modern tools (q3map2) typically error-halt.

3. **Buffer Overflow in strcpy** (Lines 49, 59): No bounds checking. If texture names exceed the `textureref_t.name` field width (likely 64 bytes from `qbsp.h` definition), a buffer overrun occurs. Mitigated by assumption that map files are curated.

4. **Off-by-One in Deduplication Search** (Line 222): The search compares `tc->texture` (the string pointer in the already-added texinfo) against `tx.texture` (the local struct). Both should point to valid C strings; no issue if `tx.texture` is a fixed-size char array. Verify in `qbsp.h` definition of `texinfo_t`.

Not clearly inferable: Whether `baseaxis` is shared/read by other compilation phases (likely yes, but not documented in the provided context).
