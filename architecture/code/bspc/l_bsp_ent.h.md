# code/bspc/l_bsp_ent.h

## File Purpose
Declares the entity and key-value pair data structures used during BSP map parsing and AAS compilation in the BSPC tool. It provides the interface for reading, writing, and querying entity key-value properties parsed from Quake III BSP/map source files.

## Core Responsibilities
- Define the `epair_t` linked-list node for storing entity key-value string pairs
- Define the `entity_t` aggregate representing a parsed map entity (brushes, origin, portals, etc.)
- Expose the global entity array and entity count used during BSP processing
- Declare parsing functions for deserializing entities and epairs from a script token stream
- Declare accessors for typed key-value lookups (string, float, vector)
- Declare mutation function `SetKeyValue` for writing entity properties

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `epair_t` | struct (typedef) | Singly-linked list node holding one key-value string pair for an entity |
| `entity_t` | struct (typedef, anonymous) | Represents a parsed map entity: spatial data, brush range, epair list, areaportal info, and flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `num_entities` | `int` | global (extern) | Count of parsed entities stored in the `entities` array |
| `entities` | `entity_t[MAX_MAP_ENTITIES]` | global (extern) | Flat array holding all parsed map entities (max 2048) |

## Key Functions / Methods

### StripTrailing
- **Signature:** `void StripTrailing(char *e)`
- **Purpose:** Removes trailing whitespace from a string in-place
- **Inputs:** Mutable C string `e`
- **Outputs/Return:** void; modifies `e` in place
- **Side effects:** Writes null terminator into `e`
- **Calls:** Not inferable from this file
- **Notes:** Utility used to sanitize parsed token strings

### SetKeyValue
- **Signature:** `void SetKeyValue(entity_t *ent, char *key, char *value)`
- **Purpose:** Inserts or updates a key-value pair in an entity's epair list
- **Inputs:** Target entity, key string, value string
- **Outputs/Return:** void; mutates `ent->epairs`
- **Side effects:** May allocate memory for a new `epair_t` node
- **Calls:** Not inferable from this file

### ValueForKey
- **Signature:** `char *ValueForKey(entity_t *ent, char *key)`
- **Purpose:** Walks the entity's epair list and returns the value string for the given key
- **Inputs:** Entity pointer, key string
- **Outputs/Return:** Pointer to value string, or `""` if key is absent
- **Side effects:** None
- **Notes:** Guaranteed non-NULL return; callers need not null-check

### FloatForKey
- **Signature:** `vec_t FloatForKey(entity_t *ent, char *key)`
- **Purpose:** Returns the numeric (float) interpretation of a key's value
- **Inputs:** Entity pointer, key string
- **Outputs/Return:** `vec_t` (float) parsed from the value string; 0.0 if absent
- **Side effects:** None

### GetVectorForKey
- **Signature:** `void GetVectorForKey(entity_t *ent, char *key, vec3_t vec)`
- **Purpose:** Parses a space-separated triple from a key's value into a `vec3_t`
- **Inputs:** Entity pointer, key string, output vector
- **Outputs/Return:** void; writes into `vec`
- **Side effects:** None

### ParseEntity
- **Signature:** `qboolean ParseEntity(script_t *script)`
- **Purpose:** Parses one complete `{ … }` entity block from the script token stream into `entities[num_entities]`
- **Inputs:** Active `script_t` token stream
- **Outputs/Return:** `qtrue` if an entity was parsed, `qfalse` at end-of-file
- **Side effects:** Increments `num_entities`; allocates `epair_t` nodes; writes into global `entities` array
- **Calls:** `ParseEpair`

### ParseEpair
- **Signature:** `epair_t *ParseEpair(script_t *script)`
- **Purpose:** Parses a single `"key" "value"` token pair from the script stream
- **Inputs:** Active `script_t` token stream
- **Outputs/Return:** Heap-allocated `epair_t *`
- **Side effects:** Allocates memory for the node and copies key/value strings

### PrintEntity
- **Signature:** `void PrintEntity(entity_t *ent)`
- **Purpose:** Debug-prints all key-value pairs of an entity to stdout/log
- **Inputs:** Entity pointer
- **Outputs/Return:** void
- **Side effects:** I/O

## Control Flow Notes
This header is consumed during BSPC's BSP-to-AAS pipeline. During map loading, `ParseEntity` is called repeatedly until it returns `qfalse`, populating the global `entities[]` array. Downstream AAS area generation code then queries entities via `ValueForKey`/`FloatForKey`/`GetVectorForKey` to extract spatial and classification data (e.g., func_areaportals, origins).

## External Dependencies
- `vec3_t`, `vec_t`, `qboolean` — defined in `q_shared.h` / math headers
- `script_t` — defined in `l_script.h` (BSPC script/tokenizer subsystem)
- `MAX_MAP_ENTITIES` — guard-defined in this header (2048) if not already defined
- `StripTrailing`, `SetKeyValue`, etc. — implemented in `l_bsp_ent.c` (defined elsewhere)
