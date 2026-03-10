# code/botlib/be_aas_bspq3.c

## File Purpose
Provides the BSP world interface layer for the Q3 bot library (botlib), bridging AAS (Area Awareness System) navigation code to the engine's collision/BSP subsystem via the `botimport` callback table. It also owns the BSP entity data store, parsing and exposing map entity key-value pairs to the rest of botlib.

## Core Responsibilities
- Delegate spatial queries (traces, point contents, PVS/PHS tests) to engine callbacks via `botimport`
- Load and cache BSP entity lump data from the engine into `bspworld`
- Parse the raw entity text into a queryable linked-list of `bsp_entity_t` / `bsp_epair_t` records
- Provide typed accessors for entity key-value pairs (string, vector, float, int)
- Provide entity iteration (`AAS_NextBSPEntity`) and range validation
- Stub out unused BSP spatial linking functions (`AAS_UnlinkFromBSPLeaves`, `AAS_BSPLinkEntity`, `AAS_BoxEntities`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `rgb_t` | struct | Simple RGB integer color triplet (used only in `BSP_DEBUG` path) |
| `bsp_epair_t` | struct | Singly-linked key/value string pair for one entity attribute |
| `bsp_entity_t` | struct | Single BSP entity; owns a linked list of `bsp_epair_t` |
| `bsp_t` | struct | Top-level BSP world state: load flag, raw entity text buffer, parsed entity array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `bspworld` | `bsp_t` | global | Singleton holding all parsed BSP entity data for the currently loaded map |
| `botimport` | `botlib_import_t` (extern) | global | Engine-provided callback table; defined in `be_interface.c` |

## Key Functions / Methods

### AAS_Trace
- **Signature:** `bsp_trace_t AAS_Trace(vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int passent, int contentmask)`
- **Purpose:** Sweeps an axial bounding box through the BSP world.
- **Inputs:** Ray endpoints, box half-extents, entity to pass through, content filter mask.
- **Outputs/Return:** `bsp_trace_t` describing the first hit.
- **Side effects:** None local; delegates entirely to `botimport.Trace`.
- **Calls:** `botimport.Trace`
- **Notes:** Thin wrapper; all collision math is in the engine.

### AAS_EntityCollision
- **Signature:** `qboolean AAS_EntityCollision(int entnum, vec3_t start, vec3_t boxmins, vec3_t boxmaxs, vec3_t end, int contentmask, bsp_trace_t *trace)`
- **Purpose:** Tests a swept box against a single entity; updates `*trace` if the entity hit is closer.
- **Inputs:** Entity number, ray, box extents, content mask, current best trace.
- **Outputs/Return:** `qtrue` if entity was hit closer than existing trace; mutates `*trace` in place.
- **Side effects:** Overwrites `*trace` via `Com_Memcpy` when a closer hit is found.
- **Calls:** `botimport.EntityTrace`, `Com_Memcpy`
- **Notes:** Caller must supply a valid prior trace; fraction comparison is the decision criterion.

### AAS_LoadBSPFile
- **Signature:** `int AAS_LoadBSPFile(void)`
- **Purpose:** Initialises `bspworld` from the engine's entity data for the current map.
- **Inputs:** None (reads from `botimport.BSPEntityData()`).
- **Outputs/Return:** `BLERR_NOERROR` on success.
- **Side effects:** Calls `AAS_DumpBSPData` (clears previous data), allocates hunk memory, copies entity text, calls `AAS_ParseBSPEntities`, sets `bspworld.loaded = qtrue`.
- **Calls:** `AAS_DumpBSPData`, `botimport.BSPEntityData`, `GetClearedHunkMemory`, `Com_Memcpy`, `AAS_ParseBSPEntities`

### AAS_ParseBSPEntities
- **Signature:** `void AAS_ParseBSPEntities(void)`
- **Purpose:** Tokenises `bspworld.dentdata` and builds the `bspworld.entities[]` array with their epair chains.
- **Inputs:** None (reads `bspworld.dentdata`/`entdatasize`).
- **Outputs/Return:** Void; populates `bspworld.numentities` and entity/epair structures.
- **Side effects:** Allocates hunk memory for each `bsp_epair_t` and key/value strings. On parse error, calls `AAS_FreeBSPEntities` and frees the script.
- **Calls:** `LoadScriptMemory`, `SetScriptFlags`, `PS_ReadToken`, `PS_ExpectTokenType`, `ScriptError`, `AAS_FreeBSPEntities`, `FreeScript`, `GetClearedHunkMemory`, `GetHunkMemory`, `StripDoubleQuotes`, `botimport.Print`
- **Notes:** Caps entity count at `MAX_BSPENTITIES` (2048). Entity index 0 is reserved; parsing starts at index 1.

### AAS_FreeBSPEntities
- **Signature:** `void AAS_FreeBSPEntities(void)`
- **Purpose:** Frees all allocated epair key/value strings and epair nodes; resets `numentities` to 0.
- **Side effects:** Calls `FreeMemory` on all epair keys, values, and nodes.

### AAS_DumpBSPData
- **Signature:** `void AAS_DumpBSPData(void)`
- **Purpose:** Full teardown of `bspworld`: frees entities, frees `dentdata`, zeroes the struct.
- **Side effects:** Calls `AAS_FreeBSPEntities`, `FreeMemory`, `Com_Memset`. Sets `loaded = qfalse`.

### AAS_ValueForBSPEpairKey
- **Signature:** `int AAS_ValueForBSPEpairKey(int ent, char *key, char *value, int size)`
- **Purpose:** Looks up a string value for a named key on a BSP entity.
- **Inputs:** Entity index, key name, output buffer and its size.
- **Outputs/Return:** `qtrue`/`qfalse`; fills `value` (null-terminated, size-safe).
- **Calls:** `AAS_BSPEntityInRange`, `strcmp`, `strncpy`

- **Notes:** `AAS_VectorForBSPEpairKey`, `AAS_FloatForBSPEpairKey`, `AAS_IntForBSPEpairKey` are thin typed wrappers around this function using `sscanf`/`atof`/`atoi`.

## Control Flow Notes
- **Init:** `AAS_LoadBSPFile` is called during botlib map load (triggered from `be_aas_main.c`). It must be called before any entity queries.
- **Frame/Query:** `AAS_Trace`, `AAS_PointContents`, `AAS_inPVS` are called per-frame by AAS movement and sampling code.
- **Shutdown:** `AAS_DumpBSPData` is called on map unload.
- The three stub functions (`AAS_UnlinkFromBSPLeaves`, `AAS_BSPLinkEntity`, `AAS_BoxEntities`) are no-ops in this Q3-specific implementation; they exist to satisfy the AAS interface.
- `AAS_inPHS` always returns `qtrue` (PHS not implemented for Q3).

## External Dependencies
- `../game/q_shared.h` — shared types (`vec3_t`, `qboolean`, `Com_Memcpy`, `Com_Memset`)
- `l_memory.h` — `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_script.h` — script tokeniser (`LoadScriptMemory`, `PS_ReadToken`, `PS_ExpectTokenType`, `FreeScript`, etc.)
- `be_aas_def.h` / `be_aas_funcs.h` — AAS internal types (`bsp_trace_t`, `bsp_link_t`)
- `aasfile.h` — AAS file format constants
- `../game/botlib.h` — `botlib_import_t` definition, `BLERR_NOERROR`, `MAX_EPAIRKEY`
- `../game/be_aas.h` — public AAS interface types
- `botimport` — defined in `be_interface.c`; all engine calls go through this table
