# code/game/g_spawn.c

## File Purpose
Parses the map's entity string at level load time, translates key/value spawn variables into binary `gentity_t` fields, and dispatches each entity to its class-specific spawn function. It is the entry point for all server-side entity instantiation from BSP data.

## Core Responsibilities
- Read and store raw key/value token pairs from the BSP entity string (`G_ParseSpawnVars`)
- Provide typed accessors for spawn variables: string, float, int, vector (`G_SpawnString`, etc.)
- Map string field names to `gentity_t` struct offsets and write typed values (`G_ParseField`)
- Look up and invoke the correct spawn function by classname (`G_CallSpawn`)
- Process the `worldspawn` entity to apply global level settings (`SP_worldspawn`)
- Filter entities by gametype flags (`notsingle`, `notteam`, `notfree`, `notq3a`/`notta`, `gametype`)
- Drive the full entity spawning loop for an entire level (`G_SpawnEntitiesFromString`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `fieldtype_t` | enum | Discriminates how a spawn key value is decoded (int, float, string, vector, angle hack, ignore) |
| `field_t` | struct | Maps a string field name to a `gentity_t` byte offset and `fieldtype_t` for reflection-style field writing |
| `spawn_t` | struct | Associates a classname string with a `void (*spawn)(gentity_t*)` function pointer |
| `fields[]` | `field_t[]` (file-static array) | Complete reflection table of spawnable `gentity_t` fields |
| `spawns[]` | `spawn_t[]` (file-static array) | Dispatch table mapping all known entity classnames to their spawn functions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `fields[]` | `field_t[]` | file-static | Reflection table used by `G_ParseField` to write entity fields by name |
| `spawns[]` | `spawn_t[]` | file-static | Classname-to-function dispatch table used by `G_CallSpawn` |

## Key Functions / Methods

### G_SpawnString
- **Signature:** `qboolean G_SpawnString( const char *key, const char *defaultString, char **out )`
- **Purpose:** Looks up a key in the current `level.spawnVars[]` table; returns a pointer to the value or the default.
- **Inputs:** Key name, default string, output pointer.
- **Outputs/Return:** `qtrue` if found; `*out` set to matching value or default.
- **Side effects:** None (read-only access to `level.spawnVars`).
- **Calls:** `Q_stricmp`
- **Notes:** Returns a temporary pointer into `level.spawnVarChars`; callers must `CopyString()` if they need to retain the value. Safe to call outside spawning (silently uses default).

### G_SpawnFloat / G_SpawnInt / G_SpawnVector
- **Signature:** `qboolean G_SpawnFloat/Int/Vector( const char *key, const char *defaultString, [float|int|float*] *out )`
- **Purpose:** Typed wrappers over `G_SpawnString` that parse the value via `atof`/`atoi`/`sscanf`.
- **Notes:** All three delegate to `G_SpawnString` and convert in place.

### G_CallSpawn
- **Signature:** `qboolean G_CallSpawn( gentity_t *ent )`
- **Purpose:** Resolves `ent->classname` to a spawn function; first checks `bg_itemlist` for items, then the `spawns[]` table for all other entity types.
- **Inputs:** Partially initialized `gentity_t` with classname set.
- **Outputs/Return:** `qtrue` on success; prints a warning and returns `qfalse` if classname is unknown.
- **Side effects:** Calls the matched spawn function, which fully initializes `ent` and may link it into the world.
- **Calls:** `G_SpawnItem`, `s->spawn(ent)`, `G_Printf`, `strcmp`

### G_NewString
- **Signature:** `char *G_NewString( const char *string )`
- **Purpose:** Allocates a level-tagged copy of a string, translating `\n` escape sequences to real newlines.
- **Inputs:** Source string (from entity data).
- **Outputs/Return:** Pointer to newly allocated string in level memory.
- **Side effects:** Calls `G_Alloc` (level-scoped heap allocation).
- **Calls:** `G_Alloc`, `strlen`

### G_ParseField
- **Signature:** `void G_ParseField( const char *key, const char *value, gentity_t *ent )`
- **Purpose:** Reflection write: locates the matching `field_t` entry and writes the parsed value directly into the `gentity_t` at the recorded byte offset.
- **Inputs:** Key/value strings, target entity.
- **Side effects:** Writes into `ent` fields; allocates string memory via `G_NewString` for `F_LSTRING` fields.
- **Calls:** `Q_stricmp`, `G_NewString`, `sscanf`, `atoi`, `atof`
- **Notes:** `F_ANGLEHACK` collapses a scalar angle into `s.angles[1]` only (yaw). `F_GSTRING` and `F_ENTITY`/`F_ITEM`/`F_CLIENT` are declared in the enum but have no `case` handler — they fall through to `F_IGNORE`.

### G_SpawnGEntityFromSpawnVars
- **Signature:** `void G_SpawnGEntityFromSpawnVars( void )`
- **Purpose:** Allocates a new `gentity_t`, applies all current spawn vars via `G_ParseField`, filters by gametype flags, then calls `G_CallSpawn`.
- **Side effects:** May call `G_FreeEntity` if filtered out or spawn fails; sets `s.pos.trBase` and `r.currentOrigin` from `s.origin`.
- **Calls:** `G_Spawn`, `G_ParseField`, `G_SpawnInt`, `G_SpawnString`, `G_FreeEntity`, `G_CallSpawn`, `VectorCopy`, `strstr`

### G_ParseSpawnVars
- **Signature:** `qboolean G_ParseSpawnVars( void )`
- **Purpose:** Reads one `{ key value ... }` block from the engine's entity token stream into `level.spawnVars[]` / `level.spawnVarChars`.
- **Outputs/Return:** `qfalse` when the token stream is exhausted (end of entity string).
- **Side effects:** Writes to `level.numSpawnVars`, `level.spawnVars`, `level.numSpawnVarChars`, `level.spawnVarChars`.
- **Calls:** `trap_GetEntityToken`, `G_AddSpawnVarToken`, `G_Error`

### SP_worldspawn
- **Signature:** `void SP_worldspawn( void )`
- **Purpose:** Handles the mandatory first entity; sets configstrings for game version, level start time, music, MOTD, gravity, dust/breath effects; initializes warmup state.
- **Side effects:** Calls `trap_SetConfigstring`, `trap_Cvar_Set`, `G_LogPrintf`; writes to `g_entities[ENTITYNUM_WORLD]` and `level.warmupTime`.
- **Calls:** `G_SpawnString`, `trap_SetConfigstring`, `trap_Cvar_Set`, `G_LogPrintf`, `Q_stricmp`, `G_Error`

### G_SpawnEntitiesFromString
- **Signature:** `void G_SpawnEntitiesFromString( void )`
- **Purpose:** Top-level entry point; sets `level.spawning = qtrue`, processes `worldspawn`, then loops spawning all remaining entities; clears `level.spawning` on exit.
- **Side effects:** Fully populates the entity array for the current level.
- **Calls:** `G_ParseSpawnVars`, `SP_worldspawn`, `G_SpawnGEntityFromSpawnVars`, `G_Error`

## Control Flow Notes
`G_SpawnEntitiesFromString` is called once during level initialization (from `g_main.c`'s map-load path). It runs entirely before the first game frame. After it returns, `level.spawning` is `qfalse` and all map entities are live. No spawn functions are called during normal frame updates.

## External Dependencies
- `g_local.h` — `gentity_t`, `level_locals_t`, `FOFS`, all `g_*` cvars, all `trap_*` syscalls
- `bg_public.h` (via `g_local.h`) — `bg_itemlist`, `gitem_t`, gametype constants (`GT_*`)
- **Defined elsewhere:** `G_Spawn`, `G_FreeEntity`, `G_Alloc`, `G_SpawnItem`, `G_Error`, `G_Printf`, `G_LogPrintf`, `trap_GetEntityToken`, `trap_SetConfigstring`, `trap_Cvar_Set`, `Q_stricmp`, all `SP_*` spawn functions (defined in `g_misc.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_items.c`, etc.)
