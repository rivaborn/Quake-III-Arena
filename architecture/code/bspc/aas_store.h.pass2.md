# code/bspc/aas_store.h — Enhanced Analysis

## Architectural Role
This header sits at the intersection of **BSPC** (the offline compiler) and **botlib** (the runtime navigation library), defining the memory allocation contract and serialization API that both subsystems must honor. BSPC pre-allocates all AAS arrays to fixed limits during compilation; at runtime, botlib loads the pre-compiled binary via identical code paths, reading from files instead of writing to them. The `BSPCINCLUDE` guard allows the same type definitions to be shared between the offline tool and the library without pulling in runtime-only infrastructure.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — Main BSPC entry point calls `AAS_AllocMaxAAS()` at startup and `AAS_FreeMaxAAS()` at shutdown; orchestrates the full compile pipeline that populates `aasworld`
- **`code/bspc/aas_store.c`** — Implementation of storage functions; reads/writes the global `aasworld` singleton to disk
- **Geometry construction passes** (`code/bspc/aas_create.c`, `aas_map.c`, `aas_gsubdiv.c`, etc.) — Populate `aasworld` arrays and call `AAS_FindPlane()` during face/edge deduplication
- **`code/botlib/be_aas_main.c`** — Runtime equivalent that calls `AAS_Setup()` and later `AAS_LoadMap()`, reusing the same `aasworld` global but from a loaded file instead of newly-constructed data

### Outgoing (what this file depends on)
- **`code/game/be_aas.h`** — Travel flags, trace structures, client move types consumed by AAS runtime queries
- **`code/botlib/be_aas_def.h`** — Complete `aas_t` struct definition and all sub-types (`aas_bbox_t`, `aas_area_t`, `aas_reachability_t`, `aas_node_t`, `aas_portal_t`, `aas_cluster_t`)
- **`q_shared.h`** — Foundational types: `vec3_t`, `qboolean` (pulled in transitively via includes)
- **Singleton storage** — `aasworld` is defined (not declared) in `code/bspc/aas_store.c` and is the central mutable state modified during compilation

## Design Patterns & Rationale

**Fixed-Size Pre-allocation Strategy:**
BSPC uses compile-time constants (e.g., `AAS_MAX_VERTEXES=512000`) to reserve worst-case memory upfront. This avoids fragmentation during multi-pass compilation and simplifies cleanup (single bulk free). The trade-off is that large/complex maps can exhaust limits and fail compilation—there is no dynamic growth.

**Conditional Compilation Guard (`BSPCINCLUDE`):**
Before including `be_aas_def.h`, the macro `BSPCINCLUDE` is set. This suppresses botlib runtime headers (reachability caches, entity links, routing state) that are irrelevant in the offline context. The same guard pattern is used in botlib headers to avoid circular or unnecessary includes during BSPC compilation.

**Singleton + Deferred Initialization:**
`aasworld` is a single global `aas_t` struct. Memory for its sub-arrays is allocated once via `AAS_AllocMaxAAS()` and shared across all compilation passes. This mirrors botlib's runtime model, where a single `aasworld` is loaded from disk and persists for the session.

**Plane Deduplication via `AAS_FindPlane()`:**
During geometry construction, faces reference planes. To avoid duplicate plane storage, `AAS_FindPlane()` searches the existing plane table and returns an index if found, or allocates a new one. This is a classic space-optimization pattern for offline geometry processing.

## Data Flow Through This File

```
BSPC startup
    ↓
AAS_AllocMaxAAS()  [reserves all arrays at max capacity]
    ↓
Geometry passes (aas_create.c, aas_map.c, aas_gsubdiv.c, …)
    ├─ calls AAS_FindPlane() to deduplicate planes
    ├─ populate: vertexes, edges, faces, areas, nodes, etc.
    └─ populate: bboxes, portals, clusters, reachability links
    ↓
AAS_StoreFile("path/to/output.aas")  [serialize aasworld to disk]
    ↓
AAS_FreeMaxAAS()  [release heap]
    ↓
BSPC shutdown
```

At runtime, botlib's `AAS_LoadAASFile()` reads this binary format back into an identically-structured `aasworld` global, but the allocation and storage logic is nearly identical (both defined in `code/botlib/be_aas_file.c` and `code/bspc/aas_file.c`).

## Learning Notes

**Offline-to-Runtime Symmetry:**
The BSPC tool and botlib library are designed as mirror images: BSPC constructs and serializes AAS data; botlib loads and queries it. By reusing the same header and struct definitions, the engine avoids duplication of AAS data structures, but the *process* is split (offline compilation vs. runtime loading).

**Why Pre-Allocation, Not Dynamic Growth:**
In the early 2000s when Quake III was written, predictable worst-case memory footprint was valued over the flexibility of dynamic resizing. BSPC could run on a single machine for hours; botlib needed to fit in limited RAM alongside the engine. Pre-allocating to known limits simplified both memory budgeting and multi-threaded synchronization during compilation.

**Plane Sharing as a Micro-Optimization:**
Modern engines might store planes inline in faces. Quake III's indexed plane table and `AAS_FindPlane()` deduplication reflects the era's emphasis on memory density—planes are a small, reusable resource, so sharing them saves space in very large maps (many faces can reference the same plane).

**Commented-Out `bspc_aas_t` Struct:**
The historical struct shows that BSPC once had its own local mirror of `aas_t`. The move to reusing `aas_t` directly indicates successful unification of the offline and runtime representations—a sign of good architecture maturity.

## Potential Issues

1. **Hard Array Limits as Failure Points:**
   If a map exceeds `AAS_MAX_VERTEXES` or any other limit, BSPC compilation fails with no easy recovery path. Modern engines might use dynamic arrays or streaming. No adaptive sizing or multi-pass fallback exists.

2. **Silent Type Compatibility:**
   Because `BSPCINCLUDE` guards suppress runtime-specific fields in botlib headers, the BSPC-compiled `aas_t` may not include fields that are always initialized at runtime load time (e.g., routing caches). This relies on careful coordination and is not enforced by the type system.

3. **No Versioning or Validation:**
   The header constants are the sole definition of capacity. If constants drift between BSPC and botlib, a compiled `.aas` file from one version might overflow arrays in the other. There is no magic-number or version field to detect this mismatch.

4. **Global Mutable State:**
   `aasworld` is a global non-const struct. Any code with access (both BSPC and botlib) can corrupt it. This is typical of early engines but lacks the isolation of modern dependency-injected architectures.
