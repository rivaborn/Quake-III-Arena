# code/ui/ui_atoms.c — Enhanced Analysis

## Architectural Role

This file serves as the **foundational UI-to-engine bridge** in the MissionPack UI subsystem. It handles three critical responsibilities: (1) **coordinate transformation** from the canonical 640×480 virtual space to physical screen pixels, enabling resolution-independent menu rendering; (2) **command dispatch** from the engine's console to UI handlers (postgame scoring, asset caching, shader remapping); and (3) **post-game score persistence** with per-map/per-gametype `.game` file storage, comparing current results against stored bests and populating cvars for display. All rendering primitives and score operations funnel through this module, making it essential plumbing between the isolated UI VM and the core engine.

## Key Cross-References

### Incoming (who depends on this file)
- **`UI_ConsoleCommand`**: Called by `CL_UISystemCalls` in `code/client/cl_ui.c` for all UI command dispatch. The client engine invokes this once per frame with the current real-time value.
- **Drawing functions** (`UI_FillRect`, `UI_DrawRect`, `UI_DrawHandlePic`, `UI_DrawNamedPic`): Called by `code/ui/ui_shared.c` and throughout the UI layer to render primitives, rectangles, and textured quads in 640×480 space.
- **`UI_AdjustFrom640`**: Used internally by all drawing functions above; also likely called by other UI code for coordinate scaling.
- **Score functions** (`UI_SetBestScores`, `UI_LoadBestScores`, `UI_ClearScores`): Called from `code/ui/ui_main.c` when entering map select screens or post-game flows.
- **`Com_Error` / `Com_Printf`** (conditionally): Used by `code/game/q_shared.c` when the UI is hard-linked (non-VM build); provides a bridging shim.

### Outgoing (what this file depends on)
- **Trap syscalls** (`trap_Cmd_*`, `trap_Cvar_*`, `trap_FS_*`, `trap_R_*`, `trap_Argv`): All defined in `code/ui/ui_syscalls.c` as indexed VM-to-engine thunks. These are the sole mechanism for UI sandbox communication.
- **`uiInfo` global** (defined in `code/ui/ui_main.c`): Master UI state structure holding map lists, server info, score data, display context (scale factors), shader handles, and timing.
- **`Display_CacheAll`, `UI_ShowPostGame`, `UI_Report`, `UI_Load`** (from `code/ui/ui_main.c`): High-level UI logic called by command handlers.
- **`Com_sprintf`, `Q_strncpyz`, `Q_stricmp`, `Info_ValueForKey`** (from `code/game/q_shared.c`): Shared utility functions for string handling and parsing.

## Design Patterns & Rationale

### Virtual Coordinate System Abstraction
The 640×480 virtual space acts as a **resolution-independent layout layer**. All UI elements are authored in this fixed coordinate space; `UI_AdjustFrom640` scales them to the physical screen at draw time using per-axis scale factors (`uiInfo.uiDC.xscale` / `yscale`). This pattern is idiomatic to early-2000s game UIs (Quake III, Half-Life 2, Unreal 2004) and allows menu layouts to work on any monitor resolution without redesign. The commented-out scale+bias code (lines 395–398) suggests a prior version tried a different scaling strategy; the current dual-axis scaling is simpler.

### Persistent Score Serialization
The `UI_CalcPostGameStats` → `UI_SetBestScores` flow demonstrates **direct struct-to-disk persistence** without schema versioning. A `postGameInfo_t` is written as raw binary with only a size header. This is fast and compact but brittle: any struct layout change breaks older `.game` files. The design prioritizes simplicity over robustness, acceptable for a single-player campaign.

### Cvar-as-Display-Cache Pattern
`UI_SetBestScores` pushes all score fields into `ui_score*` cvars (15+ calls to `trap_Cvar_Set`). These cvars serve dual purposes: **(a)** they persist state across menu navigation, and **(b)** they act as a display-cache for the renderer to bind to UI element strings without additional lookups. This is efficient but conflates display state with game logic state.

### Command Dispatch via `Q_stricmp` Chain
`UI_ConsoleCommand` uses a sequential if-chain of string comparisons to dispatch 8 command types. Modern engines would use a hash table or function-pointer dispatch table, but this approach is straightforward for a small command set and reflects early-2000s Quake architecture.

## Data Flow Through This File

**Incoming Command Flow:**
1. Engine calls `UI_ConsoleCommand(realTime)` each frame.
2. `UI_Argv(0)` fetches the command name; comparisons route to handlers.
3. Example: `postgame` command → `UI_CalcPostGameStats()` reads arguments 3–14 via `UI_Argv()` (match stats) → reads stored best from disk → computes time/skill bonuses → writes new best if higher → calls `UI_SetBestScores()` to populate cvars → calls `UI_ShowPostGame()` to display results.

**Drawing Flow:**
1. Menu rendering code calls `UI_FillRect()` or `UI_DrawHandlePic()` with 640×480 coordinates.
2. Function calls `UI_AdjustFrom640()` to scale in-place, then calls `trap_R_DrawStretchPic()` with physical pixel coordinates.
3. Renderer receives the command and rasterizes the quad in the backbuffer.

**Score Persistence Flow:**
- **Load**: `UI_LoadBestScores(map, gametype)` → read `games/<map>_<gametype>.game` → parse `postGameInfo_t` → call `UI_SetBestScores(..., qfalse)` to populate UI display cvars.
- **Store**: `UI_CalcPostGameStats()` (triggered by server postgame command) → compare new score against loaded best → if higher, write back to same file with updated struct.
- **Clear**: `UI_ClearScores()` enumerates all `.game` files, writes zero-initialized structs (clearing all entries).

## Learning Notes

**Idiomatic Early-2000s Quake Architecture:**
- **Trap syscalls**: The VM→engine boundary is entirely syscall-indexed. UI code never directly calls engine functions; all dependencies are mediated through `trap_*` wrappers. This provides complete sandbox isolation and allows the UI to be swapped (VM vs. DLL) without recompilation.
- **Fixed virtual resolution**: The 640×480 abstraction is a common pattern in this era, chosen for consistency with hardware limits and menu design conventions of the time.
- **String-based command dispatch**: Rather than structured data or callbacks, commands are text-based and parsed incrementally. This is simple to extend but brittle if arguments are out of order.
- **Cvar abuse**: Cvars are not designed for data binding, but the engine exploits them as a quick display-cache mechanism.

**Modern Alternatives:**
- A struct-based command type (like a tagged union) would be type-safe.
- Asset persistence would benefit from JSON or versioned binary formats (e.g., protobuf).
- Resolution scaling would be better served by a layout engine (like Flexbox or CSS) rather than per-coordinate manual scaling.
- The score system might use a database abstraction layer (SQLite, etc.) for robustness.

**Concepts Illustrated:**
- **Virtual coordinate systems** in real-time graphics.
- **VM/native boundary design** via trap syscalls and sandbox isolation.
- **Direct serialization** for simple persistence.
- **Cvar lifecycle** and secondary uses as transient state.

## Potential Issues

1. **Buffer overflow in `Com_Error` / `Com_Printf`** (lines 43–44, 52–53): Both use `vsprintf` into a fixed 1024-byte buffer without overflow protection. A malformed error message or varargs mismatch could corrupt the stack. Modern code would use `vsnprintf` with a size argument.

2. **No version checking on `.game` files**: The code reads a size field and compares it to `sizeof(postGameInfo_t)`, but doesn't validate that the file format matches the current struct definition. If the struct layout changes (e.g., new fields added, padding altered), old files silently fail to load with no recovery. A version number in the header would allow safe migration.

3. **Demo availability is file-existence-only**: `UI_LoadBestScores` checks whether a demo file exists (line 154) but doesn't validate it or ensure it's readable. A corrupted or incomplete demo file will still set `uiInfo.demoAvailable = qtrue`, potentially crashing playback.

4. **Uninitialized `newInfo` in `UI_LoadBestScores`** (line 136): The struct is `memset` to zero before reading, but if `trap_FS_FOpenFile` fails, `UI_SetBestScores` is called with an uninitialized or partially-initialized struct. This is only safe because `memset` is called first, but the pattern is fragile.

5. **No bounds check on score calculations**: In `UI_CalcPostGameStats`, the timeBonus is computed as `(adjustedTime - newInfo.time) * 10` without checking for integer overflow. If times are very large, the bonus could wrap. Unlikely in practice, but brittle.
