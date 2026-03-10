# code/botlib/be_ai_char.c

## File Purpose
Implements the bot character system for Quake III Arena's botlib, loading and managing personality profiles (characteristics) from script files. Each bot character is a named collection of up to 80 typed key-value slots (integer, float, or string) associated with a skill level.

## Core Responsibilities
- Load bot character files from disk, parsing skill-bracketed blocks via the precompiler/script system
- Cache loaded characters in a global handle-indexed table to avoid redundant file I/O
- Apply default characteristics from a fallback character file when slots are uninitialized
- Interpolate numeric characteristics between two skill-level characters to produce fractional-skill variants
- Provide typed accessor functions (float, bounded float, integer, bounded integer, string) for game-side queries
- Free and shut down character resources, with optional reload-on-free behavior gated by a libvar

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cvalue` | union | Holds a single characteristic value as int, float, or char* |
| `bot_characteristic_t` | struct (typedef) | One characteristic slot: type tag + cvalue union |
| `bot_character_t` | struct (typedef) | A loaded character: filename, skill level, variable-length array of `bot_characteristic_t` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `botcharacters` | `bot_character_t *[MAX_CLIENTS + 1]` | global (file-scope array) | Handle-indexed table of all loaded bot characters; index 0 unused, 1–MAX_CLIENTS valid |

## Key Functions / Methods

### BotCharacterFromHandle
- **Signature:** `bot_character_t *BotCharacterFromHandle(int handle)`
- **Purpose:** Validates handle range and returns the corresponding character pointer.
- **Inputs:** `handle` — 1-based index into `botcharacters`
- **Outputs/Return:** Pointer to `bot_character_t`, or NULL on invalid handle/unloaded slot
- **Side effects:** Prints fatal error via `botimport.Print` on bad input
- **Calls:** `botimport.Print`

### BotLoadCharacterFromFile
- **Signature:** `bot_character_t *BotLoadCharacterFromFile(char *charfile, int skill)`
- **Purpose:** Parses a `.c` character script file, locating the block matching `skill` (or any block if skill == -1) and populating a freshly allocated `bot_character_t`.
- **Inputs:** `charfile` — path to script; `skill` — target skill integer, -1 = any
- **Outputs/Return:** Heap-allocated `bot_character_t *` on success, NULL on parse error or skill not found
- **Side effects:** Allocates memory (`GetClearedMemory`, `GetMemory`); frees on error paths; loads/frees a `source_t` via precompiler
- **Calls:** `PC_SetBaseFolder`, `LoadSourceFile`, `GetClearedMemory`, `PC_ReadToken`, `PC_ExpectTokenType`, `PC_ExpectTokenString`, `PC_ExpectAnyToken`, `SourceError`, `FreeSource`, `BotFreeCharacterStrings`, `FreeMemory`, `StripDoubleQuotes`, `GetMemory`

### BotLoadCachedCharacter
- **Signature:** `int BotLoadCachedCharacter(char *charfile, float skill, int reload)`
- **Purpose:** Multi-fallback loader: tries cache → exact skill file → default character file → any-skill variants, in priority order.
- **Inputs:** `charfile`, `skill`, `reload` — if non-zero, bypasses cache lookups
- **Outputs/Return:** Handle (1–MAX_CLIENTS) on success, 0 on failure
- **Side effects:** Writes to `botcharacters[handle]`; prints messages via `botimport.Print`
- **Calls:** `BotFindCachedCharacter`, `BotLoadCharacterFromFile`, `botimport.Print`

### BotLoadCharacter
- **Signature:** `int BotLoadCharacter(char *charfile, float skill)`
- **Purpose:** Public entry point — clamps skill to [1,5], loads exact skills 1/4/5 directly, or loads bracketing skills and interpolates for fractional values.
- **Inputs:** `charfile`, `skill`
- **Outputs/Return:** Character handle or 0
- **Side effects:** May allocate an interpolated character slot; dumps result to log
- **Calls:** `BotLoadCharacterSkill`, `BotFindCachedCharacter`, `BotInterpolateCharacters`, `BotDumpCharacter`

### BotInterpolateCharacters
- **Signature:** `int BotInterpolateCharacters(int handle1, int handle2, float desiredskill)`
- **Purpose:** Creates a new character by linearly interpolating float fields between two existing characters; integers and strings are copied from `handle1`.
- **Inputs:** Two character handles, target skill
- **Outputs/Return:** New handle or 0
- **Side effects:** Allocates a new `bot_character_t`; writes to `botcharacters`
- **Calls:** `BotCharacterFromHandle`, `GetClearedMemory`, `GetMemory`

### Characteristic_Float / Characteristic_BFloat / Characteristic_Integer / Characteristic_BInteger / Characteristic_String
- Typed accessors: resolve a character handle + index to a value with optional min/max clamping.
- All call `BotCharacterFromHandle` and `CheckCharacteristicIndex`; return 0/empty on error.
- `Characteristic_Float` coerces `CT_INTEGER` → float; `Characteristic_Integer` coerces `CT_FLOAT` → int via truncation.

### BotShutdownCharacters
- **Signature:** `void BotShutdownCharacters(void)`
- **Purpose:** Frees all loaded characters unconditionally (shutdown path).
- **Calls:** `BotFreeCharacter2` for each non-NULL slot

## Control Flow Notes
- **Init:** `BotLoadCharacter` is called per-bot at spawn time by the game module via `be_interface.c`.
- **Frame:** No per-frame update; characteristics are read on-demand via accessors.
- **Shutdown:** `BotShutdownCharacters` is called from the botlib shutdown path.
- `BotFreeCharacter` (the conditional version) respects the `bot_reloadcharacters` libvar, enabling hot-reload behavior during development.

## External Dependencies
- `q_shared.h` — core types, `MAX_CLIENTS`, `MAX_QPATH`, `qboolean`, string utilities
- `l_log.h` / `Log_Write` — debug character dump output
- `l_memory.h` / `GetMemory`, `GetClearedMemory`, `FreeMemory` — botlib heap allocator
- `l_script.h` / `l_precomp.h` — lexer and precompiler (`LoadSourceFile`, `PC_ReadToken`, etc.)
- `l_libvar.h` / `LibVarGetValue` — runtime variable lookup (`bot_reloadcharacters`)
- `be_interface.h` — `botimport` global (print/error callbacks into the engine)
- `be_ai_char.h` — public interface declarations (defined elsewhere, exported from this file)
- `Sys_MilliSeconds` — timing macro used in `#ifdef DEBUG` path (defined in platform layer)
