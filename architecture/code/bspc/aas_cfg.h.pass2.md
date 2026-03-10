# code/bspc/aas_cfg.h — Enhanced Analysis

## Architectural Role

This header defines the configuration contract for the **offline AAS compiler phase** (BSPC). The `cfg_t` structure bridges tool-time physics tuning and offline area classification: BSPC reads `cfg` to parameterize reachability analysis—determining which areas are connected and at what "effort" cost—then bakes the resulting connectivity and cost weights into the compiled `.aas` binary. Runtime botlib consumes that precomputed AAS file, making `cfg` purely a BSPC concern. This design cleanly separates **tuning policy** (offline) from **navigation execution** (online).

## Key Cross-References

### Incoming (who reads/writes this file)

- **BSPC tool entry** (`code/bspc/be_aas_bspc.c`): calls `DefaultCfg()` and `LoadCfgFile()` to initialize `cfg` at tool startup
- **Reachability analysis** (`code/botlib/be_aas_reach.c`, invoked from BSPC): reads `cfg.rs_*` scoring constants when assigning cost/difficulty to inter-area links
- **Movement simulation** (`code/botlib/be_aas_move.c`, invoked from BSPC): reads `cfg.phys_*` to simulate jump arcs, fall distances, and climb capabilities during reachability validation
- **Area creation** (`code/bspc/aas_create.c`): uses bounding boxes and presence types from `cfg` to classify spatial regions during BSP→AAS conversion

### Outgoing (what this file depends on)

- Relies on `aas_bbox_t` and `AAS_MAX_BBOXES` macro (likely from `code/botlib/be_aas_def.h` or `code/bspc/aasfile.h`)
- Uses `vec3_t` from shared math layer (`code/game/q_shared.h`)
- Global `cfg` instance must be declared and linked in `code/bspc/aas_cfg.c` (implementation file, not visible here)

## Design Patterns & Rationale

**Config-as-Struct for Offline Compilation**  
Rather than hardcoding physics constants in reachability functions, Q3 centralizes them in a singleton `cfg` struct. This allows:
- **Per-game tuning** via config files (different gravity, jump power, friction for TA vs. Q3)
- **Reproducible builds**: same config → same AAS file
- **Visibility**: a single file shows all physics assumptions baked into navigation

**Dual Presence-Type Bounding Boxes**  
The `BBOXFL_GROUNDED` / `BBOXFL_NOTGROUNDED` flags suggest that bounding boxes vary by movement state. This is a spatial classification optimization: grounded movement (walking) uses a different capsule than airborne movement (jumping/falling). The presence-type system in botlib (referenced in the first-pass) likely consults these flags during point-to-area mapping.

**Reachability Scoring Decoupling**  
The `rs_*` fields (waterjump, teleport, barrierjump, etc.) are **not binary reachability flags**—they are cost multipliers. A high `rs_rocketjump` value means "rocket jumps are hard/expensive to route through." This allows the pathfinder to prefer ground routes but fallback to rocket jumps if necessary, rather than treating them as binary capabilities.

## Data Flow Through This File

```
TOOL STARTUP
    ↓
DefaultCfg() [hard-coded defaults]
    ↓
LoadCfgFile("aas.cfg") [override with per-map/per-game tuning]
    ↓
[cfg global now populated]
    ↓
AAS COMPILATION PHASE
    ├─ AAS_Create() → AAS_CreateAreas() [use bbox and presence types]
    ├─ AAS_ContinueInitReachability() → AAS_Reachability_* [read cfg.phys_* and cfg.rs_*]
    ├─ AAS_InitClustering() [read gravity direction for portal detection]
    └─ Write .aas file with precomputed reachability + costs
    ↓
[cfg is NOT used at runtime; botlib loads .aas file directly]
```

## Learning Notes

**Legacy Physics Model**: The presence of `phys_gravitydirection` (a full `vec3_t`) rather than `1.0f` downward suggests Q3's AAS was designed with flexibility for non-vertical gravity—useful in zero-G mods or elaborate level design. Modern engines typically hardcode gravity direction.

**Effort-Based Routing**: Q3's distinction between reachability (can you reach area B from A?) and cost (how hard is it?) is subtle but powerful. Many engines treat movement types as binary capabilities; Q3 allows the pathfinder to trade path length against movement-type difficulty, enabling more organic bot routes.

**Two-Phase Compilation**: The separation of `DefaultCfg` and `LoadCfgFile` is intentional—it allows BSPC to ship with sensible defaults but accept per-project/per-map overrides. This was critical for modding communities.

## Potential Issues

**No Validation on Load**: `LoadCfgFile` returns an `int` (success/fail), but there's no visible bounds checking or range validation in this header. Physics values like `phys_jumpvel` could be set to zero or negative, silently breaking reachability analysis.

**Config Persistence**: At runtime, `code/botlib` does not re-read this file; the AAS binary is immutable. If a server operator tweaks `aas.cfg` but forgets to recompile the `.aas` file, they'll see no effect. This is-by-design but a common UX pitfall in offline-compiled systems.
