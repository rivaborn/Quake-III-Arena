# code/game/g_spawn.c — Enhanced Analysis

## Architectural Role

`g_spawn.c` is the **game VM's bootstrapping kernel**: it sits at the exact boundary between the static BSP map file (produced by `q3map/`) and the live, running entity simulation managed by the rest of `code/game/`. It transforms the raw text entity string embedded in the BSP into fully initialized, world-linked `gentity_t` instances. It is invoked exactly once per map load by `g_main.c`'s `G_InitGame`, before the first simulation frame runs. The file also owns `SP_worldspawn`, making it the point where server-visible configstrings (gravity, music, MOTD, game version, level start time) are pushed to the client layer via `trap_SetConfigstring` — a cross-VM data flow touching the server snapshot system.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/game/g_main.c`** — calls `G_SpawnEntitiesFromString` during `G_InitGame`; this is the sole top-level caller.
- **All `SP_*` spawn functions** (in `g_misc.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_items.c`, `g_team.c`, `g_arenas.c`) — call back into `G_SpawnString`, `G_SpawnFloat`, `G_SpawnInt`, `G_SpawnVector` to read their own per-entity key/value pairs during their own spawn execution. This means `level.spawnVars` must remain valid across the entire nested call chain: `G_SpawnEntitiesFromString → G_SpawnGEntityFromSpawnVars → G_CallSpawn → SP_* → G_SpawnString`.
- **`code/game/g_items.c`** — `G_SpawnItem` is invoked from `G_CallSpawn` for any entity whose classname matches `bg_itemlist`; item spawn is effectively a first-class parallel path in the dispatch table.
- **`code/game/g_bot.c`** — reads entities populated during spawn (e.g., `info_player_start` locations) to place bots post-spawn.

### Outgoing (what this file depends on)

- **`code/game/g_utils.c`** — `G_Spawn`, `G_FreeEntity`, `G_Error`, `G_Printf` for entity lifecycle management.
- **`code/game/g_mem.c`** — `G_Alloc` for level-scoped string memory; these allocations live until `G_ShutdownGame`.
- **`code/game/bg_misc.c`** (shared bg layer) — `bg_itemlist` for item classname resolution in `G_CallSpawn`.
- **Engine trap syscalls** — `trap_GetEntityToken` to consume the BSP entity string token-by-token; `trap_SetConfigstring` to broadcast worldspawn data to all clients; `trap_Cvar_Set` to override gravity.
- **`q_shared.c`** — `Q_stricmp` for case-insensitive key matching.
- **All `SP_*` functions** — declared by forward-reference here and defined across `g_misc.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_items.c`, `g_team.c`, `g_arenas.c`. The `spawns[]` table is the only location in the codebase that enumerates the complete set of server-side spawnable entity types.

## Design Patterns & Rationale

- **Reflection via offset table (`fields[]`):** Rather than a giant `if/else` chain per field name, `G_ParseField` walks a static table of `{name, offset, type}` records and writes directly into `gentity_t` at the byte offset using casts. This is a common late-1990s C pattern for data-driven serialization, equivalent to what modern engines do with reflection metadata or protobuf schemas. The tradeoff: it is brittle to `gentity_t` layout changes and cannot handle computed or aliased fields (hence the `F_IGNORE` escape hatch for `light`).

- **Two-phase dispatch (`G_CallSpawn`):** Items are resolved first through `bg_itemlist` (shared between game and cgame), then through the `spawns[]` table. This separation ensures that item spawning logic (which must be symmetric with cgame for prediction purposes) is handled by `g_items.c` rather than duplicated in the dispatch table.

- **Gametype filtering at spawn time:** The `notsingle`/`notteam`/`notfree`/`notq3a`/`notta`/`gametype` flags are evaluated in `G_SpawnGEntityFromSpawnVars` before `G_CallSpawn`. This means spawn functions themselves never need to be gametype-aware — the filtering is entirely handled in the bootstrapper. The `#ifdef MISSIONPACK` gates mirror the compile-time product split between Q3A and Team Arena.

- **Temporary token-buffer pattern:** `level.spawnVarChars` acts as a linear arena for one entity's worth of key/value string data. Pointers into it (`level.spawnVars[i][1]`) are only valid until `G_ParseSpawnVars` is called again for the next entity. This is why `G_SpawnString` returns a raw pointer that callers must copy if they need durable storage — and why `G_ParseField` runs `G_NewString` to make a permanent copy for `F_LSTRING` fields.

## Data Flow Through This File

```
BSP entity string (engine memory)
         │
         ▼  trap_GetEntityToken (one token at a time)
G_ParseSpawnVars → level.spawnVars[]   (temporary char arena)
         │
         ▼  G_SpawnGEntityFromSpawnVars
         ├─ G_Spawn() → fresh gentity_t
         ├─ G_ParseField() × N   ─→ writes typed values to gentity_t fields via offset table
         ├─ gametype filter checks (G_SpawnInt for flags) ─→ may G_FreeEntity + return
         ├─ VectorCopy: s.origin → s.pos.trBase, r.currentOrigin
         └─ G_CallSpawn()
              ├─ bg_itemlist lookup → G_SpawnItem (g_items.c)
              └─ spawns[] lookup → SP_*(ent) (g_misc/mover/trigger/target/team/arenas.c)
                       │
                       └─ SP_* may call G_SpawnString/Float/Int/Vector
                          (reads back from level.spawnVars[], still valid)

worldspawn path:
SP_worldspawn → trap_SetConfigstring (CS_GAME_VERSION, CS_LEVEL_START_TIME, CS_MUSIC, etc.)
              → trap_Cvar_Set (g_gravity, sv_fps)
              → level.warmupTime
```

State transitions: `level.spawning` is `qfalse` before and after this file runs. It is set `qtrue` only for the duration of `G_SpawnEntitiesFromString`. The commented-out `G_Error` in `G_SpawnString` (for calls outside spawning) hints that this invariant was once enforced and was later relaxed — callers like `SP_*` functions (which read spawn vars late in their execution) need the window to remain open.

## Learning Notes

- **Idiomatic Q3-era C reflection:** The `FOFS(field)` macro expands to `offsetof(gentity_t, field)`, giving a compile-safe offset without C++ RTTI. Modern engines use generated code (via IDL/protobuf/reflection libraries) or language-level reflection, but the conceptual pattern is identical.

- **`F_GSTRING`, `F_ENTITY`, `F_ITEM`, `F_CLIENT` are dead code:** These fieldtype enum values exist but have no `case` in `G_ParseField`'s switch — they fall through to `F_IGNORE`. This suggests the enum was designed for a more general serialization system (possibly shared with save-game or network state) that was never fully implemented in this file.

- **`F_ANGLEHACK` reveals Quake history:** The `angle` key (a single scalar) predates Quake's 3-component `angles`. The hack converts it to yaw-only (`s.angles[1]`), preserving backward compatibility with entity definitions from earlier Quake games.

- **`SP_item_botroam` is a no-op stub** defined inline (`{}`): it exists only to consume the classname so `G_CallSpawn` doesn't print an "unknown classname" warning. The actual bot-roam hint logic lives in `be_ai_goal.c` in botlib.

- **Spawn functions are never called at frame time:** All entity initialization from map data happens before frame 0. This is a hard guarantee enforced by `level.spawning`. Compare to Unity/Unreal where `Start`/`BeginPlay` can be deferred or called mid-frame.

- **No ECS here:** Q3 uses a flat, fixed-size `gentity_t` pool (`g_entities[MAX_GENTITIES]`) with a rich union-like struct and nullable function pointers (`think`, `use`, `touch`, `pain`, `die`). This predates ECS; the spawn system populates the struct directly rather than attaching components.

## Potential Issues

- **`strstr` gametype matching is fragile:** In `G_SpawnGEntityFromSpawnVars`, `strstr(value, gametypeName)` will false-positive if a gametype name is a substring of another (e.g., `"team"` matching inside `"teamtournament"`). Since `gametypeNames[]` lists both, an entity with `gametype "team"` would incorrectly spawn in `teamtournament` matches.
- **`G_SpawnString` called outside spawning silently uses default:** The original error call is commented out; callers in later code that accidentally invoke `G_SpawnString` post-spawn will silently receive the default value with no diagnostic — a subtle debugging hazard.
- **`sscanf` in `G_SpawnVector`/`G_ParseField` has no error checking:** A malformed vector in the BSP entity string will silently produce zeroes or partial values.
