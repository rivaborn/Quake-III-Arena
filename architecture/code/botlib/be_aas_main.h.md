# code/botlib/be_aas_main.h

## File Purpose
Public and internal header for the AAS (Area Awareness System) main module within Quake III's botlib. It declares the primary lifecycle functions for initializing, loading, updating, and shutting down the AAS world, as well as a small set of public utility queries.

## Core Responsibilities
- Guard internal AAS lifecycle functions behind the `AASINTERN` preprocessor gate
- Expose the global `aasworld` state to other internal AAS modules
- Declare public query functions usable outside the AAS internals (initialized state, loaded state, time, model index lookup)
- Declare a geometric utility function for projecting a point onto a line segment

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_t` | struct (defined elsewhere) | The monolithic AAS world state; externally declared as `aasworld` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aasworld` | `aas_t` | global (extern, `AASINTERN` only) | Singleton AAS world state shared across all internal AAS subsystems |

## Key Functions / Methods

### AAS_Error
- **Signature:** `void QDECL AAS_Error(char *fmt, ...)`
- **Purpose:** Variadic error reporting for AAS subsystem failures.
- **Inputs:** `fmt` — printf-style format string; variadic arguments.
- **Outputs/Return:** void
- **Side effects:** Likely calls `botimport.Print` or equivalent; may set error/shutdown flags. Not inferable from this file alone.
- **Calls:** Not inferable from this file.
- **Notes:** `QDECL` ensures correct calling convention for cross-module use. Internal only (`AASINTERN`).

### AAS_SetInitialized
- **Signature:** `void AAS_SetInitialized(void)`
- **Purpose:** Marks the AAS system as fully initialized.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes to initialization flag in `aasworld` or equivalent global.
- **Notes:** Internal only.

### AAS_Setup
- **Signature:** `int AAS_Setup(void)`
- **Purpose:** Allocates and configures AAS data structures for the current entity/client count.
- **Inputs:** None (reads configuration from global state)
- **Outputs/Return:** Non-zero on success, 0 on failure (convention inferred from Q3 pattern).
- **Side effects:** Memory allocation; modifies `aasworld`.
- **Notes:** Internal only; called before map load.

### AAS_Shutdown
- **Signature:** `void AAS_Shutdown(void)`
- **Purpose:** Frees all AAS resources and resets state.
- **Side effects:** Memory deallocation; zeroes/resets `aasworld`.
- **Notes:** Internal only; called on level unload or engine shutdown.

### AAS_LoadMap
- **Signature:** `int AAS_LoadMap(const char *mapname)`
- **Purpose:** Loads the `.aas` file corresponding to the given map name and populates `aasworld`.
- **Inputs:** `mapname` — BSP/map name string.
- **Outputs/Return:** Non-zero on success.
- **Side effects:** File I/O; large allocations into `aasworld`.
- **Notes:** Internal only; triggers full AAS data parse.

### AAS_StartFrame
- **Signature:** `int AAS_StartFrame(float time)`
- **Purpose:** Advances the AAS simulation clock and performs per-frame updates (entity tracking, reachability refresh, etc.).
- **Inputs:** `time` — current game time in seconds.
- **Outputs/Return:** Non-zero on success.
- **Side effects:** Updates `aasworld.time`; may update entity states.
- **Notes:** Internal only; called once per server frame.

### AAS_Initialized / AAS_Loaded
- **Purpose:** Public predicates returning whether AAS is initialized or has a map loaded. Used by callers outside `AASINTERN` to guard AAS queries.

### AAS_ModelFromIndex / AAS_IndexFromModel
- **Purpose:** Bidirectional lookup between model name strings and integer indices in the AAS model table. Public API.

### AAS_Time
- **Signature:** `float AAS_Time(void)`
- **Purpose:** Returns the current AAS world time as set by the last `AAS_StartFrame` call. Public API.

### AAS_ProjectPointOntoVector
- **Signature:** `void AAS_ProjectPointOntoVector(vec3_t point, vec3_t vStart, vec3_t vEnd, vec3_t vProj)`
- **Purpose:** Projects `point` onto the infinite line defined by `vStart`→`vEnd`, storing the result in `vProj`.
- **Inputs:** `point`, `vStart`, `vEnd` — 3D vectors.
- **Outputs/Return:** Result written into `vProj` (output parameter).
- **Side effects:** None.
- **Notes:** Public utility; used by path/movement code for closest-point-on-segment queries.

## Control Flow Notes
This header participates in AAS lifecycle: `AAS_Setup` → `AAS_LoadMap` → repeated `AAS_StartFrame` calls per server frame → `AAS_Shutdown`. The `AASINTERN` guard ensures only `be_aas_main.c` and other internal AAS `.c` files (which define `AASINTERN`) see the mutable lifecycle API; external consumers see only the read-only public interface.

## External Dependencies
- `aas_t` — defined in `be_aas_def.h` (included by internal files before this header)
- `vec3_t` — defined in `q_shared.h`
- `QDECL` — calling-convention macro from `q_shared.h`
- `AASINTERN` — preprocessor symbol defined by internal AAS compilation units
