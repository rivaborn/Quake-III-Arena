# code/botlib/l_libvar.h

## File Purpose
Declares the botlib's internal configuration variable system (`libvar`), a lightweight cvar-like mechanism used exclusively within the bot library to store and query named string/float settings without going through the engine's cvar system.

## Core Responsibilities
- Define the `libvar_t` linked-list node structure for named variables
- Declare lifecycle management (allocation/deallocation of all vars)
- Declare lookup functions (by name, returning struct, string, or float)
- Declare create-or-get helpers (`LibVar`, `LibVarValue`, `LibVarString`)
- Declare mutation (`LibVarSet`) and change-detection (`LibVarChanged`, `LibVarSetNotModified`) interfaces

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `libvar_t` (`libvar_s`) | struct | Singly-linked list node holding a named botlib variable: name, string value, flags, modified flag, cached float value, and `next` pointer |

## Global / File-Static State
None declared in this header (the linked-list head is defined in `l_libvar.c`).

## Key Functions / Methods

### LibVarDeAllocAll
- Signature: `void LibVarDeAllocAll(void)`
- Purpose: Frees all allocated `libvar_t` nodes; used during botlib shutdown.
- Inputs: None
- Outputs/Return: None
- Side effects: Releases all heap memory for the var list; list head becomes null.
- Calls: Not inferable from this file.
- Notes: Must be called before botlib teardown to avoid leaks.

### LibVarGet
- Signature: `libvar_t *LibVarGet(char *var_name)`
- Purpose: Looks up and returns the `libvar_t` node matching `var_name`.
- Inputs: `var_name` — null-terminated variable name string.
- Outputs/Return: Pointer to matching `libvar_t`, or `NULL` if not found.
- Side effects: None.
- Calls: Not inferable from this file.

### LibVar
- Signature: `libvar_t *LibVar(char *var_name, char *value)`
- Purpose: Create-or-get; if the variable doesn't exist, allocates and initializes it with `value`; otherwise returns existing node.
- Inputs: `var_name`, default `value` string.
- Outputs/Return: Pointer to the (possibly newly created) `libvar_t`.
- Side effects: May allocate heap memory; inserts into the global list.
- Notes: Analogous to `Cvar_Get` in the engine.

### LibVarValue / LibVarString
- Signature: `float LibVarValue(char *var_name, char *value)` / `char *LibVarString(char *var_name, char *value)`
- Purpose: Convenience wrappers over `LibVar`; return the float or string value directly after create-or-get.
- Inputs: `var_name`, default `value`.
- Outputs/Return: Cached `float` or `char *` from the `libvar_t`.
- Side effects: Same as `LibVar` (potential allocation).

### LibVarSet
- Signature: `void LibVarSet(char *var_name, char *value)`
- Purpose: Updates an existing (or creates a new) variable's string and float value; sets `modified = qtrue`.
- Side effects: Mutates `libvar_t.string`, `.value`, `.modified`; may allocate.

### LibVarChanged / LibVarSetNotModified
- Purpose: Poll and reset the `modified` dirty flag on a named variable.
- Notes: Allows subsystems to detect and react to configuration changes without re-reading every frame.

## Control Flow Notes
This header is consumed during botlib initialization and throughout AI subsystem updates. Variables are created lazily on first access via `LibVar`/`LibVarValue`/`LibVarString`, queried per-frame by AI logic, and bulk-freed in `LibVarDeAllocAll` during shutdown. There is no engine-frame callback here; polling is done by callers as needed.

## External Dependencies
- `qboolean` — defined in `q_shared.h` (engine shared types); not defined in this file.
- Implementation body: `l_libvar.c` (defined elsewhere).
- No standard library headers included directly in this header.
