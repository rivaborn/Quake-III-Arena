# code/bspc/l_bsp_ent.c

## File Purpose
Parses and manages BSP map entity data for the BSPC (BSP Compiler) tool. It reads entity key-value pair lists from script tokens and provides accessor functions to query and mutate entity properties at compile time.

## Core Responsibilities
- Parse `{key value}` entity blocks from a script stream into `entity_t` structures
- Allocate and populate `epair_t` key-value pairs via the botlib script tokenizer
- Provide get/set accessors for entity key-value pairs (string, float, vector)
- Maintain a global flat array of all parsed map entities
- Strip trailing whitespace from parsed keys and values

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `entity_t` | struct (defined in `l_bsp_ent.h`) | Represents a single map entity; holds a linked list of `epair_t` |
| `epair_t` | struct (defined in `l_bsp_ent.h`) | A single key-value pair; singly linked list node |
| `script_t` | struct (from `l_script.h`) | Lexical parser state; cursor into a text buffer |
| `token_t` | struct (from `l_script.h`) | A single lexed token with string, type, and subtype |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `num_entities` | `int` | global | Count of entities parsed so far |
| `entities` | `entity_t[MAX_MAP_ENTITIES]` | global | Flat array storing all parsed map entities |

## Key Functions / Methods

### StripTrailing
- **Signature:** `void StripTrailing(char *e)`
- **Purpose:** Removes trailing ASCII control/whitespace characters (≤ 0x20) in-place
- **Inputs:** `e` — null-terminated string to strip
- **Outputs/Return:** void (modifies in place)
- **Side effects:** Writes `\0` over trailing whitespace
- **Calls:** `strlen`
- **Notes:** Called on both key and value after parsing to sanitize input

### ParseEpair
- **Signature:** `epair_t *ParseEpair(script_t *script)`
- **Purpose:** Reads the next two tokens from the script (key and value), allocates a new `epair_t`, and returns it
- **Inputs:** `script` — active lexer state pointing at the next key token
- **Outputs/Return:** Heap-allocated `epair_t *`; calls `Error()` on token-too-long
- **Side effects:** Allocates memory via `GetMemory`/`copystring`; calls `Error()` fatally if key ≥ 32 or value ≥ 1024 chars
- **Calls:** `GetMemory`, `memset`, `PS_ExpectAnyToken`, `StripDoubleQuotes`, `strlen`, `Error`, `copystring`, `StripTrailing`
- **Notes:** Enforces `MAX_KEY=32` and `MAX_VALUE=1024` limits

### ParseEntity
- **Signature:** `qboolean ParseEntity(script_t *script)`
- **Purpose:** Reads a `{ key value ... }` block from the script and appends a fully populated `entity_t` to the global `entities[]` array
- **Inputs:** `script` — active lexer state
- **Outputs/Return:** `true` if an entity was parsed; `false` if no opening token found (EOF)
- **Side effects:** Increments `num_entities`; writes into `entities[]` global; allocates `epair_t` heap nodes; calls `Error()` on malformed input or overflow
- **Calls:** `PS_ReadToken`, `strcmp`, `Error`, `PS_UnreadLastToken`, `ParseEpair`
- **Notes:** `epairs` are prepended (LIFO order relative to parse order); returns `false` cleanly at EOF to signal end-of-entities

### SetKeyValue
- **Signature:** `void SetKeyValue(entity_t *ent, char *key, char *value)`
- **Purpose:** Updates an existing key's value on an entity, or adds a new epair if not found
- **Inputs:** `ent` — target entity; `key` — key string; `value` — new value string
- **Outputs/Return:** void
- **Side effects:** May free old value via `FreeMemory`; allocates via `GetMemory`/`copystring`
- **Calls:** `strcmp`, `FreeMemory`, `copystring`, `GetMemory`

### ValueForKey
- **Signature:** `char *ValueForKey(entity_t *ent, char *key)`
- **Purpose:** Returns the string value for a given key, or `""` if not found
- **Inputs:** `ent`, `key`
- **Outputs/Return:** Pointer to existing value string or static `""`
- **Side effects:** None
- **Calls:** `strcmp`

### FloatForKey
- **Signature:** `vec_t FloatForKey(entity_t *ent, char *key)`
- **Purpose:** Returns the float value of an entity key
- **Calls:** `ValueForKey`, `atof`

### GetVectorForKey
- **Signature:** `void GetVectorForKey(entity_t *ent, char *key, vec3_t vec)`
- **Purpose:** Parses a space-separated triple from an entity key into a `vec3_t`
- **Inputs:** `ent`, `key`, `vec` — output vector
- **Side effects:** Writes to `vec`
- **Calls:** `ValueForKey`, `sscanf`
- **Notes:** Uses `double` intermediates to be independent of `vec_t` size (`float` vs `double`)

### PrintEntity
- **Notes:** Debug utility; prints all epairs to stdout via `printf`. Not an engine entry point.

## Control Flow Notes
Used during BSPC map compilation. `ParseEntity` is called in a loop until it returns `false`, populating the global `entities[]` array from a loaded BSP/map script. Accessor functions (`ValueForKey`, `SetKeyValue`, etc.) are called throughout the compilation pipeline to query entity properties (e.g., `classname`, `origin`, spawn flags). This file has no frame/update/render role — it is a tool-side compile-time component.

## External Dependencies
- `l_cmd.h` — `copystring`, `Error`, `qboolean`
- `l_mem.h` — `GetMemory`, `FreeMemory`
- `l_math.h` — `vec_t`, `vec3_t`
- `l_log.h` — (included but not directly called in this file)
- `botlib/l_script.h` — `script_t`, `token_t`, `PS_ReadToken`, `PS_ExpectAnyToken`, `PS_UnreadLastToken`, `StripDoubleQuotes`
- `l_bsp_ent.h` — defines `entity_t`, `epair_t`, `MAX_MAP_ENTITIES` (defined elsewhere)
- `printf`, `sscanf`, `strlen`, `strcmp`, `memset`, `atof` — C standard library
