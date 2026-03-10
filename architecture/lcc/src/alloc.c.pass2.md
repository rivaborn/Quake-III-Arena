# lcc/src/alloc.c — Enhanced Analysis

## Architectural Role

This file provides the memory management backbone for the **LCC C compiler tool**—used offline to compile game code into QVM bytecode, not for runtime engine execution. It implements an arena-based allocator optimized for compiler workloads: fast per-phase allocation with bulk deallocation between compilation phases. The dual-mode design (PURIFY vs. production) reflects a compilation-era practice of supporting both instrumented debugging (via valgrind) and optimized execution.

## Key Cross-References

### Incoming (who depends on this file)
- **Entire LCC compiler**: `lcc/src/{alloc,dag,decl,expr,gen,parse,stmt,sym,tree}.c` and similar—all call `allocate()` and `deallocate()` to manage parse trees, symbol tables, and intermediate code
- **Other lcc utility modules** in `lcc/src/`: lexer, code generator, symbol management—all use the three arena tiers for different lifetime categories
- **q3asm tool**: `q3asm/q3asm.c` reuses this allocator (shares `cmdlib.h` and memory infrastructure with lcc)

### Outgoing (what this file depends on)
- **Standard C library**: `malloc()`, `free()` for OS-level allocation
- **Common error handling**: `error()` function (libc-style), `exit()` for OOM
- **Common macros** (from `c.h`): `NELEMS()`, `assert()`, `roundup()`
- **No runtime engine dependencies** — this is build-tool code, completely isolated from `code/` subsystems

## Design Patterns & Rationale

**Two-Mode Architecture (PURIFY / Production)**:
- **PURIFY mode**: Each allocation goes directly to `malloc()`, tracked in a global arena list for later `free()`. Ideal for debugging with valgrind/AddressSanitizer—every allocation is independently tracked and freeable.
- **Production mode**: Block-based arena with a free-list; allocations carved from large pre-allocated blocks (`~10KB` chunks). Minimizes malloc/free syscall overhead during the hot compilation phase.

**Arena Indexing (3 tiers)**:
- Arenas are indexed `0, 1, 2`—likely corresponding to compiler phases or symbol-table lifetimes (e.g., global scope, function scope, statement scope)
- `deallocate(a)` atomically frees all blocks in arena `a`, resetting it to the initial empty block
- No per-allocation deallocation—only bulk arena reset; fits compiler's phase-oriented memory model

**Alignment Union** (`union align`):
- Deliberately over-aligns: includes `long`, `char*`, `double`, and function pointers to satisfy the strictest alignment requirement on the platform
- Header union combines this with the block structure, ensuring returned pointers (`new + 1`) land on correct boundaries

**Roundup Strategy**:
- All allocations are rounded up to `sizeof(union align)` to maintain alignment within a block; prevents unaligned reads/writes in code generator

## Data Flow Through This File

1. **Compiler startup**: Initialize three arenas with one empty `struct block` each
2. **Parse/codegen phases**: Call `allocate(bytes, arena_index)` to get aligned memory for parse trees, symbols, intermediate code
3. **Inter-phase boundary**: Call `deallocate(arena_index)` to reclaim all blocks in that arena at once; reset to empty state
4. **Block exhaustion**: When current block's remaining space (`ap->limit - ap->avail`) runs out:
   - Try to reuse a freed block from the free-list
   - If none available, `malloc()` a new block (~10KB + requested size)
   - Link it into the arena chain, update `arena[a]` pointer
5. **Shutdown**: Final `deallocate()` calls reclaim any remaining blocks

## Learning Notes

**Idiomatic Compiler Design (1990s era)**:
- Per-phase arena allocation is a classic compiler pattern; modern engines (e.g., LLVM) use similar models but with more sophisticated freeing (bump-pointer allocators, destructors)
- The 10KB magic constant reflects typical compilation memory pressure circa 2000—small enough to avoid huge waste, large enough to amortize syscall cost
- **No individual deallocation** enforces discipline: no memory fragmentation, no use-after-free, simpler reasoning

**Connection to Broader Codebase**:
- This allocator serves only the **build-time tool chain** (`lcc`, `q3asm`, `q3map`, `bspc`)
- The **runtime engine** (`code/client`, `code/server`, `code/game`, `code/renderer`) uses a completely separate hunk/zone system in `code/qcommon/common.c`—no code sharing
- Both subsystems are architecturally isolated; qcommon's allocator must be deterministic and multi-frame safe, whereas lcc's can be aggressive and phase-oriented

**Platform Portability**:
- The conditional `#ifdef PURIFY` demonstrates the tool's multi-platform heritage; allows same source to run under valgrind (PURIFY) on Linux or optimized on Win32/macOS

## Potential Issues

- **No overflow protection**: `allocate(n, a)` rounds up `n` but does not check for integer overflow in `m*n` within `newarray()`—a misbehaving caller can wrap around
- **Arena indices unchecked for bounds**: `assert(a < NELEMS(arena))` relies on assertions, which compile out in release builds; a buggy caller passing `a=5` corrupts heap
- **PURIFY mode memory overhead**: Every allocation stores a `union header` and keeps a next-pointer chain; fine for debugging, wasteful for production
- **No realloc**: If code generator needs to resize a block, only option is allocate-new and copy—no way to extend in-place

---

**Note**: This analysis focuses on how `lcc/src/alloc.c` fits the *tool-chain architecture*, not the runtime engine. It has no cross-dependencies with `code/qcommon`, renderer, or game logic.
