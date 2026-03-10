# lcc/etc/irix.c — Enhanced Analysis

## Architectural Role

This file is a platform-specific compiler configuration for the LCC C-to-bytecode compiler infrastructure on SGI IRIX (big-endian MIPS) systems. It defines the compilation toolchain pipeline (preprocessor → compiler → assembler → linker) used during the engine build to translate C source code into QVM bytecode for the three virtual machines: game, cgame, and ui. While the file itself is build-time infrastructure, the bytecode it produces is embedded into the runtime engine and executed by the `qcommon/vm.c` host.

## Key Cross-References

### Incoming (Build System Dependencies)
- **Build orchestration** (`code/Makefile`, `code/Construct`, platform-specific `Conscript-*` files) reads this configuration when cross-compiling or building on IRIX
- **LCC compiler build process** (`lcc/makefile`, platform-specific `*.mak` files) includes this to set up the IRIX toolchain
- **Q3ASM assembler** (`q3asm/q3asm.c`) uses the bytecode output from the `rcc` → `as` → `ld` chain

### Outgoing (Tools & External Dependencies)
- Invokes platform-specific binaries: `/usr/bin/as` (IRIX assembler), `/usr/bin/ld` (IRIX linker)
- Preprocessor directives (`-DMIPSEB`, `-DSYSTYPE_SVR4`, etc.) influence the entire LCC compilation path; these must match defines in `lcc/src/*.c` to ensure the compiler targets MIPS correctly
- Links against IRIX libc (`-lc`, `-lm`) and sysroot libraries (`/usr/lib/crt*.o`)

## Design Patterns & Rationale

**Multi-stage toolchain pattern**: The compilation is decomposed into discrete stages—preprocessor, compiler, assembler, linker—each with its own tool invocation. This allows:
- Incremental builds (preprocessing separate from codegen)
- Debug flexibility (inspecting intermediate `.i`, `.s` files)
- Tool substitutability (swap `as` for a different assembler without touching the compiler)

**Platform abstraction via #defines**: Rather than embedding IRIX-isms in the compiler source, the configuration file injects them via `-D` flags. This means `lcc/src/*.c` can remain platform-neutral while `irix.c`, `linux.c`, `win32.c` tailor the preprocessor input per target.

**Hardcoded paths and LCCDIR fallback**: The config allows override via `-lccdir=` option, enabling cross-compilation environments where the standard `/usr/local/lib/lcc/` path is inaccessible. The `option()` function updates tool pointers dynamically.

**Profile/debug option handling**: The `-p` flag swaps in `/usr/lib/mcrt1.o` (profiled CRT) instead of the standard `/usr/lib/crt1.o`, demonstrating how compilation profiles are layered without full rebuild.

## Data Flow Through This File

1. **Build system** calls LCC with source `.c` files and compiler options (e.g., `-lccdir=/custom/path -p`)
2. **`option()` handler** intercepts and modifies the toolchain arrays (`cpp`, `com`, `ld`) in-place
3. **Preprocessor** (`cpp`) receives define list and source; outputs `.i` (preprocessed) files with MIPS/IRIX macros expanded
4. **Compiler** (`rcc`) consumes `.i`, applies target-specific codegen rules for MIPS, outputs `.s` (assembly)
5. **Assembler** (`as`) assembles `.s` with IRIX directives (`-KPIC`, `-nocpp`) to `.o` object files
6. **Linker** (`ld`) combines object files with IRIX-specific flags (`-require_dynamic_link`, `-elf`, `-_SYSTYPE_SVR4`) and CRT objects to produce executables or shared objects
7. **Q3ASM** post-processes outputs into QVM bytecode archives (`.qvm` or `.qax`)

## Learning Notes

- **LCC is a cross-platform compiler**: The presence of `irix.c`, `linux.c`, `win32.c`, `solaris.c` shows that LCC was engineered to retarget itself across UNIX dialects and Win32 without source modifications—only toolchain configuration changes.
- **MIPS/IRIX specifics**: The extensive define list (`_MIPS_ISA`, `_MIPS_SIM=ABI32`, `_MIPS_FPSET=16`) enforces a specific MIPS ABI (32-bit, IEEE 754 floating-point, MIPS I ISA). Modern engines use LLVM or GCC, which auto-detect this; Q3 required explicit flags.
- **ABI compatibility**: The CRT files (`/usr/lib/crt1.o`, `/usr/lib/crtn.o`) are mandatory because IRIX enforces specific dynamic linking conventions (`_rld_new_interface`). Mismatches cause runtime crashes.
- **Static vs. dynamic linking policy**: The linker flags impose dynamic linking (`-require_dynamic_link`), a constraint of IRIX shared object model. The engine probably statically links `libllcc` but dynamically links libc and libm.

## Potential Issues

- **Hardcoded system paths** (`/usr/bin/as`, `/usr/lib/crt1.o`, `/usr/lib/crtn.o`, `/usr/lib/mcrt1.o`) fail if IRIX is cross-compiled or sysrooted; no option to redirect these at build time
- **No error checking in `option()`**: The function silently ignores unrecognized flags (returns 0), which can mask typos in `-lccdir=` paths or unsupported options
- **LCCDIR default is absolute**: If the installation path differs from `/usr/local/lib/lcc/`, users must always pass `-lccdir=` or modify the file. No environment variable fallback (unlike `$LCCDIR` or similar)
- **Profile flag (`-p`) is a global edit**: Changing `ld[12]` affects all subsequent compilations in the same process, risking state corruption in parallel builds or IDE integrations that invoke the compiler multiple times
