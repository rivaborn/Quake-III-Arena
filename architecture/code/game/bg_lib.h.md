# code/game/bg_lib.h

## File Purpose
A self-contained replacement header for standard C library declarations, intended exclusively for use when compiling game code targeting the Quake III virtual machine (QVM). It is explicitly not included in native host builds.

## Core Responsibilities
- Provides `size_t` and `va_list` type definitions for the VM environment
- Declares integer limit macros (`INT_MAX`, `CHAR_BIT`, etc.) normally found in `<limits.h>`
- Declares variadic argument macros (`va_start`, `va_arg`, `va_end`) normally from `<stdarg.h>`
- Declares string manipulation function prototypes replacing `<string.h>`
- Declares memory operation prototypes replacing `<string.h>`/`<memory.h>`
- Declares math function prototypes replacing `<math.h>`
- Declares misc stdlib prototypes (`qsort`, `rand`, `atoi`, `atof`, etc.) replacing `<stdlib.h>`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `size_t` | typedef (`int`) | VM-safe size type; replaces the platform's native `size_t` |
| `va_list` | typedef (`char *`) | Variadic argument list pointer for the VM calling convention |
| `cmp_t` | typedef (function pointer) | Comparator signature for `qsort` |

## Global / File-Static State
None.

## Key Functions / Methods

No function bodies are defined here — this is a header of declarations only. The implementations live in `bg_lib.c`.

**Notes on notable declarations:**
- `_atof` / `_atoi`: Non-standard variants that accept a `const char **` and advance the pointer, enabling in-place string parsing.
- `vsprintf` / `sscanf`: Formatted I/O without file handle dependencies, safe for the sandboxed VM.
- `qsort`: Custom sort implementation; uses the `cmp_t` function-pointer typedef for comparators.

## Control Flow Notes
This header is a passive compile-time shim. It is pulled in by VM-targeted translation units (cgame, game, ui modules compiled with `q3asm`/`lcc`) that cannot link against the host OS's C runtime. There is no init/frame/shutdown participation; it simply ensures declarations are available at compile time so `bg_lib.c` implementations resolve correctly inside the QVM.

## External Dependencies
- No includes — this file is itself the bottom of the dependency chain for VM builds.
- All declared symbols are **defined in** `code/game/bg_lib.c` (not inferable from this file alone, but implied by the file comment).
- `va_start`/`va_arg`/`va_end` macros assume a simple cdecl-style stack layout matching the QVM's int-aligned argument passing; they are **not** portable to x86-64 or other ABIs and must never be used in native builds.
