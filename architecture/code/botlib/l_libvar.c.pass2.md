# code/botlib/l_libvar.c — Enhanced Analysis

## Architectural Role
This file implements botlib's internal configuration variable system—a minimal, self-contained key-value store used by all AI and pathfinding subsystems to access tunable parameters at runtime. Unlike the engine's cvar system (which requires syscalls), libvars remain purely internal to botlib and are initialized via the library's own script/configuration parsing pipeline. Variables are lazy-created on first access and tracked with a per-frame `modified` flag, allowing subsystems to efficiently detect and react to configuration changes without polling the entire state tree.

## Key Cross-References

### Incoming (who depends on this file)
- **be_aas_main.c** — Uses libvar for AAS subsystem parameters (gravity, step height, entity linking thresholds)
- **be_aas_move.c** — Reads libvars for movement model tuning (acceleration, friction, jump power)
- **be_aas_route.c** — Uses libvar for routing cache size and strategy flags
- **be_ai_*.c** (move, goal, weap, char, chat, weight) — All AI behavior modules read libvars for personality traits, fuzzy scoring weights, and action thresholds
- **be_interface.c** — Calls `LibVarDeAllocAll()` during `BotLibShutdown` to drain all allocated variables
- **l_script.c** / **l_libvar.h** — Script preprocessor/lexer may invoke `LibVar` / `LibVarSet` when parsing `[variable]` declarations or commands in `.c` bot definition files

### Outgoing (what this file depends on)
- **l_memory.h** — `GetMemory()` / `FreeMemory()` for heap allocation (botlib-local allocator, not engine)
- **../game/q_shared.h** — `Q_stricmp()` (case-insensitive string comparison), `Com_Memset()`, `qboolean` / `qtrue` / `qfalse` types
- **C stdlib** — `strcpy()`, `strlen()` (this code compiles outside Q3VM, so has full C lib access unlike game/cgame VMs)

No outgoing calls to other botlib subsystems; purely self-contained utility layer.

## Design Patterns & Rationale

**Lazy Initialization (Get-or-Create)**
- `LibVar(name, value)` creates a variable only on first request with a default value, avoiding upfront declaration overhead
- Rationale: Bot parameters can be numerous and optional; many are never accessed if they remain at defaults, so lazy creation saves memory and initialization cost

**Change Tracking via `modified` Flag**
- Every `LibVar` / `LibVarSet` call sets `modified = qtrue`; caller later polls `LibVarChanged(name)` and clears with `LibVarSetNotModified(name)`
- Rationale: Allows AI subsystems (e.g., weapon selection FSM) to cache expensive decisions until a relevant config parameter actually changes, avoiding per-frame recomputation of weight tables or thresholds

**Dual String + Float Representation**
- Each variable stores both the original string and a precomputed `float` value via `LibVarStringValue()`
- Rationale: Avoids repeated parsing of the same string value; float is more efficient for numeric comparisons in AI logic; string is preserved for debugging/display

**Custom Float Parser (`LibVarStringValue`)**
- Implements a simple decimal parser without using standard `atof()`
- Rationale: Botlib must be compilable standalone (separate from the main engine), so relying on minimal stdlib; the parser also remains deterministic and under explicit control
- **Design tradeoff:** Rejects leading signs and exponents (returns 0), making it unsuitable for general-purpose parsing but safe for this narrow use case (config values are simple positive decimals)

**Single-Allocation Name Storage**
- Variable name is stored immediately after the `libvar_t` struct in one contiguous block, reducing heap fragmentation
- Rationale: Each libvar typically persists for the entire bot library lifetime, so this micro-optimization reduces allocator overhead for many small objects

**Linear Search (O(n))**
- `LibVarGet()` walks the linked list; no hash table
- Rationale: Libvar counts are typically small (tens to hundreds); hashing overhead would not justify itself; list simplifies debugging and teardown
- **Scalability note:** If bot libraries evolved to thousands of variables, a hash table would be beneficial, but Q3A's scope didn't warrant it

## Data Flow Through This File

**Initialization Phase (Startup)**
```
be_interface.c: BotLibSetup()
  ↓
be_aas_main.c: AAS_Setup() / be_ai_*.c: AI initialization
  ↓
Script parser (l_script.c) or direct calls invoke LibVar(name, defaultValue)
  ↓
libvarlist grows as needed; variables allocated from botlib heap
```

**Runtime Phase (Per-Frame)**
```
Server: SV_Frame() → trap_BotLibFrame()
  ↓
botlib AI/pathfinding loops read variables via LibVarGet/LibVarGetString/LibVarGetValue
  ↓
If a libvar was modified (via console or script): modified flag is true
  ↓
Subsystem polls LibVarChanged(name), updates cached state, calls LibVarSetNotModified()
  ↓
Next frame: loop repeats
```

**Shutdown Phase**
```
be_interface.c: BotLibShutdown()
  ↓
LibVarDeAllocAll() iterates libvarlist, frees each variable's string + node
  ↓
botlib heap is flushed
```

## Learning Notes

**Idiomatic to this era / engine:**
- **Simple, ad-hoc config layer:** Modern engines use structured config (JSON, YAML, asset databases); Q3A's botlib uses a raw string key-value store, which is simpler but requires the consuming code to know what variables to expect and what types they are
- **No schema or reflection:** There is no "list all variables" or "get variable type" API; callers must hardcode knowledge of variable names and meaning
- **Polling for changes:** Modern engines often use event callbacks or reactive systems; botlib uses explicit polling of the `modified` flag, a simpler but less elegant pattern
- **Manual memory management:** No RAII or automatic cleanup; `LibVarDeAllocAll()` must be called explicitly

**What developers studying this engine learn:**
1. **Memory discipline in subsystems:** Even utility layers explicitly manage heap (GetMemory/FreeMemory), reflecting Q3A's tight real-time constraints
2. **Separation of concerns:** Botlib's utilities (memory, logging, libvar, script, struct serialization) are stacked in layers, each self-contained; this makes botlib portable and testable standalone
3. **Runtime tunability:** The `modified` flag pattern shows how gameplay parameters can be tweaked between frames without restarting the bot library—a critical feature for live iterative balance tuning
4. **Custom implementations over stdlib:** Custom float parser, custom memory management, custom linked lists—Q3A did not rely on external deps, even in 1999

## Potential Issues

1. **No thread safety:** Global `libvarlist` is not protected by locks. Q3A's server was single-threaded, but if botlib were multi-threaded, concurrent calls to `LibVar` / `LibVarSet` could race. *(Severity: Low in Q3A context; would matter in modern ports.)*

2. **Ambiguous float parse errors:** `LibVarStringValue()` returns `0` on any parse failure (invalid character, leading minus, etc.), making it impossible to distinguish "variable is '0'" from "variable is invalid". *(Severity: Low; config mistakes would be caught during testing.)*

3. **No variable removal:** Only `LibVarDeAllocAll()` exists; no per-variable delete. A long-running server or persistent-memory scenario could accumulate stale variables. *(Severity: Very low; bot library is reloaded per map.)*

4. **String copy without bounds checks:** `strcpy(v->string, value)` relies on caller-provided strings being safe. If a malformed string is passed from script parsing, buffer overflow is possible. *(Severity: Medium; depends on script parser's robustness, which is outside this file's scope.)*

5. **Linear lookup performance:** For `LibVarGet()`, each access is O(n). With hundreds of variables, this could add up in tight per-frame loops. In practice, Q3A libvar counts are small (<100), so this is not a real bottleneck.
