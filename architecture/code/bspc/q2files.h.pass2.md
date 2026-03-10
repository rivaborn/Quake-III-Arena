# code/bspc/q2files.h — Enhanced Analysis

## Architectural Role

This header serves as a **legacy format bridge** enabling BSPC (the offline AAS compiler) to load and interpret Quake 2 BSP maps alongside Q3A and other game formats. BSPC is a **cross-game compilation tool** that normalizes maps from multiple engine lineages (Q1, Q2, Quake 3A, Half-Life, SiN) into a unified AAS navigation mesh. This file defines the exact binary on-disk layout for Q2 assets, which `l_bsp_q2.c` deserializes during map ingestion. The structures here are never used at runtime—only during the offline pipeline—but they are critical for BSPC's **format plurality**: supporting Q2 maps lets BSPC serve a broader ecosystem of level editors and legacy maps.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/l_bsp_q2.c` / `code/bspc/l_bsp_q2.h`** — Q2 BSP file loader; memory-maps or reads these structures directly from disk
- **`code/bspc/aas_map.c`** — Ingests Q2 brush/entity data post-deserialization; filters using `CONTENTS_*` flags
- **`code/bspc/be_aas_bspc.c`** — Top-level AAS compilation pipeline; orchestrates Q2 BSP → AAS conversion
- **`code/bspc/bspc.c`** — Main tool driver; selects format handler (Q1, Q2, Q3A, etc.) based on file magic
- **`code/botlib/be_aas_file.c` (indirect)** — Reads compiled AAS output; the flags here influence AAS lump structure and classification

### Outgoing (what this file depends on)
- **No outgoing dependencies.** Pure data definition; depends only on C's built-in types (`float`, `int`, `byte`, `short`, `unsigned`).
- **Implicit dependency on alignment/endianness assumptions**: Code assuming little-endian, struct packing with no explicit `#pragma pack`.
- **`q_shared.h` mirrors** — Content/surface flags are mirrored in game-side headers for consistency; renamed `CONTENTS_Q2TRANSLUCENT` to avoid collision with Q3A `CONTENTS_TRANSLUCENT`.

## Design Patterns & Rationale

**Binary Serialization Header Pattern**: All structures are designed for **direct memory mapping** or fixed-offset disk reads. No function pointers, variable-length arrays (except trailing `[1]` as unbounded sentinel), or implicit padding. This was standard in pre-2000s game engines before reflection/marshaling became common.

**Lump-Directory Architecture**: The `dheader_t` → `lump_t` indirection enables **partial parsing** and **optional chunks**. Tools only load lumps they need (e.g., lighting can be rebuilt, so skip `LUMP_LIGHTING`). This scales better than monolithic file formats and was influential in Q3A's own BSP design.

**Content/Surface Flags as Bit-Fields**: Rather than storing per-brush metadata in a hierarchy, each content/surface type is a **single bit**, enabling compact storage and fast boolean tests. The proliferation (14+ flag types) shows organic growth: each new game mechanic added a new flag. This is memory-efficient but not extensible—adding flags requires breaking backward compatibility.

**Precision Trade-off in `dtrivertx_t`**: Vertex positions are stored as **3 bytes** (0–255 range scaled by frame min/max), sacrificing sub-unit precision for **3× storage savings** on skeletal animations. This trade-off is feasible because MD2 is a low-poly format and frame data is read-only at runtime.

**Plane Type Classification** (`PLANE_X`, `PLANE_Y`, `PLANE_Z`, `PLANE_ANYX`, etc.): Pre-computed plane type hints for **fast axis-aligned queries** in collision detection. Axial planes (types 0–2) enable AABB tests; non-axial planes require full dot products.

**Why Q2 Support?** Quake 2 preceded Q3A and shared significant technology. BSPC's multi-game support follows id's philosophy of **tool reuse across the engine family**—the same compilation infrastructure that shipped with Q3A could ingest legacy Q2 maps, extending its utility to level editors and modders.

## Data Flow Through This File

1. **Load Phase** (`l_bsp_q2.c`):
   - File bytes → `dheader_t` (magic, version, lump table)
   - For each lump: read offset/size → allocate memory → deserialize (`dmodel_t`, `dplane_t`, `dnode_t`, `dface_t`, `dbrush_t`, etc.)
   - Validate: `ident == IDBSPHEADER`, `version == BSPVERSION`

2. **Conversion Phase** (`aas_map.c`):
   - Q2 brushes (convex polytopes defined by `dbrushside_t` → planes) → internal area/face representation
   - Q2 entities (parsed from `LUMP_ENTITIES` string blob) → bot spawn points, objectives, teleporters
   - Content/surface flags → classify ground/water/lava/solid for reachability analysis
   - BSP tree traversal using `dnode_t` / `dleaf_t` hierarchy

3. **Output Phase**:
   - Processed geometry → AAS lump structures (areas, faces, edges, reachability links)
   - AAS file written with Q3A format (different on-disk layout)
   - **Q2 format consumed; Q3A format produced** — BSPC is a **translation layer**.

## Learning Notes

**Binary Format Archaeology**: This header reveals how game engines circa 2000 handled multi-format support. Modern pipelines (Unreal, Unity) use:
- Reflection/metadata systems (knowing field types at runtime)
- Versioning/migration code (not hard-coded version numbers)
- Language-agnostic serialization (protobuf, JSON, MessagePack)

Q2's approach was pragmatic for its era: **compile-time structure layout is deterministic**; if your disk file matches your headers, parsing is trivial.

**The Name Collision Fix** (`CONTENTS_Q2TRANSLUCENT` vs. Q3A `CONTENTS_TRANSLUCENT`) shows **namespace hygiene under pressure**. When merging two game's symbols into one tool, renaming is safer than overloading—it prevents subtle flag-value mismatches that would only surface at runtime.

**Cluster-Based PVS**: The visibility lump uses **16-bit cluster indices**, grouping leaves for efficient PVS bitmap compression. This is a spatial partitioning strategy distinct from raw leaf-based PVS and was a key innovation in Q2 engine design.

**Area Portals and Cluster Switching**: Q2's `dareaportal_t` / `darea_t` system decouples visibility (PVS/cluster) from collision (portal culling). A door closing breaks area connectivity even if PVS says you should see through. BSPC must preserve these portal boundaries when generating AAS.

**Comparative Context**: Q1's format was simpler (no areas/clusters); Q3A extended Q2's design with more lumps and finer subdivision. BSPC's ability to ingest all three shows the **gradual evolution** of BSP architecture across a decade.

## Potential Issues

- **Endianness Assumptions**: No `#pragma pack` or `__attribute__((packed))`; struct packing depends on compiler and platform. Cross-platform loading could fail silently if alignment differs.
- **64-Byte Skinname Hardwire**: `MAX_SKINNAME=64` is arbitrary. Texture names longer than 64 chars truncate; no bounds check in deserialization.
- **Fixed Lump Count** (`HEADER_LUMPS=19`): Adding a new lump type requires redefining the entire lump enum, breaking all existing files. Q3A later addressed this with variable-length lump tables.
- **Vertex Compression Loss**: `dtrivertx_t` using `byte` coordinates means precision is frame-dependent; decompression to world space requires both frame's `scale` and `translate`—non-obvious and error-prone.
- **No Versioning Within Lumps**: If a lump's internal format changes (e.g., extended `dface_t`), there's no way to detect it without comparing byte sizes—fragile forward compatibility.
