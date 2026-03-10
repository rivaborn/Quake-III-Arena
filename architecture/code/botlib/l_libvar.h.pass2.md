# code/botlib/l_libvar.h — Enhanced Analysis

## Architectural Role

`l_libvar.h` defines botlib's internal configuration variable subsystem—a lightweight, standalone cvar mechanism used exclusively within the bot library to manage runtime settings without coupling to the engine's global cvar system. This isolation is deliberate: botlib is a closed-box AI+navigation library that qcommon's `Cvar_*` functions cannot directly reach (they're only available via syscalls through the game VM). The libvar system allows AAS, pathfinding, and AI decision modules to read/write shared configuration state through a simple linked-list API, decoupled from the engine's cvar lifecycle and validation. This is loaded and dealloc'd during `AAS_LoadMap` / `AAS_Shutdown`, not at engine startup.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/botlib/be_aas_*.c`** (AAS geometry, clustering, routing, reachability) — queries libvar settings for tuning thresholds, debug flags, and feature toggles.
- **`code/botlib/be_ai_*.c`** (move planning, goal selection, weapon choice, chat, weight learning) — polls libvars for weights, personality parameters, and behavioral flags.
- **`code/botlib/be_ea.c`** (elementary action buffer) — may read libvar settings for input constraints.
- **`code/botlib/be_interface.c`** (public botlib export vtable) — initializes/shuts down libvar during `BotLibSetup` / `BotLibShutdown`.

### Outgoing (what this file depends on)
- **`code/qcommon/q_shared.h`** — `qboolean`, `char`, `float` base types; no other engine dependencies visible here.
- **`code/botlib/l_memory.c`** (implicit via `.c` implementation) — heap allocation for `libvar_t` nodes and string copies.
- **No syscalls or engine imports** — botlib config is entirely internal; it does not read or write engine cvars at this layer.

## Design Patterns & Rationale

| Pattern | Mechanism | Why |
|---------|-----------|-----|
| **Linked-list singleton** | Global head pointer (in `.c`); node traversal on lookup | Simple, no hash overhead; botlib has few variables so O(n) is acceptable |
| **Lazy initialization** | `LibVar(name, default)` creates if missing | Variables instantiated on first AI/AAS use; supports hot-reloading mid-session |
| **Dual representation** | Both `string` and cached `float` value | Avoids `atof()` per-frame in inner loops; trade ~8 bytes per var for convenience |
| **Dirty-flag polling** | `modified` flag set by `LibVarSet`, polled by `LibVarChanged` | Per-frame change detection without callbacks; caller decides reaction |
| **No validation** | Flags field exists but no type/range enforcement | Lightweight; validation deferred to caller or post-load passes |

**Rationale for isolation:** The engine's `Cvar_*` system requires going through `qcommon/cvar.c` (which enforces flags like `CVAR_ROM`, `CVAR_LATCH`, `CVAR_CHEAT`). Botlib, running inside the game VM or as an opaque DLL, cannot touch engine cvars directly. Thus a shadow config system inside botlib avoids circular dependencies and allows bot settings to be serialized/validated in botlib's own cache format (`.aas` binary, route cache, etc.).

## Data Flow Through This File

```
┌─────────────────────────────────────────────────────┐
│ Per-Frame AI/Routing Logic                          │
│  (be_ai_*.c, be_aas_*.c)                            │
└────────┬────────────────────────────────────────────┘
         │ LibVarGet(name)
         ▼ LibVarGetValue(name)
    ┌─────────────────────────┐
    │ libvar_t linked list    │ ◄─ LibVarSet(name, value)
    │ (global singly-linked)  │    (also: LibVar/LibVarValue/String)
    └────────┬────────────────┘
             │
             ├─ Returns: string, float, or struct pointer
             │
             ├─ Caches: modified flag per-variable
             │
             └─ Lifecycle: allocated at first use
                           freed in LibVarDeAllocAll()
```

- **Initialization:** `LibVar("bot_debug", "0")` called by AI init → allocates node, copies name + value string, parses float.
- **Query loop:** `LibVarGetValue("bot_aggressiveness")` searches linked list, returns cached `float` (no re-parse).
- **Mutation:** `LibVarSet("bot_debug", "1")` updates string + float, sets `modified = qtrue`.
- **Change detection:** AI subsystem calls `LibVarChanged("bot_debug")`, reacts (e.g., re-cache), calls `LibVarSetNotModified` to clear flag.
- **Shutdown:** `LibVarDeAllocAll()` walked linked list, `free()` all nodes (implementation in `l_libvar.c`).

## Learning Notes

### Patterns & Idioms Specific to Quake 3 Era
1. **No C++ or reflection:** Plain linked-list traversal; no hash tables or dynamic dispatching. This was acceptable in 1999–2005 when bot configs were small (<20 variables).
2. **Polling over callbacks:** No observers/event system. Callers are responsible for checking `LibVarChanged()` each frame—explicit, not magic.
3. **Dual value caching:** Before SIMD/cache-obliviousness was a big deal, caching the float value was a pragmatic micro-optimization to avoid `atof()` in inner loops.
4. **No type system:** Flags field is generic; caller must know what type each variable is. Modern engines use enums or variant types.

### Comparison to Modern Engine Practice
| Q3 (1999) | Modern Engine (2024) |
|-----------|---------------------|
| Linked-list linear search | Hash map or B-tree lookup |
| String + float dual rep | Variant or type-erased value with lazy parse |
| Polling dirty flag | Observer/listener pattern or event queue |
| No nested scopes | Config hierarchies (global, per-subsystem, per-entity) |
| Manual memory mgmt | Reference counting or arena allocators |

### What This File Teaches
- **Simplicity under constraints:** libvar shows how to build a lightweight config system when you can't use the engine's cvar API. No unnecessary abstraction; just enough to unblock AI subsystem needs.
- **Tradeoffs:** Memory (string + float both stored) vs. CPU (avoid per-frame parsing). Linked-list (no allocator fragmentation) vs. lookup speed (OK for small N).
- **Encapsulation:** botlib's public API (`be_interface.c`) exposes only `GetBotLibAPI`, which never returns raw function pointers to `LibVar*`—users go through `BotLib_*` trap syscalls. This header is botlib-internal, not engine-facing.

## Potential Issues

1. **Linear search performance:** With many libvars, `LibVarGet()` becomes O(n). No sorting or hashing. Unlikely to matter in practice (botlib has ~10–15 config variables), but not scalable.
2. **No thread safety:** No mutexes or atomics. If botlib were multi-threaded (it isn't), concurrent `LibVarSet` + `LibVarChanged` could race. Not an issue in Quake 3's single-threaded AI frame.
3. **Unbounded string copying (in `.c` impl):** The header doesn't show bounds, but `l_libvar.c` likely uses fixed-size buffers or unsafe `strcpy`. Modern code would use `strlcpy`.
4. **Modified flag is coarse-grained:** Only per-variable, not per-field. If a variable is updated to the same value, `modified` is still set. Callers must be idempotent on reaction.

---

**File dependencies resolved:** This header is self-contained; only `qboolean` and standard types required. Implementation (`l_libvar.c`) links to `l_memory.c` for allocation and inherits botlib's import API (`botimport.Print`, etc.) for logging.
