# code/bspc/map_q2.c — Enhanced Analysis

## Architectural Role

This file is the **Q2 map format translation layer** in the bspc offline compiler. It ingests Quake 2 maps in two formats (`.map` text or compiled BSP binaries) and converts them to the internal `mapbrush_t`/`entity_t` representation that feeds into the AAS (Area Awareness System) compilation pipeline. Critically, **bspc is not part of the runtime engine**—it is an offline tool run by level designers/build systems to precompute bot navigation data (`.aas` files) before shipping a map.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/map.c`** — Top-level map dispatch routine calls `Q2_LoadMapFile` and `Q2_LoadMapFromBSP` based on file extension or explicit format selection.
- **bspc tool entry point** — These functions are entry points invoked from the build pipeline, never called at runtime.

### Outgoing (what this file depends on)
- **`aas_map.h` (`AAS_CreateMapBrushes`)** — If `create_aas` flag is set, brushes are forwarded directly to AAS geometry creation instead of standard BSP pipeline.
- **Plane/brush infrastructure** (`qbsp.h` globals): `FindFloatPlane`, `PlaneFromPoints`, `mapbrushes[]`, `brushsides[]`, `mapplanes[]`, `entities[]`
- **Brush utilities** — `FindMiptex`, `TexinfoForBrushTexture`, `MakeBrushWindings`, `AddBrushBevels`, `BrushExists`, `MarkBrushBevels`
- **Q2 BSP loader** (`l_bsp_q2.h`) — `Q2_LoadBSPFile`, `Q2_ParseEntities`, BSP lump globals (`dbrushes`, `dbrushsides`, `dleafs`, `dleafbrushes`, `texinfo`, etc.)
- **Script parser** — `PS_ReadToken`, `PS_ExpectTokenString`, `PS_CheckTokenType`
- **Memory & logging** — `GetMemory`, `FreeMemory`, `Log_Print`

## Design Patterns & Rationale

### 1. **Manual Stack-Based BSP Tree Traversal** (`Q2_SetBrushModelNumbers`)
The BSP tree walk uses an explicit `nodestack[]` instead of recursion. This reflects pragmatic avoidance of stack overflow on large maps—mid-2000s tool chains lacked graceful recursion depth management. Modern engines would use tail-call optimization or bounded iterative algorithms.

### 2. **Dual Format Abstraction**
Both `.map` (text) and `.bsp` (binary) inputs converge on the same internal `mapbrush_t` representation. This decoupling allows loading maps from two sources without duplicating downstream logic. The BSP path (`Q2_LoadMapFromBSP`) additionally populates `brushmodelnumbers[]` and `dbrushleafnums[]` before brush conversion.

### 3. **Conditional AAS-Fast-Path**
The `create_aas` flag enables early dispatch to `AAS_CreateMapBrushes`, bypassing the normal bevel/clipping pipeline. This is a design choice to optimize AAS compilation: for navigation-only compilation, skip BSP-specific geometry refinements.

### 4. **Per-Brush Model Number Mapping**
Quake 2 (and Q3A) support multi-brush entities (e.g., `func_group` combines multiple brushes as one). The `brushmodelnumbers[]` array maps each brush's index to its parent BSP model number. Computed via iterative BSP tree walk, this is critical for entity-brush association during AAS creation.

## Data Flow Through This File

```
Q2 .map file                    Q2 .bsp file
     │                               │
     ├─ Q2_LoadMapFile              ├─ Q2_LoadMapFromBSP
     │  ├─ LoadScriptFile           │  ├─ Q2_LoadBSPFile (lumps: brushes, brushsides, texinfo, planes, entities)
     │  └─ Q2_ParseMapEntity loop   │  ├─ Q2_ParseEntities
     │     ├─ Q2_ParseBrush         │  └─ Q2_ParseBSPEntity per entity
     │     │  ├─ PlaneFromPoints    │     ├─ Q2_SetBrushModelNumbers (iterative BSP walk)
     │     │  ├─ TexinfoForBrushTexture  │  ├─ DPlanes2MapPlanes (BSP plane→map plane mapping)
     │     │  ├─ Q2_BrushContents   │     ├─ Q2_BSPBrushToMapBrush
     │     │  └─ [AAS_CreateMapBrushes]  │  │  ├─ FindFloatPlane
     │     │  or [MakeBrushWindings]     │  │  ├─ Q2_BrushContents
     │     │     AddBrushBevels         │  │  └─ [AAS_CreateMapBrushes]
     │     └─ Q2_MoveBrushesToWorld    │  └─ Q2_CreateMapTexinfo
     │        (for func_group)         │
     └─ Q2_CreateMapTexinfo         └─ PrintMapInfo
        
Output: mapbrushes[], brushsides[], entities[], map_texinfo[]
        → fed to AAS compiler or standard BSP pipeline
```

**Key state mutations**: `nummapbrushes`, `nummapbrushsides`, `num_entities`, all global arrays populated side-effect style.

## Learning Notes

### What This File Teaches

1. **Offline Compilation Pipeline**: Unlike modern engines that bake data at editor-time in-process, Q3A used standalone tools (bspc, q3map). This separation allowed minimal runtime dependencies and reusable preprocessing.

2. **Format Translation as Core Task**: Level designers could edit in `.map` format or import compiled `.bsp` files; the tool transparently handled both. Modern PBR engines often ship custom model-import pipelines for similar reasons.

3. **Multi-Model Entity Handling**: Q2/Q3 brush-based engines required explicit model number assignment for grouped entities. Modern engines use scene graphs or composition; this shows the precursor pattern.

4. **Iterative vs. Recursive Algorithms**: The manual `nodestack[]` in `Q2_SetBrushModelNumbers` is a workaround for recursion depth concerns on 2000s hardware. Modern C/C++ with tail-call optimization or safe recursion depth checking would just recurse.

5. **Pragmatic Dual-Mode Code**: AAS mode (`create_aas` flag) skips BSP refinements; standard mode does full BSP. This reflects tuning for two different use cases (bot nav vs. playable geometry).

### Idiomatic to Q3A's Era

- **String functions without bounds** (`strcpy`, `sprintf`) — standard practice pre-2005
- **Global mutable state** — memory pool approach predates arena allocators and dependency injection
- **Conditional compilation (`#ifdef ME`)** — gates AAS-specific code; Q3A source was repurposed for multiple tools
- **Explicit memory management** — `GetMemory`/`FreeMemory` instead of RAII or garbage collection

## Potential Issues

### Flagged in Code
- **Line ~250**: Comment explicitly notes "IDBUG: mixed use of MAX_MAPFILE_? and MAX_MAP_? this could lead to out of bound indexing"
  - This was a known issue; buffer sizes may not match array declarations
  - Never exploited in released maps, but fragile for edge cases

### Unsafe String Operations
- `strcpy(map_texinfo[i].texture, texinfo[i].texture)` — no bounds checking
- `sprintf(string, "%i %i %i", ...)` into a `char[32]` — potential overflow if origin coords are extreme
- Would require explicit bounds (`strncpy`, `snprintf`) in modern code

### Recursion Depth
- `AAS_CreateMapBrushes` (called conditionally) may recurse; despite the explicit stack for BSP walks, indirect calls may still overflow on pathological geometry
- Not observed as a real issue, but the mixed strategies (some iterative, some recursive) suggest historical caution

### Minor: Incomplete Context Handling
- `Q2_ParseMapEntity` silently ignores entities with malformed key-value pairs (e.g., missing class); no validation pass ensures completeness
- Not a crash risk, but may silently drop map data if the `.map` is corrupted
