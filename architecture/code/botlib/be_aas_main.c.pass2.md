# code/botlib/be_aas_main.c — Enhanced Analysis

## Architectural Role
This file is the **lifecycle and coordination facade** for the entire AAS (Area Awareness System) subsystem within botlib. It sits between the engine (via `be_interface.c`) and all AAS subsystems (reachability, routing, clustering, entity tracking), orchestrating map load, deferred multi-frame initialization, and per-frame entity/routing updates. Every bot navigation query ultimately depends on state managed by this file's global `aasworld` singleton.

## Key Cross-References

### Incoming (who depends on this file)
- **`be_interface.c`**: The botlib's sole engine-facing boundary. Calls `AAS_Setup`, `AAS_LoadMap`, `AAS_StartFrame`, `AAS_Shutdown` during botlib lifecycle and per-frame server ticks.
- **`code/server/sv_bot.c`**: Server-side bot management; indirectly relies on AAS state for navigation queries funneled through botlib API.
- **All AAS subsystems** (`be_aas_route.c`, `be_aas_reach.c`, `be_aas_cluster.c`, etc.): Read/write global `aasworld` state and call back into this file's initialization orchestration (e.g., `AAS_ContinueInit` calls `AAS_ContinueInitReachability`, `AAS_InitClustering`, `AAS_InitRouting`).

### Outgoing (what this file depends on)
- **AAS subsystem bootstrap**: `AAS_LoadBSPFile`, `AAS_LoadAASFile`, `AAS_DumpAASData`, `AAS_DumpBSPFile`, `AAS_FreeRoutingCaches`, `AAS_FreeAASLinkHeap`, etc. (defined in `be_aas_*.c`)
- **Reachability pipeline**: `AAS_ContinueInitReachability`, `AAS_InitReachability`, `AAS_Optimize`, `AAS_WriteAASFile` (deferred init via `AAS_ContinueInit`)
- **Routing & clustering**: `AAS_InitClustering`, `AAS_InitRouting`, `AAS_WriteRouteCache`, `AAS_RoutingInfo` (diagnostics)
- **Entity tracking**: `AAS_UnlinkInvalidEntities`, `AAS_InvalidateEntities`, `AAS_ResetEntityLinks`, `AAS_InitAASLinkedEntities`
- **Library utilities**: `LibVar`, `LibVarGetValue`, `LibVarSet` (config string introspection); `GetMemory`, `GetClearedHunkMemory`, `FreeMemory` (memory management); `botimport` callbacks (error reporting, print)
- **Math**: `VectorSubtract`, `VectorNormalize`, `VectorMA`, `DotProduct` (vector utilities used in `AAS_ProjectPointOntoVector`)

## Design Patterns & Rationale

1. **Deferred Multi-Frame Initialization**: `AAS_ContinueInit` spreads reachability computation (expensive, blocking) across game frames. Early 2000s engines avoided stalls by deferring cold-path work—a precursor to modern async initialization patterns. Server checks `aasworld.initialized` before running bot AI each frame.

2. **Singleton State Registry**: Global `aasworld` avoids passing state through deep call stacks. File-global `saveroutingcache` libvar pointer is read on every frame—a light-weight pattern for config-driven behavior without per-call parameter threading.

3. **Accessors & Guards**: `AAS_StringFromIndex` / `AAS_IndexFromString` enforce bounds checking and NULL validation; `aasworld.indexessetup` acts as a "ready" flag. Model name lookup goes through these accessors, preventing direct access to raw config string arrays.

4. **Lifecycle Flags**: Separate `loaded` and `initialized` flags disambiguate states: loaded→ready-to-init, initialized→all systems go. Explicit `AAS_SetInitialized` allows downstream systems to signal completion.

5. **Per-Frame Vs. Deferred Work Separation**: `AAS_StartFrame` is kept hot (entity linking, invalidation, lib-var diagnostics). Heavy work (reachability, routing, file I/O) lives in `AAS_ContinueInit`, called conditionally until done.

## Data Flow Through This File

```
ENGINE INIT:
  AAS_Setup
    ↓ allocates entity pool, reads maxclients/maxentities libvars
  
MAP LOAD (via be_interface → AAS_LoadMap):
  AAS_LoadMap(mapname)
    ↓ frees old routing caches
  AAS_LoadFiles(mapname)
    ├─ AAS_ResetEntityLinks (clear links to areas/clusters)
    ├─ AAS_LoadBSPFile
    └─ AAS_LoadAASFile → sets aasworld.loaded = true
    ↓
  AAS_InitSettings / AAS_InitAASLinkHeap / AAS_InitAASLinkedEntities
    ↓ prepare subsystems
  AAS_InitReachability / AAS_InitAlternativeRouting
    ↓ register reachability heap, portal/alternative routing (incremental)

PER-FRAME (AAS_StartFrame):
  aasworld.time ← time
  AAS_UnlinkInvalidEntities() ← stale entities removed from AAS links
  AAS_InvalidateEntities() ← reset update flags
  AAS_ContinueInit(time)  ← if not initialized:
    ├─ AAS_ContinueInitReachability (incremental, returns when done)
    ├─ AAS_InitClustering
    ├─ [optional] AAS_Optimize + AAS_WriteAASFile
    └─ AAS_InitRouting → sets aasworld.initialized = true
  [if saveroutingcache libvar] AAS_WriteRouteCache()
  aasworld.numframes++
  
ENGINE SHUTDOWN:
  AAS_Shutdown
    ↓ frees all subsystem memory, zeroes aasworld
```

## Learning Notes

1. **Deferred Initialization at Scale**: Reachability calculation (finding all reachable areas from each area via ~14 travel types) is expensive. Rather than block the server on map load, `AAS_ContinueInitReachability` is called incrementally across frames. This pattern predates modern async/await but achieves similar goals—responsive server, bounded per-frame cost.

2. **Singleton vs. Modularity Tradeoff**: Every AAS function accesses global `aasworld` rather than taking it as a parameter. This simplified the 2005 codebase but creates hard dependencies; modern engines would use context objects or ECS-style component systems.

3. **Libvar Configuration at Runtime**: `LibVar("saveroutingcache", "0")` returns a pointer that's checked every frame. This avoids expensive string lookups; the libvar system is a lightweight "config that can be toggled without restart."

4. **String Index Registry Pattern**: `AAS_UpdateStringIndexes` copies config strings from the engine into local pointers. Early binding (copy-on-update rather than lazy lookup) trades memory for speed and avoids engine callback overhead in hot paths.

5. **Contrast with Modern Approaches**:
   - **Immediate vs. Deferred Initialization**: Modern engines use background threads or streaming loaders; this game defers within a single thread across frames.
   - **Global Singletons vs. ECS**: AAS_* functions all implicitly read/write `aasworld`; modern engines pass state objects or use component queries.
   - **Per-Frame Ticking**: `AAS_StartFrame` is explicitly called once per server frame—a "tick" pattern that contrasts with event-driven or continuous simulation.

## Potential Issues

1. **Memory Leak in `AAS_UpdateStringIndexes`**: Commented-out `FreeMemory(aasworld.configstrings[i])` line (L143). On config string updates (which may happen mid-game), old allocations are not freed, causing gradual memory bloat if strings are updated frequently. Low risk if config strings are stable post-load, but fragile.

2. **No Concurrency Safeguards**: Global `aasworld` and `saveroutingcache` are accessed without locks. Single-threaded engine assumption; would break if server and bot AI ran on separate threads.

3. **`AAS_ProjectPointOntoVector` Unbounded**: Projects point onto infinite line, not line *segment*. Result may lie far outside `[vStart, vEnd]`. Caller must clamp if a bounded segment projection is needed. Not necessarily a bug, but easy to misuse.

4. **Error Recovery**: `AAS_LoadFiles` and `AAS_ContinueInit` set flags but don't roll back cleanly on partial failure. If `AAS_WriteAASFile` fails mid-init, the world is half-initialized. Engine must manually call `AAS_Shutdown` to recover.
