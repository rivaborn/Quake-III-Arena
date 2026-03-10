# code/bspc/aas_cfg.c — Enhanced Analysis

## Architectural Role

This file is the **configuration bridge** between the offline BSPC map compiler and the online botlib AAS navigation subsystem. It provides a reflection-based, generic config parser that parameterizes all physics and reachability behavior used during AAS file generation from BSP maps. The loaded `cfg` global is then published into the botlib libvar system, making it available to all downstream `be_aas_*.c` modules that drive reachability analysis and movement simulation. This decouples hardcoded Q3A defaults from per-game customization (e.g., different friction, jump heights, or bbox sizes for Team Arena).

## Key Cross-References

### Incoming (Callers)
- **BSPC tool initialization** (`code/bspc/bspc.c` / `code/bspc/be_aas_bspc.c`): Calls `DefaultCfg()` first, then `LoadCfgFile(filename)` to override from a game-specific `.cfg` file during map compilation.
- **Reachability computation** (`code/bspc/be_aas_bspc.c` → `AAS_CalcReachAndClusters`): Depends on `cfg` being populated before reachability analysis begins.

### Outgoing (Dependencies)
- **Botlib precompiler** (`code/botlib/l_precomp.h`, `l_struct.h`, `l_libvar.h`): Provides generic config parsing infrastructure (`LoadSourceFile`, `PC_ReadToken`, `ReadStructure`, `LibVarSet`).
- **Botlib movement** (`code/botlib/be_aas_move.c`): Reads `cfg` fields (gravity direction, max velocities, steepness, jump parameters) to simulate movement arcs and validate reachability.
- **Botlib reach computation** (`code/botlib/be_aas_reach.c`, `be_aas_sample.c`): Bounding boxes define which player sizes can reach an area; presence-type logic gates area connectivity based on player stance.
- **Game VM** (`code/game/g_*.c`): May optionally read some cfg values at runtime via the botlib API (see shared `cfg_t` typedef in `code/game/`).

## Design Patterns & Rationale

**Reflection-based field descriptors**: The `fielddef_t` offset-macro pattern (lines 34–37) is a late-90s technique for binary-safe struct serialization without requiring language reflection. By pairing field name, offset, and type in arrays, `ReadStructure` can parse text config files into arbitrary struct layouts without hardcoded parsing code. This is **generic but opaque**: modern engines use JSON/TOML with schema validation.

**Sentinel sentinel pattern**: `FLT_MAX` (line 108) marks "field not set by config file"; `SetCfgLibVars` (line 195) skips publishing those fields. This avoids clobbering hardcoded defaults, but is fragile (collisions with legitimate max values) and non-obvious.

**Two-slot varargs buffer**: The `va()` ping-pong pattern (lines 153–164) allows nested calls without explicit temporary string allocation, typical of Quake-era code. It's non-thread-safe and fails beyond one level of nesting—acceptable in a single-threaded offline tool but noted as a limitation.

**Validation minimal**: Only gravity-direction magnitude and bbox count are checked; invalid float ranges (e.g., negative friction) are silently accepted, delegating validation to AAS build or runtime.

## Data Flow Through This File

1. **Init time**: `DefaultCfg()` zeros the global `cfg` and writes safe Q3A defaults (two bboxes: standing 32 units tall, crouching 16; gravity -1 in Z).
2. **Config load**: `LoadCfgFile(filename)` calls the botlib precompiler to parse `bbox` and `settings` blocks, populating the global `cfg` struct in-place.
3. **Publication**: `SetCfgLibVars()` iterates `cfg_fields` and calls `LibVarSet` for each non-`FLT_MAX` float, registering them in the botlib variable store.
4. **Consumption**: Downstream `be_aas_*.c` modules read `cfg.phys_*` and `cfg.rs_*` fields directly (or via libvar) to drive reachability simulation. Bbox list in `cfg.bboxes[0..numbboxes-1]` defines player collision shapes for area accessibility tests.

## Learning Notes

**Generic config parsing**: This is an idiomatic Q3A approach—fielddef arrays define struct layout without language-level reflection. Modern engines use JSON/TOML loaders with type-safe schema definitions. The tradeoff: minimal code per struct type but debugging is harder and schema errors are runtime-only.

**Offline tool parameterization**: The BSPC tool (and botlib within it) is **highly configurable**. Different games (Q3A, Team Arena, modded variants) can have different physics, presence-type definitions, and reachability rules. The config file is the lever for this customization without recompiling the tool.

**Shared VM/tool boundary**: The `cfg_t` typedef is defined in both `code/bspc/` (for the offline compiler) and `code/game/` (for the runtime bot library). This suggests the **same config system is usable at runtime** if a game wants to expose dynamic bot physics tuning—though in practice, Q3A bakes the config into the `.aas` file and loads it as read-only binary data.

**Physics-first design**: All config fields are physics or reachability related (gravity, friction, max velocity, jump velocity, barrier height, fall damage). This reflects botlib's core mission: *simulate player movement faithfully to determine connectivity*. Contrast with modern AI config (behavior trees, perception ranges, team tactics), which are absent here.

## Potential Issues

- **No schema validation**: Callers can load a corrupt `.cfg` file with garbage float values; only gravity direction is range-checked. Negative friction or max velocity would silently propagate downstream and cause nonsensical reachability results.
- **Silent integer/array omission**: `SetCfgLibVars` only publishes float fields; integer bbox-related config is loaded but never registered in libvars, potentially misleading downstream code that expects all fields to be registered.
- **FLT_MAX ambiguity**: If a legitimate config parameter should be `FLT_MAX` (e.g., unlimited max velocity), the sentinel pattern breaks. No clear guidance in code.
- **Thread-unsafe `va()`**: The ping-pong buffer is unsafe if BSPC ever becomes multi-threaded or if config loading is deferred; not currently an issue but worth documenting.

---

**Token count estimate:** ~1200 words, under 1500 target.
