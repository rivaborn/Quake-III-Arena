# code/bspc/be_aas_bspc.c — Enhanced Analysis

## Architectural Role

This file is a **context adapter** that transforms the BSPC offline tool into a viable consumer of the botlib AAS (Area Awareness System) library. Rather than embedding botlib logic directly, it implements the `botlib_import_t` dependency-injection vtable, allowing botlib to remain context-agnostic while `be_aas_bspc.c` bridges to the tool's collision model and logging infrastructure. It is the critical glue that enables `AAS_CalcReachAndClusters`—the offline counterpart to the engine's real-time reachability/clustering pipeline—to execute the full AAS build within BSPC.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — Main BSPC tool entry point calls `AAS_CalcReachAndClusters` to compute AAS data for a given map
- **`code/botlib/be_aas_*.c`** — All AAS subsystems (reach, cluster, route, entity, sample, etc.) indirectly depend on the `botimport` vtable populated here; they call functions like `botimport.Trace`, `botimport.PointContents`, `botimport.BSPEntityData` for collision queries, entity data, and logging

### Outgoing (what this file depends on)
- **`code/qcommon/cm_public.h`** — Collision model (`CM_LoadMap`, `CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, `CM_ModelBounds`, `CM_EntityString`)
- **`code/botlib/be_aas_def.h`** — Exposes global `aasworld` singleton where results are stored
- **`code/botlib/be_aas_*.h`** — All AAS subsystem entry points (`AAS_LoadBSPFile`, `AAS_InitSettings`, `AAS_InitReachability`, `AAS_InitClustering`, `AAS_ContinueInitReachability`)
- **`code/bspc/l_log.h`** — Logging (`Log_Print`, `Log_Write`)
- **Extern `Error`** — Fatal error handler from BSPC utility code; invoked by `AAS_Error`

## Design Patterns & Rationale

**Dependency Injection via Function-Pointer Vtable:**  
The core pattern is the `botlib_import_t` function-pointer struct, allowing botlib to be agnostic to its execution environment (runtime engine vs. offline tool). This decoupling is essential because:
- Botlib's AAS and pathfinding algorithms are stateless/algorithmic; they delegate I/O, memory, and physics to injected callbacks
- Runtime (server) and offline (BSPC) contexts have different collision backends, memory models, and logging
- The same botlib binary/source can be reused in both contexts

**Stub & Wrapper Pattern:**  
Several functions are no-ops (`AAS_DebugLine`, `AAS_ClearShownDebugLines`) or minimal wrappers (`Sys_MilliSeconds` via `clock()`), reflecting that the tool needs only the minimum viable subset of services. `Com_Memset`, `Com_Memcpy`, and `COM_Compress` are simple redirects to standard C library functions, avoiding duplication of botlib's dependencies.

**Sequential Pipeline Invocation:**  
`AAS_CalcReachAndClusters` orchestrates a strictly linear initialization sequence (map load → AAS load → settings init → reachability → clustering), which contrasts with the runtime engine's incremental per-frame updates. The dummy `time` parameter in the reachability loop (`AAS_ContinueInitReachability`) highlights the offline context: time is not real elapsed time but a completion counter.

## Data Flow Through This File

1. **Ingress:** `AAS_CalcReachAndClusters` receives a `quakefile_t` descriptor pointing to a Quake map
2. **Map & Collision Load:** `CM_LoadMap` parses the BSP file, populating the collision model; `CM_InlineModel(0)` retrieves the world geometry handle
3. **Import Vtable Setup:** `AAS_InitBotImport` populates the global `botimport` struct, making all tool-context services available to botlib
4. **BSP Entity Parsing:** `AAS_LoadBSPFile` retrieves entity strings via `BotImport_BSPEntityData` (which calls `CM_EntityString`)
5. **Reachability Computation:** `AAS_InitReachability` + the loop calling `AAS_ContinueInitReachability` computes all inter-area travel links; during this phase, botlib calls `BotImport_Trace`, `BotImport_PointContents`, `BotImport_BSPModelMinsMaxsOrigin` to validate jump arcs and reachability
6. **Cluster Computation:** `AAS_InitClustering` partitions areas into visibility-connected clusters
7. **Output:** All results written into the global `aasworld` singleton, which can then be serialized to disk by BSPC

## Learning Notes

**Why Offline AAS Compilation?**  
Q3A pre-computes AAS data (area connectivity, reachability types, routing indices) because real-time computation would be prohibitively expensive. The BSPC tool runs this compilation once per map during development; the runtime engine loads the `.aas` binary and uses fast lookup tables.

**Idiomatic Offline Tool Pattern:**  
This file exemplifies a common game-engine pattern: separating tool context from runtime context. Modern engines (Unreal, Unity) often use similar vtable patterns to allow tools (editors, compilers, importers) to use engine libraries offline. The key is identifying the minimal I/O boundary (`botlib_import_t` here) and providing context-specific implementations.

**Virtual Filesystem & Collision Integration:**  
The tool leverages the same `CM_*` collision API used at runtime, ensuring reachability computations are consistent with in-game physics. This unity is critical for bot reliability.

## Potential Issues

1. **Bug in `BotImport_BSPModelMinsMaxsOrigin`** (lines 183–192):  
   When expanding for rotation, the code recomputes `maxs[i]` using the already-mutated `mins[i]`:
   ```c
   mins[i] = (mins[i] + maxs[i]) * 0.5 - max;
   maxs[i] = (mins[i] + maxs[i]) * 0.5 + max;  // ← mins[i] has changed!
   ```
   Should cache the center before mutation. Impact: Incorrect bounding boxes for rotated BSP models (e.g., func_rotating entities), potentially missing reachability links.

2. **`Sys_MilliSeconds` via `clock()`** (line 72):  
   Uses `clock()` (CPU time, low resolution) instead of wall-clock time. Suitable only for progress reporting; would mislead any reachability code expecting real elapsed time.

3. **Ignored `passent` Parameter** (line 101):  
   `BotImport_Trace` ignores the `passent` parameter (entity to skip), unlike the runtime engine's trace which honors per-entity collision filters. If reachability computation ever needed to skip specific entities, this would be a silent failure.

4. **Hardcoded `exp_dist` and `sidenum`** (lines 113–114):  
   These are always 0, which may be acceptable for AAS compilation but indicates incomplete trace result mapping if `sidenum` (BSP face index) was ever needed downstream.
