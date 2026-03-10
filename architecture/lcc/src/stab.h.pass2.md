# lcc/src/stab.h — Enhanced Analysis

## Architectural Role

This file defines the debugging symbol table (`stab`) format used by the **lcc C compiler** during QVM bytecode generation. While most Quake III code concerns the runtime engine, lcc is a compile-time tool responsible for translating game VM source (cgame, game, ui) into QVM bytecode. The symbol definitions here enable post-compilation debug information embedding—a bridge between compiler intermediate representation and debugger/profiler consumption.

Unlike the runtime engine subsystems, this file has **no runtime impact**: the QVM bytecode shipped to players contains only stab entries the compiler chose to emit; the engine never reads or interprets these symbols at load time.

## Key Cross-References

### Incoming (who depends on this file)
- **lcc compiler internals** (`lcc/src/*.c`): parser, code generator, symbol table management use these constants to tag symbol entries
- **Q3ASM** (`q3asm/*.c`): may consume stab symbols if imported from intermediate `.o` files
- Indirectly: **q3map** and **bspc** (offline tools) may preserve stab entries when processing object files, though the cross-reference list shows no explicit includes

### Outgoing (what this file depends on)
- **a.out.h format** (referenced in comments): provides base `nlist` structure; this file extends it with debugger-specific fields (`n_desc`, `n_value` reinterpretation)
- **No subsystem dependencies**: pure type/constant definitions; no function calls, no runtime engine interaction

## Design Patterns & Rationale

**Historical Sun/BSD Format**: This is a Sun Microsystems standard (1990, SMI copyright) for ELF/a.out interoperability. The design reflects **early-1990s Unix debugging conventions**:

- **Bit flags in `n_type`**: `N_STAB` bits (high bits of `n_type`) distinguish debug symbols from relocatable code symbols
- **Semantic field overloading**: `n_desc` and `n_value` fields are reused for debug metadata (e.g., `n_desc` = scope nesting level for `N_LBRAC`/`N_RBRAC`; = type code for `N_LSYM`)
- **Pascal, Fortran support**: constants like `N_PC` (berkeley pascal), `N_SO_FORTRAN` show this was designed for multi-language compiler suites

**Rationale for embedding in lcc**: Rather than use modern DWARF or source-line-only debug info, lcc preserves **backward-compatible a.out/stab format** because:
1. Simplifies compiler backend (no new IR needed)
2. Integrates with existing Unix debugging infrastructure (`sdb`, `gdb` partial support)
3. Minimal overhead: optional compile-time emission

## Data Flow Through This File

```
Compiler source (game/*.c, cgame/*.c)
    ↓
lcc parser / semantic analysis
    ↓
Code generator marks entities with N_GSYM, N_STSYM, N_LSYM, etc.
    ↓
Compiler backend writes nlist[] entries to object file (.o)
    ↓
q3asm reads .o, assembles to QVM bytecode (may strip stab or preserve)
    ↓
Engine loads .qvm → no runtime stab interpretation
```

The file itself is **static definition**; data flows *through* its constants, not *to/from* the file.

## Learning Notes

**What developers studying this engine learn**:
- **No runtime debugging**: Unlike modern engines, Q3A's VM bytecode carries optional debug metadata that the shipped engine ignores. Developers rebuild locally with debug symbols for offline analysis.
- **Compiler-toolchain separation**: This demonstrates that the compilation pipeline (lcc → q3asm → .qvm) is **completely decoupled** from the runtime (engine loads .qvm as opaque bytecode).
- **Historical artifact**: The file is unchanged since 2005 release and reflects 1990s practices. Modern game engines use DWARF or custom formats; Q3A chose minimal compatibility over feature-richness.

**Idiomatic to this engine/era**:
- Symbol table entries as **compile-time metadata only** (no runtime interpretation)
- Reuse of platform's native debugging format (Sun stab) rather than engine-specific format
- **Optional** debug info: stripped by default for shipping bytecode, embedded when needed for offline profiling

## Potential Issues

No issues are inferable from this file alone. **Implicit assumption**: lcc correctly generates `N_*` constants when emitting debug symbols. If a custom debugger/profiler were built, misalignment between lcc's emitted values and this header's constants would cause symbol misinterpretation—but this is a compile-time contract, not a runtime risk.

The file provides **no validation**: a malformed `nlist` entry from a broken lcc run would silently be ignored by the engine and might confuse offline tools. No checksums or magic markers guard this data.
