# code/botlib/l_struct.h

## File Purpose
Defines a generic, data-driven framework for reading and writing arbitrary C structs from/to script files and disk. Field descriptors encode name, offset, type, and constraints, enabling reflection-like serialization of botlib configuration structures.

## Core Responsibilities
- Define field type constants (`FT_CHAR`, `FT_INT`, `FT_FLOAT`, `FT_STRING`, `FT_STRUCT`) and subtype modifier flags (`FT_ARRAY`, `FT_BOUNDED`, `FT_UNSIGNED`)
- Provide `fielddef_t` to describe a single field within a struct (name, byte offset, type info, bounds, nested struct pointer)
- Provide `structdef_t` to describe a complete struct (size + field array)
- Declare `ReadStructure` for deserializing a struct from a parsed script token stream
- Declare `WriteStructure` for serializing a struct to a `FILE*`
- Declare utility formatters `WriteIndent` and `WriteFloat`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `fielddef_t` | struct (`fielddef_s`) | Describes one field: name, byte offset into parent struct, type flags, array max, float bounds, optional nested `structdef_t*` |
| `structdef_t` | struct (`structdef_s`) | Top-level struct descriptor: total size in bytes and pointer to `fielddef_t` array |

## Global / File-Static State
None.

## Key Functions / Methods

### ReadStructure
- **Signature:** `int ReadStructure(source_t *source, structdef_t *def, char *structure);`
- **Purpose:** Parses a script token stream (`source_t`) and populates the raw memory block `structure` according to `def`, dispatching by field type.
- **Inputs:** `source` — active script parser state; `def` — struct schema; `structure` — raw destination buffer
- **Outputs/Return:** Non-zero on success, 0 on parse error (inferred from botlib conventions)
- **Side effects:** Writes decoded field values directly into `structure` at offsets from `def->fields`; may advance script parser state
- **Calls:** Defined in `l_struct.c`; calls into `l_script.c` token reading routines internally
- **Notes:** Uses `fielddef_t.offset` for direct memory writes — caller must pass a correctly-sized buffer matching `def->size`

### WriteStructure
- **Signature:** `int WriteStructure(FILE *fp, structdef_t *def, char *structure);`
- **Purpose:** Iterates over all fields in `def`, reading values from `structure` and writing formatted text to `fp`.
- **Inputs:** `fp` — output file handle; `def` — struct schema; `structure` — raw source buffer
- **Outputs/Return:** Non-zero on success
- **Side effects:** File I/O to `fp`; uses `WriteIndent` and `WriteFloat` for formatting
- **Calls:** `WriteIndent`, `WriteFloat`; defined in `l_struct.c`

### WriteIndent / WriteFloat
- **Notes:** Trivial formatting helpers. `WriteIndent` emits tab/space indentation; `WriteFloat` prints a float suppressing unnecessary trailing zeros.

## Control Flow Notes
This header is a utility module with no frame/update lifecycle. It is used during **initialization** (loading bot AI config files via `ReadStructure`) and optionally during **shutdown or config export** (`WriteStructure`). `source_t` comes from `l_script.h`/`l_precomp.h` — callers first open a script file, then pass the resulting `source_t` to `ReadStructure`.

## External Dependencies
- `source_t` — defined in `code/botlib/l_script.h` (script parser state)
- `struct structdef_s` — self-referential forward declaration within `fielddef_t.substruct` to support nested struct recursion
- `FILE*` — standard C `<stdio.h>`
- Implementation: `code/botlib/l_struct.c`
