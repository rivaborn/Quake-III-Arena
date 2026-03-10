# code/game/g_utils.c

## File Purpose
Provides core utility functions for the Quake III Arena server-side game module, including entity lifecycle management (spawn, free, temp entities), entity search/targeting, event signaling, shader remapping, and miscellaneous math/string helpers.

## Core Responsibilities
- Entity allocation (`G_Spawn`), initialization (`G_InitGentity`), and deallocation (`G_FreeEntity`)
- Temporary event-entity creation (`G_TempEntity`)
- Entity search by field offset (`G_Find`) and random target selection (`G_PickTarget`)
- Target chain activation (`G_UseTargets`) and team-broadcast commands (`G_TeamCommand`)
- Game event attachment to entities (`G_AddEvent`, `G_AddPredictableEvent`)
- Shader remapping table management (`AddRemap`, `BuildShaderStateConfig`)
- Configstring index registration for models and sounds (`G_FindConfigstringIndex`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `shaderRemap_t` | struct | Holds an old/new shader name pair and a time offset for dynamic shader substitution |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `remapCount` | `int` | global (file) | Number of active shader remappings |
| `remappedShaders` | `shaderRemap_t[128]` | global (file) | Table of up to 128 shader remap entries |

## Key Functions / Methods

### AddRemap
- **Signature:** `void AddRemap(const char *oldShader, const char *newShader, float timeOffset)`
- **Purpose:** Inserts or updates an entry in the shader remap table.
- **Inputs:** Old shader name, new shader name, time offset.
- **Outputs/Return:** None.
- **Side effects:** Mutates `remappedShaders[]` and `remapCount`.
- **Calls:** `Q_stricmp`, `strcpy`
- **Notes:** Silently drops entries beyond `MAX_SHADER_REMAPS` (128).

### BuildShaderStateConfig
- **Signature:** `const char *BuildShaderStateConfig()`
- **Purpose:** Serializes the shader remap table into a configstring-compatible format.
- **Inputs:** None (reads global `remappedShaders`).
- **Outputs/Return:** Pointer to a static `char` buffer (`MAX_STRING_CHARS*4`).
- **Side effects:** Overwrites static buffer on each call; not re-entrant.
- **Calls:** `Com_sprintf`, `Q_strcat`

### G_FindConfigstringIndex
- **Signature:** `int G_FindConfigstringIndex(char *name, int start, int max, qboolean create)`
- **Purpose:** Finds or registers a named resource (model/sound) in the configstring table.
- **Inputs:** Resource name, configstring base index, max slot count, create flag.
- **Outputs/Return:** Slot index (1-based), or 0 if not found and `create` is false.
- **Side effects:** Calls `trap_SetConfigstring` when creating; calls `G_Error` on overflow.
- **Calls:** `trap_GetConfigstring`, `trap_SetConfigstring`, `G_Error`

### G_Find
- **Signature:** `gentity_t *G_Find(gentity_t *from, int fieldofs, const char *match)`
- **Purpose:** Linear scan of the active entity list matching a string field by byte offset.
- **Inputs:** Starting entity (NULL = beginning), field offset via `FOFS()`, match string.
- **Outputs/Return:** Next matching `gentity_t*`, or NULL.
- **Calls:** `Q_stricmp`
- **Notes:** Field offset pattern enables generic search without per-field functions.

### G_UseTargets
- **Signature:** `void G_UseTargets(gentity_t *ent, gentity_t *activator)`
- **Purpose:** Fires all entities whose `targetname` matches `ent->target`, and applies any shader remap.
- **Inputs:** Triggering entity, activating entity.
- **Side effects:** Calls each target's `.use` callback; may remap shaders via `trap_SetConfigstring`; guards against entity self-deletion mid-loop.
- **Calls:** `AddRemap`, `BuildShaderStateConfig`, `trap_SetConfigstring`, `G_Find`, `G_Printf`

### G_Spawn
- **Signature:** `gentity_t *G_Spawn(void)`
- **Purpose:** Allocates a free entity slot from `g_entities[]`, skipping recently-freed slots to avoid client interpolation artifacts.
- **Inputs:** None.
- **Outputs/Return:** Initialized `gentity_t*`.
- **Side effects:** May increment `level.num_entities` and call `trap_LocateGameData` to notify the server. Calls `G_Error` if no slots remain.
- **Calls:** `G_InitGentity`, `trap_LocateGameData`, `G_Error`, `G_Printf`
- **Notes:** Two-pass loop: first pass respects 1-second reuse cooldown; second pass forces reuse.

### G_FreeEntity
- **Signature:** `void G_FreeEntity(gentity_t *ed)`
- **Purpose:** Unlinks and zeroes an entity, marking it free.
- **Side effects:** Calls `trap_UnlinkEntity`; memsets the slot; respects `neverFree` flag.
- **Calls:** `trap_UnlinkEntity`, `memset`

### G_TempEntity
- **Signature:** `gentity_t *G_TempEntity(vec3_t origin, int event)`
- **Purpose:** Spawns a short-lived event entity that is auto-removed after the event fires.
- **Side effects:** Snaps origin to grid; links entity into world via `trap_LinkEntity`.
- **Calls:** `G_Spawn`, `SnapVector`, `G_SetOrigin`, `trap_LinkEntity`

### G_KillBox
- **Signature:** `void G_KillBox(gentity_t *ent)`
- **Purpose:** Kills all client entities occupying the bounding box of `ent` (telefrag).
- **Calls:** `trap_EntitiesInBox`, `G_Damage`

### G_AddEvent
- **Signature:** `void G_AddEvent(gentity_t *ent, int event, int eventParm)`
- **Purpose:** Attaches a networked event to an entity, toggling the event counter bit.
- **Side effects:** Modifies `playerState_t.externalEvent` (for clients) or `entityState_t.event` (for non-clients).
- **Notes:** Uses `EV_EVENT_BITS` rolling counter to prevent duplicate suppression.

### G_SetOrigin
- **Signature:** `void G_SetOrigin(gentity_t *ent, vec3_t origin)`
- **Purpose:** Sets both trajectory base and `currentOrigin` for a stationary entity.
- **Notes:** Sets `trType = TR_STATIONARY`.

### DebugLine
- **Signature:** `int DebugLine(vec3_t start, vec3_t end, int color)`
- **Purpose:** Draws a debug quad polygon between two points using the renderer's debug polygon API.
- **Calls:** `trap_DebugPolygonCreate`
- **Notes:** Only visible when `r_debugSurface 2` is set in a local game.

## Control Flow Notes
This file is a support module with no frame loop of its own. Functions are called throughout the game frame:
- `G_Spawn` / `G_FreeEntity` during map load and entity lifecycle events.
- `G_UseTargets` from trigger/mover callbacks each frame.
- `G_TempEntity` / `G_AddEvent` from combat and pickup code each frame.
- `G_KillBox` during player teleportation (client spawn path).

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (`level_locals_t`), all `trap_*` syscall stubs, `G_Damage`, `BG_AddPredictableEventToPlayerstate`, `AngleVectors`, `VectorCompare`, `Com_sprintf`, `Q_stricmp`, `Q_strcat`, `SnapVector`
