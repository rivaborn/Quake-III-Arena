# code/ui/ui_gameinfo.c — Enhanced Analysis

## Architectural Role

This file is the **data-loading bridge** for the UI subsystem, responsible for populating two critical UI resources at startup: the map list (for single-player and server creation) and the bot registry (for bot selections and team composition). It sits at the intersection of the **virtual filesystem** (via `trap_FS_*` syscalls) and the **UI data model** (`uiInfo.mapList` and static bot arrays), abstracting filesystem/parsing complexity away from the menu logic layer. All work occurs during UI initialization; there is no per-frame involvement.

## Key Cross-References

### Incoming (who depends on this file)
- **`UI_Load()` / `UI_InitGameinfo()`** — engine-supplied entry points that call `UI_LoadArenas()` and `UI_LoadBots()` during UI VM startup
- **Menu system** — calls `UI_GetBotInfoByNumber()`, `UI_GetBotInfoByName()`, and `UI_GetBotNameByNumber()` when populating bot selection menus and match setup screens
- **`uiInfo` global** — populated by `UI_LoadArenas()` with `mapList[]` and `mapCount`, consumed by map selection menus and server creation dialogs
- **Cvar system** — reads `g_arenasFile` / `g_botsFile` to allow modders to override default `.txt` paths without code changes

### Outgoing (what this file depends on)
- **VFS layer** (`trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `trap_FS_GetFileList`) — all file I/O is abstracted via trap syscalls; never touches the native filesystem directly (enforced by QVM sandbox)
- **Common parsing utilities** (`COM_Parse`, `COM_ParseExt`, `COM_Compress`, `Info_SetValueForKey`, `Info_ValueForKey`) — shared across qcommon, reducing code duplication
- **String utilities** (`Q_strncpyz`, `Q_stricmp`, `va()`) — from `q_shared.c`
- **Memory allocators** (`UI_Alloc`, `String_Alloc`) — zone-based pool allocators that prevent fragmentation in a QVM context
- **Debug/error** (`trap_Print`, `Com_Printf`) — console output for load diagnostics and warnings

## Design Patterns & Rationale

1. **Two-tier info string design**: Raw text parsed into opaque key-value strings (using Q3's `.info` format), then extracted on-demand with `Info_ValueForKey()`. This allows:
   - Lazy value extraction (only read fields when needed)
   - Easy addition of new bot/arena properties without code recompilation
   - Direct modder editability of plain-text files

2. **Wildcard + primary file scanning**: Loads a default file (`scripts/arenas.txt`) plus all matching `*.arena` files. This pattern:
   - Preserves backward compatibility (base game data in primary file)
   - Enables modular content addition (each mod can drop its own `.arena` files)
   - Avoids hardcoded content lists

3. **Cvar-driven overrides** (`g_arenasFile`, `g_botsFile`): Allows runtime path remapping without recompilation—critical for:
   - Testing alternate content sets
   - Supporting multiple game modes (base Q3A vs. mods)
   - Server-side customization without rebuilding the client

4. **Game-type bitfield encoding** (lines ~195–210): Each map stores a bitmask of compatible game modes (`(1 << GT_FFA)`, etc.), avoiding string-based game-type lookup in hot paths (menu filtering, server creation).

5. **Stateless accessor functions**: `UI_GetBotInfoByNumber()` and `UI_GetBotInfoByName()` are pure lookups—no side effects, no state mutations. This allows callers to safely cache the returned pointer until the UI is reinitialized.

## Data Flow Through This File

1. **Load phase** (once at UI startup):
   - `UI_LoadArenas()` opens primary arena file (or override via `g_arenasFile`)
   - Scans `scripts/` for `*.arena` files via VFS directory listing
   - Calls `UI_ParseInfos()` for each file → allocates heap strings via `UI_Alloc`
   - Populates `ui_arenaInfos[]` (raw info strings) and `ui_numArenas`
   - Iterates parsed arenas, extracts `map`, `longname`, `type` keys
   - Populates `uiInfo.mapList[]` with:
     - `mapLoadName` (BSP filename from `map=` key)
     - `mapName` (human-readable from `longname=` key)
     - `imageName` (derived path `levelshots/{mapLoadName}`)
     - `typeBits` (bitmask from substring-matching `type=` value)
   - Similarly for bots: load into `ui_botInfos[]` for later lookup

2. **Lookup phase** (during menu interaction):
   - Menu code calls `UI_GetBotInfoByNumber(n)` → O(1) array access
   - Menu code calls `UI_GetBotInfoByName(str)` → O(n) linear search, extract `name=` key
   - Returns opaque info string; caller uses `Info_ValueForKey(info, "key")` to extract fields
   - No re-parsing occurs; everything is cached in `ui_arenaInfos[]` and `ui_botInfos[]`

3. **Termination**: Static arrays persist for the UI VM's entire lifetime; no explicit cleanup needed (VM is destroyed on map load or client disconnect).

## Learning Notes

**What studying this file teaches:**

- **Q3's lightweight info format**: An alternative to JSON/XML—just `\key\value\key2\value2\...` with escape sequences. Enables fast parsing and easy modding.
- **VFS abstraction power**: All filesystem access is via trap syscalls, enabling:
  - Pure server validation (PK3 checksums, pure server mode)
  - Seamless `.pk3` archive handling (ZIP transparently merged with directories)
  - Memory-mapped or compressed asset loading (engine can swap implementations)
  - Separation of concerns (UI code never knows about platform details)
- **One-time load patterns in VMs**: Since QVMs are sandboxed and stateless (reinitialized on each map or after a disconnect), initialization costs are amortized. This file's O(n) startup cost is acceptable.
- **Modular content via scanning**: Rather than hardcoding content lists, scanning `scripts/` allows mods to add content by simply dropping files—a key Q3 extensibility mechanism.

**How modern engines differ:**

- **Asset pipelines**: Modern engines (UE4, Unity) cook/serialize assets offline, then load binary formats; Q3 parses text at runtime.
- **Schema validation**: Modern engines validate against schemas; Q3 silently ignores unknown keys (forward-compatible by default).
- **Lazy loading**: Modern engines stream/lazy-load assets; Q3 loads everything at map startup.
- **Type safety**: Modern engines use strongly-typed asset classes; Q3 treats everything as opaque key-value strings until extracted.

Despite these differences, Q3's approach proves **robust and highly moddable**—testament to its design clarity.

## Potential Issues

1. **Buffer overflow in loop** (line ~163): The pattern `for (...; dirptr += dirlen+1)` relies on `dirlist` being properly null-terminated. If `trap_FS_GetFileList()` returns a malformed buffer, reading past the end is possible. No assertion guards this.

2. **Unused `\num\` suffix** (line ~83): Comment mentions allocating extra space for `\num\<index>`, but the code never populates this field. Either dead code or incomplete feature.

3. **Brittle game-type detection** (lines ~195–210): Substring matching (`strstr(type, "ffa")`) fails if:
   - Type string has different case (`"FFA"` vs. `"ffa"`)
   - Type string is a superset containing partial matches (`"funfair"` would match `"ffa"`)
   - No case-insensitive variant is used (unlike `Q_stricmp` elsewhere)

4. **No fallback for `UI_OutOfMemory()`** (line ~165): After loading, if the memory pool is exhausted, the code prints a warning but continues. Subsequent allocations will fail silently, causing undefined behavior. Should either fail-fast or skip remaining arenas.

5. **Linear search in `UI_GetBotInfoByName()`** (line ~299): O(n) search on every call. For a menu that might query multiple bots per frame, consider caching the result or building a hash table. (Unlikely to be a real bottleneck, but idiomatic performance issue.)

6. **No duplicate detection**: If `scripts/arenas.txt` and `scripts/custom.arena` both define a map with the same name, both are loaded and `uiInfo.mapCount` increments. The UI menu would show duplicates unless deduped elsewhere.

7. **Implicit type default** (line ~202): If `type` is empty/missing, maps silently default to FFA. No warning or logging; modders might not realize their intent isn't respected.
