# lcc/tst/stdarg.c — Enhanced Analysis

## Architectural Role

This test file validates **variadic argument handling** (`stdarg.h`) in the LCC C compiler, which is the primary C→QVM bytecode compiler for the Quake III build pipeline. While lcc itself is an offline tool (not runtime engine), correct `stdarg` support is critical because the engine's **game VM** (`code/game`), **cgame VM** (`code/cgame`), and **botlib** extensively use variadic functions like `G_Printf`, `CG_Printf`, and custom logging/formatting functions that must compile correctly to QVM bytecode.

## Key Cross-References

### Incoming (Why stdarg matters in Q3A)
- **Game VM** (`code/game/g_local.h`): Defines `G_Printf(...)` used throughout entity logic and debugging
- **cgame VM** (`code/cgame/cg_local.h`): Defines `CG_Printf(...)` for client-side logging
- **botlib** (`code/botlib/l_log.c`): Logging infrastructure relies on variadic functions
- **qcommon** (`code/qcommon/cmd.c`, `common.c`): Core engine services like `Com_Printf` use varargs

### Outgoing (what lcc depends on)
- LCC compiler runtime: includes `<stdarg.h>` from lcc's own libc implementation (not the host system's)
- Test harness assumes `printf` availability in lcc's hosted environment

## Design Patterns & Rationale

**Compiler Test Structure:**
- Tests **both custom format specifiers** (`%b` for struct, `%w` for short hex) and **standard C specifiers** (`%d`, `%f`, `%c`, `%s`)
- Validates **struct-by-value passing** through varargs (the `%b` case with `struct node x`)—this is non-trivial to compile correctly into VM code
- Tests **type coercion** edge cases: `va_arg(ap, char)` vs `va_arg(ap, short)`, `va_arg(ap, double)` vs integer formats

**Why this matters:** QVM code runs in a sandboxed virtual machine with its own calling conventions. The LCC compiler must generate correct push/pop sequences for varargs that may cross word boundaries (structs wider than pointers, doubles, etc.). The standard C library test suite (lcc/tst/) acts as a regression suite for LCC itself.

## Data Flow Through This File

1. **Compile-time:** LCC parses `stdarg.h` macro definitions (`va_start`, `va_arg`, `va_end`)
2. **Code generation:** Compiler generates VM bytecode that:
   - Saves `ap` pointer on stack as offset from frame pointer
   - Increments `ap` by correct size for each `va_arg` call (size varies by type and alignment)
3. **Runtime (when test is executed by lcc test harness):** VM bytecode executes, printing formatted output
4. **Validation:** Output compared against expected baseline

## Learning Notes

**Idiomatic to lcc and early 2000s compilers:**
- **No DWARF debug info:** The test relies purely on source inspection, not runtime inspection of types
- **Manual type size calculation:** The `va_arg(ap, TYPE)` macro must be manually sized; modern compilers would use `sizeof()`
- **Struct-by-value is exceptional:** Passing `struct node` through varargs was uncommon in 1990s C; most codebases passed pointers. LCC's test validates an edge case

**Contrast with modern engines:**
- Modern game engines (Unreal, Unity) run C# in a managed runtime with built-in reflection, eliminating varargs complexity
- Quake III's VM-based architecture required lcc to handle these edge cases because QVM code *is* the game logic layer

**Connection to QVM architecture:**
- This test indirectly validates the entire QVM calling convention: if `stdarg` works, so do function pointers, struct returns, and stack discipline
- The presence of `struct node` in varargs is deliberate stress-testing of the ABI

## Potential Issues

**ABI portability risk (if inferable):**
- The `%w` format specifier assumes `short` is 16 bits and extracts as hex via `%x`—this breaks on platforms where `int` ≠ 32 bits (rare but possible)
- The test does not validate that `printf("%f", va_arg(ap, double))` produces correct output across different FPU implementations (lcc supports multiple target architectures: x86, PPC, MIPS)

**No coverage of:**
- Misalignment of 8-byte values (doubles, long long) in varargs on stack—modern architectures require alignment, but test may not catch this if the test harness tolerates subtle corruption
