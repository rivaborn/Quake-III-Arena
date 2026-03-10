# code/qcommon/cvar.c

## File Purpose
Implements Quake III Arena's console variable (cvar) system, providing dynamic runtime configuration variables accessible from the console, config files, and C code. It manages cvar storage, lookup, value setting with protection flags, and VM-module bridging.

## Core Responsibilities
- Allocate and register cvars in a fixed-size pool with hash-table fast lookup
- Enforce protection flags: `CVAR_ROM`, `CVAR_INIT`, `CVAR_LATCH`, `CVAR_CHEAT`
- Track modification state per-cvar and globally via `cvar_modifiedFlags`
- Provide console commands: `toggle`, `set`, `sets`, `setu`, `seta`, `reset`, `cvarlist`, `cvar_restart`
- Serialize archived cvars to config file via `Cvar_WriteVariables`
- Bridge native cvars to VM (QVM) modules via `vmCvar_t` handle system
- Build info strings for userinfo/serverinfo/systeminfo network transmission

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cvar_t` | struct (defined in q_shared.h) | Core cvar node: name, string, resetString, latchedString, flags, modificationCount, value, integer, linked-list + hash-list pointers |
| `vmCvar_t` | struct (defined in q_shared.h) | VM-safe cvar mirror: integer handle, modificationCount, value, integer, string copy |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cvar_vars` | `cvar_t *` | global | Head of all-cvars linked list |
| `cvar_cheats` | `cvar_t *` | global | Pointer to `sv_cheats`; gates `CVAR_CHEAT` writes |
| `cvar_modifiedFlags` | `int` | global | OR-accumulator of flags from any modified cvar; polled by server/client to detect class-level changes |
| `cvar_indexes[MAX_CVARS]` | `cvar_t[1024]` | static (file) | Fixed pool for all cvar allocations |
| `cvar_numIndexes` | `int` | static (file) | Count of allocated entries in `cvar_indexes` |
| `hashTable[FILE_HASH_SIZE]` | `cvar_t*[256]` | static (file) | Hash buckets for O(1) name lookup |

## Key Functions / Methods

### Cvar_Get
- **Signature:** `cvar_t *Cvar_Get( const char *var_name, const char *var_value, int flags )`
- **Purpose:** Primary registration entry point. Returns existing cvar (merging flags and updating resetString) or allocates a new one from `cvar_indexes`.
- **Inputs:** Variable name, default value string, flag bitmask
- **Outputs/Return:** Pointer to `cvar_t`; fatal error if pool exhausted
- **Side effects:** Allocates `CopyString` heap strings; inserts into `cvar_vars` list and `hashTable`; may call `Cvar_Set2` to apply a pending `latchedString`
- **Calls:** `Cvar_ValidateString`, `Cvar_FindVar`, `Z_Free`, `CopyString`, `Cvar_Set2`, `Com_Error`, `Com_DPrintf`
- **Notes:** `CVAR_USER_CREATED` flag is stripped when C code re-registers a user-set variable, promoting it to a proper cvar with the C default as resetString.

### Cvar_Set2
- **Signature:** `cvar_t *Cvar_Set2( const char *var_name, const char *value, qboolean force )`
- **Purpose:** Core write implementation used by all public set variants. Enforces ROM/INIT/LATCH/CHEAT protection when `force == qfalse`.
- **Inputs:** Name, new value (NULL = reset to resetString), force flag
- **Outputs/Return:** Pointer to (possibly unchanged) `cvar_t`
- **Side effects:** Updates `cvar_modifiedFlags`; frees/replaces `latchedString` or `string`; increments `modificationCount`; sets `modified = qtrue`
- **Calls:** `Cvar_ValidateString`, `Cvar_FindVar`, `Cvar_Get`, `Z_Free`, `CopyString`, `Com_Printf`, `Com_DPrintf`
- **Notes:** LATCH writes store value in `latchedString` and defer application until next `Cvar_Get` call; duplicate-value check avoids spurious modification.

### Cvar_SetCheatState
- **Signature:** `void Cvar_SetCheatState( void )`
- **Purpose:** Resets all `CVAR_CHEAT` cvars to their `resetString` when cheats are disabled.
- **Side effects:** Clears any pending `latchedString` on cheat vars before resetting; calls `Cvar_Set` per affected cvar
- **Notes:** Prevents `CVAR_LATCH|CVAR_CHEAT` combinations from escaping the reset via a stale latch.

### Cvar_Restart_f
- **Signature:** `void Cvar_Restart_f( void )`
- **Purpose:** Console command handler; resets all non-ROM/INIT/NORESTART cvars to hardcoded defaults, removing user-created cvars from the list entirely.
- **Side effects:** `Z_Free`s name/string/latch/reset strings of purged entries; `Com_Memset`s the pool slot to zero (index slot cannot be reclaimed)

### Cvar_Register / Cvar_Update
- **Signature:** `void Cvar_Register( vmCvar_t*, const char*, const char*, int )` / `void Cvar_Update( vmCvar_t* )`
- **Purpose:** VM bridge — maps a `vmCvar_t` handle to a pool index and synchronizes value/integer/string into the VM-accessible struct on demand.
- **Side effects:** `Cvar_Update` copies string into `vmCvar->string` (bounded to `MAX_CVAR_VALUE_STRING`); drops with `ERR_DROP` on out-of-range handle or oversized string.

### Cvar_InfoString / Cvar_InfoString_Big
- **Signature:** `char *Cvar_InfoString( int bit )` / `char *Cvar_InfoString_Big( int bit )`
- **Purpose:** Builds a key=value info string from all cvars whose flags match `bit` (e.g. `CVAR_USERINFO`, `CVAR_SERVERINFO`).
- **Outputs/Return:** Pointer to a **static** internal buffer — not reentrant.
- **Side effects:** None beyond writing the static buffer.

### Notes (minor helpers)
- `generateHashValue`: polynomial hash over lowercased name, masked to 256 buckets.
- `Cvar_ValidateString`: rejects names/values containing `\`, `"`, or `;` (command-injection prevention).
- `Cvar_FindVar`: hash lookup with case-insensitive `Q_stricmp` chain walk.
- `Cvar_WriteVariables`: iterates list, writes `seta name "value"` lines for `CVAR_ARCHIVE` cvars, skipping `cl_cdkey`.

## Control Flow Notes
`Cvar_Init` is called during engine startup (within `Com_Init`). It registers `sv_cheats` as a ROM cvar and binds all console commands. Thereafter the cvar system is stateless between frames — no per-frame update function; callers poll `cvar_modifiedFlags` and per-cvar `modificationCount` to detect changes. `Cvar_SetCheatState` is called by the server when `sv_cheats` transitions to 0.

## External Dependencies
- **Includes:** `../game/q_shared.h` (types, `cvar_t`, `vmCvar_t`, flag constants), `qcommon.h` (Z_*, `CopyString`, `Com_*`, `Cmd_*`, `FS_Printf`, `Info_SetValueForKey*`)
- **Defined elsewhere:** `Z_Free`, `Z_Malloc`/`CopyString`, `Com_Error`, `Com_Printf`, `Com_DPrintf`, `Com_sprintf`, `Com_Filter`, `Cmd_Argc`, `Cmd_Argv`, `Cmd_AddCommand`, `FS_Printf`, `Info_SetValueForKey`, `Info_SetValueForKey_Big`, `Q_stricmp`, `Q_strncpyz`
