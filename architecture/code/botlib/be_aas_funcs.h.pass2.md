# code/botlib/be_aas_funcs.h — Enhanced Analysis

## Architectural Role
This header serves as the **primary runtime aggregation point** for the complete AAS (Area Awareness System) module APIs within botlib. It is included by all runtime botlib translation units that need the full AAS toolkit, but is explicitly guarded against the offline BSPC map compiler tool to maintain a clean separation between runtime and compilation-time AAS implementations. By bundling all AAS sub-module headers in one place, it enables the botlib interface layer (`be_interface.c`) to cleanly export a versioned `botlib_export_t` function-pointer vtable to the engine without exposing all internal module details.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/botlib/be_interface.c`** — botlib's sole public entry point; includes this header to access the full AAS API for synthesizing the `botlib_export_t` vtable exported to the server
- **Internal botlib translation units** (e.g., `be_ai_*.c`, `l_*.c`) — may include this for intra-botlib AAS queries during AI decision-making and movement simulation
- **`code/bspc/be_aas_bspc.c`** — *not* affected by this header due to `#ifndef BSPCINCLUDE` guard; BSPC uses its own AAS stub and offline implementation

### Outgoing (what this file depends on)
- **12 bundled AAS sub-module headers** — each encapsulates a distinct AAS responsibility (entity tracking, sampling, clustering, reachability, routing, etc.)
- **`BSPCINCLUDE` macro** — defined externally by BSPC's build system; its presence suppresses all includes to prevent linker conflicts and unnecessary runtime dependencies in the offline tool
- **`AASINTERN` macro** — gates internal-only function declarations (e.g., `AAS_LoadBSPFile`, `AAS_FreeRoutingCaches`) in included headers, visible only to botlib-internal callers and not to external engine code

## Design Patterns & Rationale

**Facade Pattern via Selective Aggregation**
- This header does not re-export or wrap; it simply aggregates 12 sister headers into a single include.
- Clients never include individual AAS sub-headers directly; they always go through this facade.
- This design decouples the public `botlib_export_t` vtable from internal AAS module structure. If a new AAS sub-module is added, only `be_interface.c` needs to be aware of it; external callers remain unchanged.

**Build-Time Polymorphism via Preprocessor Guards**
- The `#ifndef BSPCINCLUDE` guard achieves two build-time behaviors with one source tree: runtime botlib and offline BSPC.
- BSPC defines `BSPCINCLUDE` at compile time to disable all runtime AAS includes, allowing it to substitute its own stub implementations and avoid symbol conflicts.
- This avoids code duplication: both BSPC and botlib reuse the same AAS cluster, reachability, and optimization algorithms (defined in `code/bspc/` and duplicated in `code/botlib/`).

**Internal vs. External API Boundary**
- The `AASINTERN` macro (used within bundled headers) gates declarations meant only for botlib-internal translation units. External code (the server/game via `trap_BotLib*` syscalls) only sees the `botlib_export_t` vtable, not these internal APIs.
- Example: `AAS_LoadBSPFile` is marked `AASINTERN`, so the engine cannot call it directly; only `be_interface.c` (botlib's init code) uses it.

## Data Flow Through This File

**Initialization Phase** (server startup):
1. Server calls `GetBotLibAPI()` (defined in `be_interface.c`) to obtain the `botlib_export_t` vtable.
2. `be_interface.c` includes this header and calls internal AAS functions (e.g., `AAS_LoadMap`, `AAS_InitClustering`, `AAS_InitRouting`) to populate the `aasworld` global singleton.
3. All 12 AAS sub-modules are now initialized and ready.

**Per-Frame Bot AI Phase** (each server frame):
1. Server calls `BotLibUpdateArea()` and `BotLibAI()` (from `botlib_export_t`).
2. These vtable entries dispatch to internal implementations that use the aggregated AAS APIs: e.g., `AAS_PointAreaNum()` (from `be_aas_sample.h`), `AAS_AreaRouteToGoalArea()` (from `be_aas_route.h`).
3. AAS query results flow back to the AI decision-making layer (`be_ai_*.c`).

**Routing Cache Management** (on-demand during pathfinding):
1. AI code calls `AAS_AreaRouteToGoalArea()`, which lazily builds and caches routing data.
2. Routing caches (from `be_aas_route.h`) are evicted when memory pressure is detected.
3. When the server shuts down, `AAS_FreeRoutingCaches()` and `AAS_Shutdown()` are called to clean up.

## Learning Notes

**What a Developer Studies Here:**
- How a complex subsystem (AAS with 12 internal modules) is cleanly wrapped and exposed through a single facade header.
- The power of preprocessor guards to enable two different build configurations (runtime vs. offline) from one source tree without code duplication.
- The idiom of **file-static initialization** (all AAS state in global `aasworld` singleton, initialized on-demand by `be_interface.c`, never touched by external code).

**Idiomatic Patterns of This Engine/Era:**
- **No namespace/module system**: In 1999-2005, C had no language-level namespacing, so header aggregation + `AASINTERN` macros were the primary tools for API privacy.
- **Monolithic globals**: All AAS state lives in a single `aasworld` struct; there is no per-map or per-instance isolation (the engine is single-threaded with one active server at a time).
- **Dual-purpose source trees**: BSPC and botlib share algorithm implementations but have separate build outputs; the `BSPCINCLUDE` guard keeps them decoupled at link time.

**How Modern Engines Differ:**
- Modern engines use **header-only or namespace-scoped APIs** (C++ namespaces, Rust modules) rather than preprocessor guards.
- **Static analysis and visibility enforcement** replace runtime vtable dispatch for internal subsystems.
- **Pluggable routing/pathfinding** (like Recast/Detour in modern games) is separate from the core engine; Quake III tightly couples AAS to botlib.

## Potential Issues

None clearly inferable from this file alone. However, cross-referencing with the subsystem architecture:

- The **tight coupling of AAS to botlib** means that if the engine ever needs navigation features outside of bot AI (e.g., for NPC movement, dynamic obstacle avoidance), the AAS module cannot be easily reused or evolved independently.
- The **global `aasworld` singleton** pattern (implied by the header aggregation) means no per-instance isolation or thread-safety for a hypothetical multi-map or multithreaded server.
- The **`BSPCINCLUDE` guard** requires careful coordination between build systems; if a developer adds a new header to this aggregation without updating the BSPC build rules, linker errors may occur in offline tool builds.
