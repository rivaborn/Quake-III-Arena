# code/botlib/be_aas_optimize.h — Enhanced Analysis

## Architectural Role
`AAS_Optimize` is the compaction phase in a three-stage AAS initialization pipeline: **load** → **optimize** → **route**. It post-processes the binary world data loaded from disk, stripping unused geometry (non-ladders), de-fragmenting arrays, and merging redundant edges/faces to minimize the runtime footprint before routing tables are computed. This housekeeping phase is essential to fitting large AAS worlds into botlib's fixed memory budget and preparing the structure for efficient pathfinding lookups.

## Key Cross-References

### Incoming (who depends on this)
- **`AAS_LoadFiles`** (`code/botlib/be_aas_main.c`) — Calls `AAS_Optimize` immediately after `AAS_LoadAASFile` returns, as part of the top-level AAS initialization sequence
- **`AAS_Setup`** (in `be_aas_main.c`) — Indirect caller via the load phase, as part of engine startup
- Indirectly: **`GetBotLibAPI`** → engine → server calls into botlib to initialize AAS once per map

### Outgoing (what this depends on)
- **Global `aasworld` singleton** (defined in `be_aas_def.h`/`be_aas_main.c`) — Reads and writes all AAS geometry, topology, and reachability arrays directly
- **Internal helpers in `be_aas_optimize.c`**: `AAS_OptimizeAlloc`, `AAS_OptimizeStore`, `AAS_OptimizeArea`, `AAS_OptimizeFace`, `AAS_OptimizeEdge`, `AAS_KeepFace`, `AAS_KeepEdge` — All operate on the same global state
- **No dynamic subsystem calls** — Pure data structure compaction; does not invoke renderer, filesystem, or collision queries

## Design Patterns & Rationale

**Three-Phase Initialization Model**: Botlib uses a clear separation of concerns:
- **Load phase** (`AAS_LoadAASFile`): Deserialize binary blob, byte-swap for endianness, populate `aasworld`
- **Optimize phase** (`AAS_Optimize`): Compact redundancy, strip unused geometry, prepare for routing
- **Routing phase** (`AAS_InitRouting`): Precompute all reachability paths, cluster hierarchy, portal caches

This separation allows the offline compiler (bspc) and runtime engine to share the optimize step, ensuring consistency between precomputed and runtime data.

**Global State Coupling**: Unlike the game/cgame VMs, which are sandboxed and re-entrant, botlib embraces a **monolithic global state** pattern (`aasworld`). This is typical for non-VM subsystems in Q3A and reflects the era's memory and threading constraints. `AAS_Optimize` operates directly on that state without parameters or return values — pure side effects.

**Memory Compaction Strategy**: The optimize pass filters arrays in-place, eliminating "dead" elements (faces/edges marked for pruning) and reallocating to recover fragmentation. This is a one-shot operation during init, never invoked per-frame, so the cost is amortized.

## Data Flow Through This File

**Input**:
- Fully loaded `aasworld` with all vertices, edges, faces, areas, reachabilities from the binary file
- May include unnecessary geometry (e.g., non-ladder faces that contribute no pathfinding value)

**Transformation**:
1. Mark faces/edges for retention based on reachability requirements (`AAS_KeepFace`, `AAS_KeepEdge`)
2. Compact vertices, edges, faces arrays by eliminating unmarked entries
3. Update all cross-references and indices to maintain consistency
4. Rebuild area topology with compacted face/edge lists

**Output**:
- Optimized `aasworld` ready for routing table computation
- Reduced memory footprint (typically 10–40% savings on large maps)
- All pointers/indices remain valid; structure is semantically identical to input

## Learning Notes

- **Demonstrates staged initialization**: Modern engines often adopt similar phases (load asset, optimize representation, build acceleration structures). Q3A does this explicitly in botlib, less so in the main engine.
- **Global state trade-off**: Botlib's monolithic `aasworld` avoids allocation overhead and maintains simplicity but precludes multi-instance AAS or dynamic reloading—a deliberate choice for late-90s constraints.
- **Offline/runtime code reuse**: Both bspc (offline compiler) and the engine can call `AAS_Optimize` by linking the same `.c` file, a form of compile-time polymorphism. Modern engines would use separate offline and runtime pipelines.
- **No async support**: Optimization is synchronous and blocking; the engine stalls briefly during map load. Acceptable for single-threaded init but would be a bottleneck in modern streaming architectures.

## Potential Issues

- **No idempotency check**: If `AAS_Optimize` is called twice (e.g., due to a bug in the load sequence), it may corrupt already-optimized data or double-free. No guard exists.
- **Silent data loss**: Faces/edges marked as non-ladder are silently discarded. If the reachability logic is incorrect, bots lose valid pathways with no diagnostic output.
- **Tightly coupled global state**: Cannot test or optimize a single map's AAS independently without setting up the full `aasworld` global; no dependency injection or handles.
