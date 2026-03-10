# code/botlib/be_aas_file.h — Enhanced Analysis

## Architectural Role

This header defines the persistence layer for botlib's core singleton: the compiled AAS (Area Awareness System) world. It bridges offline compilation (BSPC) and runtime (game engine) via a shared binary file format—the `.aas` file—which encodes the complete precomputed navigation mesh (areas, clusters, reachability edges, travel costs). The `AASINTERN` guard enforces strict internal visibility: only botlib and BSPC are permitted to load/write/inspect AAS data; all external consumers (game VM, server, clients) query AAS through public APIs in other headers.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/botlib/be_aas_main.c`** — `AAS_LoadFiles` / `AAS_LoadMap` call `AAS_LoadAASFile` during botlib initialization when a new map loads
- **`code/botlib/be_aas_route.c`** — routing cache queries implicitly depend on a successfully loaded AAS world
- **`code/bspc/aas_file.c`** — the offline BSPC compiler tool has an identical/parallel implementation; calls `AAS_WriteAASFile` to serialize compiled AAS data
- **`code/server/sv_bot.c`** — indirectly depends: server calls bot initialization, which triggers map load and AAS load

### Outgoing (what this file depends on)
- **`code/botlib/be_aas_def.h`** — defines `aasworld_t` singleton and all AAS data structure types (areas, clusters, edges, faces, vertices, etc.)
- **`code/botlib/be_aas_main.c`** — `AAS_Error` for error reporting; `AAS_Setup` / `AAS_Shutdown` for lifecycle
- **`code/botlib/l_memory.c`** — memory allocation/deallocation for AAS data (via import vtable)
- **`botlib_import_t botimport`** — file system (`FS_ReadFile`, `FS_WriteFile`) and memory primitives supplied by the server at runtime

## Design Patterns & Rationale

**1. Singleton Persistence**  
The global `aasworld` is the only navigation database. Rather than dynamically query or recompute pathfinding on-demand, Q3A precomputes the *entire* graph offline and persists it as a binary blob. This trades storage for O(1) area queries and O(N) shortest-path lookups—critical for 64–256 concurrent bots.

**2. Tool/Runtime Code Reuse**  
Both BSPC and the game engine link the same botlib source (`be_aas_file.c`). This eliminates format divergence: the tool writes exactly what the engine reads. The parallel implementation in `code/bspc/aas_file.c` uses the same logic but with a stubbed `botlib_import_t` (pointing to BSPC's own file and memory APIs).

**3. Access Control via Preprocessor Guard**  
`AASINTERN` is defined only within botlib/BSPC translation units. External consumers (cgame, game, server) see only the public AAS API (in headers like `be_aas_main.h`, `be_aas_route.h`, etc.), which internally call these file functions but never expose them.

**4. Binary Format Over Text**  
AAS data is persisted as a binary dump (not JSON/XML/text). This minimizes load time and disk footprint—critical for 50+ MB BSP maps with millions of edges/faces.

## Data Flow Through This File

**Load Path (Game Startup):**
1. Server calls `AAS_LoadMap(mapname)` from `be_aas_main.c`
2. `AAS_LoadMap` → `AAS_LoadFiles` → `AAS_LoadAASFile("maps/q3dm1.aas")`
3. `AAS_LoadAASFile` reads the binary file via `botimport.FS_ReadFile`
4. Deserializes/byte-swaps lump structures (areas, clusters, edges, etc.) into the global `aasworld` struct
5. Returns success/failure code
6. If successful, AAS is now ready for pathfinding queries from bots

**Unload Path (Map Change):**
1. Server calls `AAS_Shutdown` from `be_aas_main.c`
2. `AAS_Shutdown` → `AAS_DumpAASData`
3. Frees all allocated AAS data, zeroing the `aasworld` singleton
4. Next map load repeats the cycle

**Write Path (BSPC Tool Only):**
1. BSPC compiles BSP → AAS graph offline
2. Calls `AAS_WriteAASFile("maps/q3dm1.aas")` to serialize the compiled result
3. File is shipped with the map; game engine later loads it via path above

## Learning Notes

**What a Developer Studies Here:**
- **Offline/Online Duality:** How precomputation (BSPC tool) feeds runtime (game). This is fundamental to Q3A's bot architecture: bots don't compute pathfinding—they traverse precomputed edges.
- **Binary Format Design:** The `.aas` file is a lump-based format (areas lump, edges lump, etc.), similar to BSP. Developers learn how to serialize complex graphs for fast deserialization.
- **Module Boundaries:** The `AASINTERN` pattern is Q3A's way of enforcing "internal APIs." Modern engines use `private` or namespace visibility; Q3A uses preprocessor guards.
- **Idiomatic Q3A:** This file exemplifies late-1990s game engine practices: global singletons, binary persistence, tight offline/runtime coupling, no garbage collection (manual malloc/free).

**Modern Contrast:**
- Modern engines (Unreal, Godot) use runtime navmesh generation or serialize to platform-agnostic formats (JSON, YAML, protobuf).
- Q3A's approach assumes 100% of navigation can be precomputed offline—true for static maps but limits dynamic obstacles.
- No async file I/O here; blocking reads are synchronous.

## Potential Issues

1. **Missing Format Versioning:**  
   No mention of format version numbers. If the `.aas` file format ever changes, old `.aas` files will silently misinterpret. A version header would catch this.

2. **No Thread Safety Notes:**  
   If `AAS_LoadAASFile` is called while bots are simultaneously querying the old `aasworld`, a race condition occurs. The code appears single-threaded (no locks visible), so the calling code must ensure mutual exclusion—not documented here.

3. **Error Recovery Unclear:**  
   Return type `int` is vague; typical Q3A convention is `1 = success, 0 = failure`, but nowhere documented. A failure to load `.aas` likely leaves `aasworld` in an inconsistent state.

4. **No Endianness Documentation:**  
   Binary format requires byte-swapping on big-endian platforms. `AAS_SwapAASData` (referenced in cross-index) must be called, but this header doesn't mention it—a caller might forget.
