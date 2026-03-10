# code/botlib/be_aas_main.h — Enhanced Analysis

## Architectural Role

This header is the **public contract and internal lifecycle gateway** for the Area Awareness System within botlib. It bridges the engine's bot subsystem with the internal AAS network (reachability, routing, clustering). The `AASINTERN` preprocessor guard enforces a strict two-tier API: read-only public predicates and query functions exposed to the server and game code, while lifecycle operations (`Setup`, `LoadMap`, `StartFrame`, `Shutdown`) remain internal to AAS compilation units. This design isolates the mutable singleton `aasworld` from accidental misuse by external code.

## Key Cross-References

### Incoming (who depends on this file)

- **botlib/be_interface.c**: Implements the `botlib_export_t` vtable exposed to the server; wraps public AAS queries (`AAS_Initialized`, `AAS_Loaded`, `AAS_Time`, `AAS_ModelFromIndex`) to route them through `be_aas_main.c` implementations
- **server/sv_bot.c**: Calls `AAS_Initialized()` to guard bot-subsystem initialization; drives per-frame AAS updates via `trap_BotLib*` syscalls
- **botlib/be_aas_reach.c, be_aas_route.c, be_aas_entity.c, be_aas_move.c, be_aas_cluster.c**: All internal AAS subsystems include this header for `AASINTERN` definitions and `aasworld` access
- **bspc/be_aas_bspc.c**: Offline AAS compiler includes this header; calls `AAS_Error` for build-time diagnostics
- **game/ai_main.c** (indirectly via server): Bot AI decision-making consumes AAS data through public API without seeing internal state

### Outgoing (what this file depends on)

- **be_aas_def.h**: Defines `aas_t` structure (the monolithic `aasworld` state); included by internal AAS modules before this header
- **be_aas_main.c**: Implements all declared functions; defines `aasworld` extern
- **be_aas_bspq3.c, be_aas_sample.c, be_aas_file.c, etc.**: Subsystems called during `AAS_Setup`, `AAS_LoadMap`, `AAS_StartFrame` lifecycle
- **q_shared.h**: Provides `vec3_t`, `QDECL` macro, and other foundational types

## Design Patterns & Rationale

**1. Preprocessor-Gated Visibility (AASINTERN Pattern)**  
The `#ifdef AASINTERN` guard is an early-2000s C idiom predating modern opaque pointers. It enforces compile-time API partitioning: only `.c` files that `#define AASINTERN` before including this header see `aasworld` and lifecycle functions. This prevents:
- Game code from calling `AAS_Setup` out of order
- Accidental `aasworld` reads from external subsystems
- Symbol pollution (no extra vtables or indirection layers needed in 1999)

**2. Singleton Global (`aasworld`) Pattern**  
All AAS subsystems share a single `aas_t` struct. This is space-efficient and matches Q3's era (Quake II heritage), but creates a strict constraint: only one AAS world can exist per process. The public `AAS_Loaded()` / `AAS_Initialized()` predicates are guard functions to safely check state before queries.

**3. Lifecycle Ordering Invariant**  
The header implicitly documents a strict state machine via function sequencing:
```
AAS_Setup() → AAS_LoadMap(mapname) → [ AAS_StartFrame(time) ]* → AAS_Shutdown()
```
This ordering is enforced by convention, not by the API itself—a risk in large codebases.

**4. Variadic Error Reporting**  
`AAS_Error(fmt, ...)` with `QDECL` follows Q3's cross-module calling-convention hygiene. On x86, QDECL ensures correct parameter-passing semantics when botlib is compiled as a separate DLL from the engine.

## Data Flow Through This File

**Initialization Phase:**
```
Server startup
  → trap_BotLibSetup() (server → botlib interface)
  → AAS_Setup() (internal)
     → allocate aasworld, initialize subsystems
  → AAS_SetInitialized() (write flag to aasworld)
  → AAS_LoadMap(mapname)
     → AAS_LoadFiles() / AAS_ParseBSPEntities() (read BSP)
     → AAS_InitClustering(), AAS_InitReachability(), AAS_InitRouting()
     → populate aasworld with graph topology
```

**Per-Frame Phase:**
```
Server frame loop
  → trap_BotLibUpdateEntity() / trap_BotLibUpdateGoal()
     → AAS_EntityInfo(), AAS_PointAreaNum() (read aasworld)
  → trap_BotLibChooseWeapon() → AAS_AreaInfo(), AAS_RouteToGoal()
  → AAS_StartFrame(time)
     → AAS_InvalidateEntities(), AAS_UpdateEntity() (refresh dynamic state)
     → entity cache invalidation; routing cache eviction
```

**Shutdown:**
```
Map unload
  → AAS_Shutdown()
     → Free all aasworld allocations
     → Reset to zeroed state
```

External callers never see `aasworld` directly—all queries go through public functions returning derived data (`AreaNum`, `RoutingCache`, `BestArea`, etc.).

## Learning Notes

**Idiomatic Q3 Architecture:**
- Pre-DLL-era engines (Quake II) used global singletons with no visibility control
- Q3 added DLL support but relied on preprocessor guards rather than opaque pointers
- Modern engines (2010+) would encapsulate this as `struct aasworld_t* AAS_Create()` → `void* aasworld_opaque` in the public API

**Why This Design:**
1. **Memory efficiency**: Single `aasworld` ≈ 2–10 MB per map; OOP vtables add overhead
2. **Fast iteration**: 1999 C idioms prioritized compile-time over runtime checks
3. **Coupling tolerance**: Botlib was originally monolithic; tight coupling to `aasworld` was acceptable

**Modern Parallels:**
- This header mirrors the role of a Rust module's `pub` / `private` visibility
- The `AASINTERN` guard is analogous to `#[cfg(test)]` compile-time gating
- The lifecycle ordering (Setup→Load→Frame→Shutdown) is universal in game engines (see also renderer initialization)

## Potential Issues

1. **AASINTERN Symbol Leakage**  
   If `bspc/` or other tool code includes this header without defining `AASINTERN`, it silently loses access to lifecycle functions. No compile error—just a missing declaration. This is brittle in large codebases.

2. **No Re-entrance Guard**  
   `AAS_Setup()` or `AAS_LoadMap()` called twice would corrupt `aasworld`. The header provides no error if someone calls `AAS_LoadMap()` before `AAS_Setup()`. Only convention enforces ordering.

3. **Global State Coupling**  
   Any AAS subsystem that reads/writes `aasworld` has implicit hard dependency on initialization order. If `AAS_InitReachability()` is called before clustering is complete, crashes may be subtle and hard to trace.

4. **No Thread Safety Annotations**  
   `AAS_StartFrame()` updates `aasworld` each frame; no guards prevent simultaneous reads from another thread. Q3 is single-threaded, but this would be a major issue in modern engines.

5. **Cross-Reference Ambiguity**  
   The cross-reference shows both `be_aas_main.c` and `bspc/be_aas_bspc.c` define `AAS_Error`. This suggests code duplication or a macro—the header alone doesn't clarify which.

---

**Architectural Insight:** This header is a **capsule of 1990s game engine design**. Modern equivalents would use opaque handles, vtable indirection, or language-level visibility (C++ `private`). Its persistent use here reflects both Q3's legacy stability and the cost of refactoring cross-module APIs in shipped codebases.
