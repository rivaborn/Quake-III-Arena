# lcc/include/alpha/osf/stdlib.h — Enhanced Analysis

## Architectural Role

This file is a platform-specific C standard library header for the **Alpha OSF** target architecture within the LCC compiler. LCC is the off-chain tool that compiles game module source (game VM, cgame VM, UI VM) into QVM bytecode. This particular header provides the stdlib interface declarations that the LCC compiler exposes when cross-compiling code destined for Alpha/OSF runtime environments, ensuring standard library function signatures and types are available during source compilation.

## Key Cross-References

### Incoming (who includes this file)
- **Source files compiled by LCC**: Any `.c` file in `code/game/`, `code/cgame/`, `code/q3_ui/`, or `code/ui/` that `#include <stdlib.h>` will resolve to this file (when targeting Alpha/OSF)
- **LCC compiler frontend**: The preprocessor includes this during the `#include <stdlib.h>` resolution phase for Alpha/OSF targets
- **Other platform-specific stdlib headers**: Similar role fulfilled by `lcc/include/x86/linux/stdlib.h`, `lcc/include/mips/irix/stdlib.h`, `lcc/include/sparc/solaris/stdlib.h`, etc.

### Outgoing (what this file depends on)
- No runtime dependencies; this is a compile-time header only
- The declared functions (`malloc`, `free`, `qsort`, `atoi`, etc.) must be implemented or linked at QVM bytecode runtime
- Type definitions (`size_t`, `wchar_t`) are target-intrinsic and do not refer to other headers

## Design Patterns & Rationale

1. **Platform-specific directory tree** (`lcc/include/<arch>/<os>/`): Each target platform gets its own `stdlib.h` because standard library type sizes and ABIs vary (e.g., `size_t` is `unsigned long` on Alpha/OSF but could differ on 32-bit x86).

2. **Double-guard for types** (`_SIZE_T` / `_SIZE_T_` and `_WCHAR_T` / `_WCHAR_T_`): Prevents redefinition if multiple headers in the same compilation unit (or system headers) define the same types. Alpha/OSF's include system may define these elsewhere, so both guards ensure compatibility.

3. **Minimal type definitions**: Only `div_t`, `ldiv_t`, `size_t`, and `wchar_t` are defined; no complex structs or opaque handles. This keeps the header lightweight and portable.

4. **Extern function declarations without definitions**: The header declares function **signatures only**; implementations are provided by the QVM runtime's standard library or game module's own implementations. This allows the compiler to type-check calls without requiring the actual binary.

5. **Standard macro constants** (`EXIT_FAILURE`, `EXIT_SUCCESS`, `NULL`, `RAND_MAX`): These are ABI-stable constants that don't change between compilations, so they can be baked into the source at compile time.

## Data Flow Through This File

1. **Compilation phase**: Source in `code/game/`, `code/cgame/`, etc., includes `<stdlib.h>`.
2. **Preprocessor resolution**: LCC's preprocessor finds `lcc/include/alpha/osf/stdlib.h` (based on target platform).
3. **Type checking**: Type definitions are inserted into the compilation unit; function signatures are recorded in the symbol table.
4. **Code generation**: Any call to (e.g.) `malloc(size)` is type-checked against the extern declaration; the call is emitted as a syscall or direct function reference in QVM bytecode.
5. **Linking/QVM assembly**: The Q3ASM assembler resolves function names to syscall indices or VM imports (via `code/game/g_syscalls.c`, `code/cgame/cg_syscalls.c`, etc.).
6. **Runtime**: The game/cgame VMs execute; syscalls trap back to the engine, which dispatches to real `malloc`, `free`, etc., or custom implementations.

## Learning Notes

- **Libc is not linked into QVM**: Unlike native executables, QVM modules do not link against a traditional libc. Instead, they declare stdlib functions as **extern** and rely on the engine's **syscall ABI** to provide implementations at runtime. This is how the VM sandbox enforces memory isolation: `malloc` and `free` trap into the engine, not user-space.
- **Platform portability via separate headers**: Quake III's build pipeline supports multiple target platforms (Alpha/OSF, x86/Linux, MIPS/IRIX, SPARC/Solaris, x86/Win32). Each platform's LCC tree has its own `stdlib.h` so that type sizes (e.g., `size_t`, `wchar_t`) match the target architecture. This is **not** dynamic; it's a compile-time choice.
- **Contrast with modern engines**: Modern game engines typically use either (a) a platform-agnostic scripting language (Lua, C#) or (b) WASM with a well-defined ABI. Quake III chose to **recompile the entire source tree for each target**, so each platform got its own LCC toolchain with platform-specific headers.
- **No POSIX assumptions**: Despite being Alpha/OSF-specific, the header only declares standard C library functions, not POSIX extensions (no `open`, `read`, `mmap`, etc.). This keeps the QVM sandbox small and predictable.

## Potential Issues

- **No bounds checking on string functions**: Declarations like `extern char *getenv(const char *)` and `extern int system(const char *)` have no length parameters. Game code using these must be careful to avoid buffer overflows. (This is inherited from C89, but worth noting for VM sandbox security.)
- **Missing stdlib functions for modern C**: Functions like `malloc_usable_size`, `aligned_alloc`, and other C11+ features are not present. Code compiled with LCC is constrained to C89/C99 stdlib.
- **wchar_t as `unsigned char`**: On Alpha/OSF, `wchar_t` is defined as a single byte, which is unusual (most platforms use 2 or 4 bytes). This may cause issues if game code assumes wide characters are wider than 8 bits, though in practice Quake III does not use wide characters extensively.
