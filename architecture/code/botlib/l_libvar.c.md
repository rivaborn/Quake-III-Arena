# code/botlib/l_libvar.c

## File Purpose
Implements a lightweight key-value variable system ("libvars") internal to the bot library. It provides create, read, update, and delete operations for named variables that store both a string value and a precomputed float value, independent of the engine's main cvar system.

## Core Responsibilities
- Allocate and free individual `libvar_t` nodes from bot library heap
- Maintain a singly-linked global list of all active libvars
- Perform lazy creation: create a variable on first access with a default value
- Convert string values to floats via a custom parser (`LibVarStringValue`)
- Track a `modified` flag per variable so callers can poll for changes
- Provide bulk teardown of all libvars at bot library shutdown

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `libvar_t` | struct (typedef) | A single named variable node: holds `name`, `string`, `flags`, `modified`, `value` (float), and `next` pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `libvarlist` | `libvar_t *` | global | Head of the singly-linked list of all allocated libvars |

## Key Functions / Methods

### LibVarStringValue
- **Signature:** `float LibVarStringValue(char *string)`
- **Purpose:** Converts a decimal ASCII string to a float without using `atof`.
- **Inputs:** Null-terminated string of digits with at most one `.`
- **Outputs/Return:** Parsed float; returns `0` on first non-digit, non-dot character.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Only handles positive decimals; returns `0` immediately on any unexpected character, including leading minus signs or exponents.

### LibVarAlloc
- **Signature:** `libvar_t *LibVarAlloc(char *var_name)`
- **Purpose:** Allocates a new `libvar_t` node (name string packed into the same allocation) and prepends it to `libvarlist`.
- **Inputs:** Variable name string.
- **Outputs/Return:** Pointer to the new (zeroed) `libvar_t`.
- **Side effects:** Modifies `libvarlist`; allocates heap memory via `GetMemory`.
- **Calls:** `GetMemory`, `Com_Memset`, `strcpy`.
- **Notes:** Name is stored immediately after the struct in one contiguous allocation; `v->string` is NOT set here.

### LibVarDeAlloc
- **Signature:** `void LibVarDeAlloc(libvar_t *v)`
- **Purpose:** Frees the string buffer (if any) and then the node itself.
- **Inputs:** Valid `libvar_t *`.
- **Side effects:** Heap free via `FreeMemory`. Does not unlink from `libvarlist`.
- **Calls:** `FreeMemory`.
- **Notes:** Caller is responsible for list unlinking before calling.

### LibVarDeAllocAll
- **Signature:** `void LibVarDeAllocAll(void)`
- **Purpose:** Iterates `libvarlist`, freeing every node; sets `libvarlist = NULL`.
- **Side effects:** Fully drains the libvar list and all associated heap memory.
- **Calls:** `LibVarDeAlloc`.

### LibVarGet
- **Signature:** `libvar_t *LibVarGet(char *var_name)`
- **Purpose:** Linear search of `libvarlist` by case-insensitive name match.
- **Outputs/Return:** Matching `libvar_t *` or `NULL`.
- **Calls:** `Q_stricmp`.

### LibVar
- **Signature:** `libvar_t *LibVar(char *var_name, char *value)`
- **Purpose:** Get-or-create: returns existing variable or allocates a new one with the given default string value and computed float.
- **Side effects:** May allocate heap; sets `modified = qtrue` on creation.
- **Calls:** `LibVarGet`, `LibVarAlloc`, `GetMemory`, `strcpy`, `LibVarStringValue`.

### LibVarSet
- **Signature:** `void LibVarSet(char *var_name, char *value)`
- **Purpose:** Unconditionally sets a libvar's string and recomputed float value, creating the node if absent. Always sets `modified = qtrue`.
- **Side effects:** Frees old string, allocates new string, modifies `libvarlist`.
- **Calls:** `LibVarGet`, `FreeMemory`, `LibVarAlloc`, `GetMemory`, `strcpy`, `LibVarStringValue`.

### LibVarChanged / LibVarSetNotModified
- Trivial wrappers that read/clear `v->modified` via `LibVarGet`; return `qfalse`/no-op if variable not found.

### Notes
- `LibVarGetString`, `LibVarGetValue`, `LibVarString`, `LibVarValue` are thin accessors delegating to `LibVarGet`/`LibVar`.

## Control Flow Notes
- **Init:** `LibVar` / `LibVarSet` calls implicitly build the list as bot subsystems request variables.
- **Shutdown:** `LibVarDeAllocAll` is called during bot library teardown (from `be_aas_main.c` / `be_interface.c`) to free all nodes.
- No per-frame update; variables are polled on demand via `LibVarChanged`.

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `qtrue`/`qfalse`, `Q_stricmp`, `Com_Memset`
- `l_memory.h` — `GetMemory`, `FreeMemory` (bot library heap wrappers)
- `strcpy`, `strlen` — C standard library (available because this code compiles outside the Q3VM)
- `libvar_t` defined in `l_libvar.h`
