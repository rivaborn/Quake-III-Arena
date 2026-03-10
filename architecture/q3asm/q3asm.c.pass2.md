# q3asm/q3asm.c — Enhanced Analysis

## Architectural Role

`q3asm.c` is a standalone QVM bytecode assembler—the final stage of the offline QVM compilation pipeline. It converts text-based Q3 assembly (produced by the lcc compiler) into executable QVM bytecode for the virtual machine. As an offline tool, it sits outside the runtime engine but is essential to the build process: all three runtime QVMs (`game`, `cgame`, `ui`) flow through this assembler. It has no dependencies on the renderer, network, or any other runtime subsystem—only on shared utility headers (`cmdlib.h`, `mathlib.h`, `qfiles.h`).

## Key Cross-References

### Incoming (who depends on this file)
- **Build system** (Makefile, Conscript files in `code/game/`, `code/cgame/`, `code/q3_ui/`, `code/ui/`) invokes `q3asm` as a command-line tool to assemble `.asm` files → `.qvm` files
- **lcc compiler** (`lcc/src/`) produces `.asm` source that `q3asm` consumes
- **qcommon/vm.c** (runtime) loads the resulting `.qvm` files but never calls `q3asm` directly

### Outgoing (what this file depends on)
- **lcc utilities** (`cmdlib.h`, `mathlib.h`): memory allocation, file I/O, error handling
- **qfiles.h**: defines the on-disk `.qvm` file format (`qvm_t`, executable header layout)
- **opstrings.h** (included at compile time): mnemonics ↔ opcode mapping table
- No runtime engine dependencies; purely offline/toolchain-oriented

## Design Patterns & Rationale

**Two-pass assembly:** Pass 0 collects all symbol definitions (functions, labels, static data); Pass 1 emits bytecode with symbol references resolved. This is the textbook approach for any assembler and allows forward references (jumping to a label defined later in the file).

**Hash-table symbol lookup:** Symbols are stored in a linked list ordered by value, with hash-based fast search (`HashString`). This is lightweight and appropriate for the QVM's modest symbol count (typically 50–500 per module).

**Four-segment model:** `{CODESEG, DATASEG, LITSEG, BSSSEG}` mirrors typical executable layout: executable instructions, initialized data (will be byte-swapped for endianness), string literal pool, and zero-initialized memory. Each segment has its own buffer and `segmentBase` offset, computed at the end of Pass 0.

**Deferred expression parsing:** The `ParseExpression` function handles symbol offsets, constant folding, and operator precedence (addition/subtraction) inline during opcode emission. No separate linker pass is needed.

**File-local symbol namespacing:** Symbols starting with `$` are prefixed with the source file index (`$foo` → `<fileindex>_$foo`) to guarantee uniqueness across multi-file links—elegant and prevents name collisions without a full linker.

## Data Flow Through This File

1. **Load phase:** Multiple `.asm` input files are read sequentially; `currentFileIndex` tracks identity for local symbol scoping.
2. **Pass 0 (symbol collection):**
   - `ExtractLine` → `Parse` → `ParseValue`/`ParseExpression` to extract pseudo-ops and labels
   - Directives like `.code`, `.data`, `.lit`, `.bss` switch `currentSegment`
   - Symbol definitions (`DefineSymbol`) are inserted into ordered linked list
   - vmMain entry-point security check: if non-zero offset, `Error()`
3. **Inter-pass:** Segment base addresses are computed by summing used bytes in order.
4. **Pass 1 (code emission):**
   - Opcode mnemonics are hashed against `sourceOps[]` table; operands are expressions evaluated via `LookupSymbol`
   - `EmitByte` / `EmitInt` append to segment buffers
5. **Output phase:** Binary segments are ordered (code, data, lit, bss), written to `.qvm` file with executable header.

## Learning Notes

**Minimal VM calling convention:** Q3's QVM is extremely simple—no relocation records, no complex linking, just four flat segments and symbol offsets. This contrasts sharply with ELF/PE loaders but reflects the VM's design: the engine knows the segment layout at load time.

**Byte-swap-on-emit:** `EmitInt` explicitly writes little-endian 32-bit values byte-by-byte (lines 223–227), ensuring portable binary format regardless of host endianness. Modern assemblers often defer this to a final linker pass; q3asm does it inline.

**Error-driven symbol resolution:** `LookupSymbol` (line 293) deliberately fails on undefined symbols during Pass 1, then retroactively defines them as zero to suppress cascading errors. This is a pragmatic error-recovery strategy.

**Static opcode table:** The `sourceOps[]` array (line 189) is built from `#include "opstrings.h"` at compile time. This ensures the assembler's opcode list is always synchronized with the VM's execution engine.

**Idiomatic for the era:** No AST, no intermediate representation—just line-by-line recursive-descent parsing. This was the standard approach in the 1990s–2000s for tools written in C.

## Potential Issues

1. **No cross-file forward references:** Symbols defined in File B cannot be referenced in File A if A is processed first. The multi-file link is depth-first and order-dependent—not a problem in practice since build rules ensure a fixed compile order, but fragile.

2. **Buffer overflow risk in `Parse()`** (lines 339–354): The token buffer is `MAX_LINE_LENGTH` (1024 bytes), and `lineBuffer` is also 1024. If a token happens to extend to the buffer boundary, subsequent reads could exceed bounds. Modern tools would use dynamic allocation or explicit length checks.

3. **Security check placement:** The vmMain offset check (lines 271–273) happens only on `DefineSymbol`, which is Pass 0 only. If someone somehow emits a relocatable vmMain reference in Pass 1, it would not be caught. The check is correct but relies on the assumption that vmMain is always a definition, never a reference.

4. **No error context in segment emission:** If `MAX_IMAGE` is exceeded, the error message "MAX_IMAGE" provides no file/line context, making debugging difficult on large files.
