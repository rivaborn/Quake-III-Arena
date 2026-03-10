# code/game/q_shared.c — Enhanced Analysis

## Architectural Role

This is a **universally-required stateless utility library** compiled into every executable module (all VMs, all engine subsystems). It acts as a foundational vocabulary that enables the VM sandbox model: because it contains no initialization code and no subsystem dependencies, it can be safely compiled into bytecode (QVM) or native code (DLL/SO) without creating circular dependencies or platform-coupling. It bridges the gap between system-layer concerns (byte order, filesystem paths, formatted printing) and the modular architecture where cgame, game, ui, and botlib must each operate in isolation while sharing a common string/parsing baseline.

## Key Cross-References

### Incoming (dependents)
- **All VMs** (`code/game`, `code/cgame`, `code/ui`, `code/q3_ui`, `code/botlib`) import functions from here for text parsing, string manipulation, info-string key/value lookup
- **qcommon subsystem** (`cmd.c`, `cvar.c`, `files.c`): calls `COM_Parse*` for tokenization of config files, console input, cvar definitions, and shader scripts
- **Renderer** (`code/renderer/tr_shader.c`, `tr_image.c`): parses `.shader` files and texture definitions via `COM_ParseExt`
- **Byte-swap functions**: called by file loaders (`code/botlib/be_aas_file.c` for AAS endianness, `code/renderer/tr_bsp.c` for BSP), enabling cross-platform binary compatibility
- **Server** (`code/server/sv_init.c`): parses map entity strings via `COM_Parse`; info-strings for client userinfo via `Info_ValueForKey`
- **Game VM** (`code/game/g_spawn.c`, `g_client.c`): parses entity spawn definitions and userinfo
- **Client** (`code/client/cl_parse.c`): processes configstring info-strings

### Outgoing (dependencies)
- **Com_Error** and **Com_Printf**: called for fatal errors and debug logging; provided by qcommon or VM trap layer (environment-dependent)
- **Standard C library** (`vsprintf`, `strncpy`, `strlen`, `strchr`, `strcmp`, etc.)
- **No other subsystems**: fully self-contained

## Design Patterns & Rationale

1. **Stateless but stateful parsing**: `COM_ParseExt` uses three static buffers (`com_token`, `com_parsename`, `com_lines`) for convenience, avoiding re-entrancy and multi-threaded access. This is a **portability trade-off**: simplicity for single-threaded console parsing vs. thread safety. The engine's single-threaded QVM design makes this acceptable.

2. **Alternating static buffers in va()**: Returns one of two 32 KB static buffers, alternating on each call. Permits up to 2 levels of nesting (e.g., `va(va(...), ...)`) but breaks beyond that. Classic pattern from early engines; modern engines use arena allocators.

3. **Direct byte-swap functions instead of function pointers**: Commented-out `Swap_Init` shows an old dispatch-table design; now replaced with compile-time or runtime selection. Direct function calls are faster and simpler in a modern architecture.

4. **Info-string `\key\value\` encoding**: Text-based, human-readable, but fragile — no escaping of `\` or `"` inside keys/values. Works because the engine uses disciplined key names (no special characters in practice).

5. **Hard size limits (MAX_TOKEN_CHARS, 32 KB stack in Com_sprintf)**: Simplifies buffer management but creates **crash vectors** if parsing untrusted input (e.g., malformed maps, overlong tokens).

## Data Flow Through This File

- **Parser flow**: Input text → `COM_ParseExt` → tokenized stream → consumers (qcommon cvar/cmd system, renderer shader parser, server entity loader)
- **Info-string flow**: Raw `\k1\v1\k2\v2\` → `Info_ValueForKey` (read) or `Info_SetValueForKey` (modify) → serialized back to configstrings
- **Byte-swap flow**: Binary file data (little-endian/big-endian) → `*Swap` primitives → native byte order (for BSP/AAS cross-platform loading)
- **Formatted printing**: User-provided format string + args → `Com_sprintf` → stack buffer → clamped copy to destination (with overflow logging)

## Learning Notes

1. **VM-architecture legacy**: This file encodes the design principle that QVM modules are stateless sandboxes. Com_Error and Com_Printf are the only "export points" back to the host. Compare to modern engines where utilities are more tightly integrated.

2. **Text parsing as a core competency**: Quake III parses shader definitions, map entity data, console scripts, and info-strings at runtime. The parser here is purposefully simple (no AST, no error recovery), favoring speed over diagnostics.

3. **Byte-order abstraction**: The byte-swap functions are a pre-C99 approach to architecture-independent binary formats. Modern engines would use serialization libraries (flatbuffers, protobuf) or enforce native endianness. Here, the abstraction is thin and explicit.

4. **Idiomatic alternating buffers pattern**: The `va()` function is famous in id Tech engines for enabling `printf(va(...))` syntax without explicit buffer allocation. It's a convenience anti-pattern by modern standards (limits nesting, poor cache locality), but it simplified call sites in the 1990s.

## Potential Issues

1. **Static parser state not re-entrant**: If `COM_Parse` were called from nested contexts (e.g., shader parser invoked during entity parse), `com_lines` would be corrupted, causing wrong error line numbers.

2. **Info-string overflow without clear indication**: `Info_SetValueForKey` silently truncates if the result exceeds `MAX_INFO_STRING` (512 bytes), but only prints a warning. Callers may not detect failure.

3. **Token size silently discarded**: `COM_ParseExt` discards tokens exceeding `MAX_TOKEN_CHARS` (1024 bytes) by resetting `len=0`, returning an empty token. This masks the error silently.

4. **Stack buffer overflow in Com_sprintf**: A 32 KB `vsprintf` call will trigger `Com_Error(ERR_FATAL)`, halting the entire engine. Malformed format strings or excessive arguments could crash the server.

5. **Byte-swap functions assume aligned access**: `FloatSwap` casts a `float*` to `_FloatByteUnion*` and reads `.i` (unsigned int). On strict-alignment architectures (ARM, SPARC), this could cause a bus error if the pointer is unaligned.
