# code/game/q_shared.h

## File Purpose
The universal shared header included first by all Quake III Arena program modules (game, cgame, UI, botlib, renderer, and tools). It defines the engine's foundational type system, math library, string utilities, network-communicated data structures, and cross-platform portability layer. Mod authors must never modify this file.

## Core Responsibilities
- Cross-platform portability: compiler warnings, CPU detection, `QDECL`, `ID_INLINE`, `PATH_SEP`, byte-order swap functions
- Primitive type aliases (`byte`, `qboolean`, `qhandle_t`, `vec_t`, `vec3_t`, etc.)
- Math library: vector/angle/matrix macros and inline functions, `Q_rsqrt`, `Q_fabs`, bounding-box helpers
- String utilities: `Q_stricmp`, `Q_strncpyz`, color-sequence stripping, `va()`, `Com_sprintf`
- Engine data structures communicated over the network: `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `gameState_t`
- Cvar system interface: `cvar_t`, `vmCvar_t`, and all `CVAR_*` flag bits
- Collision primitives: `cplane_t`, `trace_t`, `markFragment_t`
- Info-string key/value API declarations
- VM compatibility: conditionally includes `bg_lib.h` instead of standard C headers when compiled for the Q3 virtual machine

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `qboolean` | enum | Engine boolean (`qfalse`/`qtrue`) |
| `vec3_t` / `vec4_t` | typedef (array) | Primary vector types for 3-D/4-D math |
| `cplane_t` | struct | Collision plane with normal, distance, type, signbits |
| `trace_t` | struct | Result of a swept-box collision query |
| `playerState_t` | struct | Full client-predicted player state sent client↔server |
| `entityState_t` | struct | Minimal per-entity state delta-compressed over network |
| `usercmd_t` | struct | Client input command sent to server each frame |
| `trajectory_t` | struct | Parametric motion description (linear, sine, gravity, etc.) |
| `cvar_t` | struct | Console variable node in the cvar linked list |
| `vmCvar_t` | struct | VM-safe snapshot of a cvar (handle + value copy) |
| `gameState_t` | struct | All config strings delivered to clients at connect |
| `glyphInfo_t` / `fontInfo_t` | struct | Font glyph atlas metadata for UI/cgame rendering |
| `pc_token_t` | struct | Parsed token from the script/config parser |
| `qint64` | struct | Portable 64-bit integer (8 bytes, QVM-safe) |
| `connstate_t` | enum | Client connection state machine states |
| `trType_t` | enum | Trajectory interpolation modes |
| `soundChannel_t` | enum | Audio channel priority slots |
| `cbufExec_t` | enum | Command-buffer insertion modes |
| `ha_pref` | enum | Hunk allocator placement preference (high/low/dontcare) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `bytedirs[NUMVERTEXNORMALS]` | `vec3_t[]` | global (extern) | Lookup table of 162 precalculated unit normals for `DirToByte`/`ByteToDir` |
| `vec3_origin` | `vec3_t` | global (extern) | Constant zero vector |
| `axisDefault[3]` | `vec3_t[3]` | global (extern) | Identity axis matrix |
| `colorBlack…colorDkGrey` | `vec4_t` | global (extern) | Named RGBA color constants used by renderer/UI |
| `g_color_table[8]` | `vec4_t[]` | global (extern) | RGBA values indexed by `ColorIndex(c)` for `^N` color codes |

## Key Functions / Methods

### VectorNormalizeFast
- Signature: `static ID_INLINE void VectorNormalizeFast(vec3_t v)`
- Purpose: Normalizes a vector in-place using the fast reciprocal-square-root approximation.
- Inputs: `v` — vector to normalize (modified in place)
- Outputs/Return: void; `v` is mutated
- Side effects: none
- Calls: `Q_rsqrt`, `DotProduct` (macro)
- Notes: Does **not** check for zero-length; caller must ensure `|v| > 0`.

### CrossProduct
- Signature: `static ID_INLINE void CrossProduct(const vec3_t v1, const vec3_t v2, vec3_t cross)`
- Purpose: Computes the cross product of two 3-D vectors.
- Inputs: `v1`, `v2` — input vectors; `cross` — output vector (must not alias inputs)
- Outputs/Return: void; result in `cross`
- Side effects: none
- Calls: none

### Q_rsqrt
- Signature: `static inline float Q_rsqrt(float number)` (PPC) / `float Q_rsqrt(float f)` (x86)
- Purpose: Fast reciprocal square root; on PPC uses `frsqrte` + one Newton-Raphson step.
- Inputs: `number` — positive float
- Outputs/Return: approximate `1/sqrt(number)`
- Side effects: none
- Notes: Not defined here for x86 — declared extern, implemented in `q_math.c`.

### Com_sprintf / va
- Signature: `void QDECL Com_sprintf(char *dest, int size, const char *fmt, ...)` / `char * QDECL va(char *format, ...)`
- Purpose: Safe bounded `sprintf`; `va()` returns a rotating static string buffer.
- Side effects: `va()` writes to an internal static circular buffer.

### Info_* family
- `Info_ValueForKey`, `Info_SetValueForKey`, `Info_RemoveKey`, `Info_Validate`, `Info_NextPair`
- Purpose: Parse and mutate `\key\value\key\value` info strings used for serverinfo/userinfo cvars.
- Notes: Trivial but security-critical; `Info_Validate` rejects semicolons/quotes to prevent injection.

### Notes (trivial helpers)
- `VectorCompare`, `VectorLength`, `VectorLengthSquared`, `Distance`, `DistanceSquared`, `VectorInverse` — all small inline math utilities.
- `ClampChar`, `ClampShort`, `DirToByte`, `ByteToDir` — integer-range clamping and normal quantization.
- Angle helpers: `AngleMod`, `LerpAngle`, `AngleSubtract`, `AngleNormalize360/180`, `AngleDelta`, `AngleVectors`, `vectoangles`, `AnglesToAxis`.
- String helpers: `Q_stricmp`, `Q_strncpyz`, `Q_strcat`, `Q_PrintStrlen`, `Q_CleanStr`.
- Parse helpers: `COM_Parse`, `COM_ParseExt`, `COM_BeginParseSession`, `Parse1DMatrix/2DMatrix/3DMatrix`.

## Control Flow Notes
This is a **pure header** — it contains no entry points of its own. Every compilation unit in the engine `#include`s it before any other local header. At VM compile time (`Q3_VM` defined) the standard library is replaced by `bg_lib.h`. Platform selection (`WIN32`, `MACOS_X`, `__MACOS__`, `__linux__`, `__FreeBSD__`) at the top of the file configures byte-swap inline functions, `PATH_SEP`, `CPUSTRING`, and `ID_INLINE` for the rest of the build.

## External Dependencies
- `bg_lib.h` — VM-only C standard library replacement (included conditionally)
- `surfaceflags.h` — `CONTENTS_*` and `SURF_*` bitmask constants shared with q3map
- Standard C headers (`assert.h`, `math.h`, `stdio.h`, `stdarg.h`, `string.h`, `stdlib.h`, `time.h`, `ctype.h`, `limits.h`) — native builds only
- **Defined elsewhere:** `ShortSwap`, `LongSwap`, `FloatSwap` (byte-order helpers in `q_shared.c`); `Q_rsqrt`, `Q_fabs` on x86 (`q_math.c`); all `extern vec3_t`/`vec4_t` globals (`q_shared.c`); `Hunk_Alloc`/`Hunk_AllocDebug` (engine hunk allocator); `Com_Error`, `Com_Printf` (implemented per-module in engine/game/cgame/ui)
