# lcc/tst/array.c — Enhanced Analysis

## Architectural Role

This file is a **compiler regression test**, not runtime engine code. It exercises the lcc C compiler's code generation for multi-dimensional arrays, pointer arrays, and old K&R-style function declarations. It belongs to the offline **tooling ecosystem** (`lcc/` compiler → `q3asm/` assembler → `bspc/` AAS compiler) that generates QVM bytecode and offline assets, not to the runtime engine core (qcommon, client, server, renderer).

## Key Cross-References

### Incoming
- **None from runtime engine** — this file is consumed only by the lcc compiler test harness (`lcc/src/main.c` and build system)
- Test discovery/execution is driven by `lcc/makefile` or `buildnt.sh` / `buildnt.sh` scripts in the lcc root

### Outgoing
- **No engine dependencies** — does not call or link against qcommon, renderer, server, game VM, or botlib
- Uses only implicit C standard library functions (`printf`)
- No platform layer dependencies (no `Sys_*`, `NET_*`, `FS_*` syscalls)

## Design Patterns & Rationale

**Deterministic test case**: The code is structured to verify **array decay to pointers** and **multi-dimensional array parameter passing** — two historically compiler-fragile transformations:

- `y[i] = x[i]` tests implicit decay of `int[3][4]` → `int(*)[4]` (pointer to array)
- `g(x, y)` with `int x[][4], *y[]` parameters tests the K&R-style declaration syntax and compiler's ability to reconcile array parameters across compilation units
- The code is intentionally simple (deterministic output) so compiler behavior is directly observable

## Data Flow Through This File

1. **Initialization**: Global `x[3][4]` and `y[3]` are zero-initialized; local `z[3][4]` is stack-allocated
2. **Transformation**: `main()` populates `x`, assigns row pointers to `y`, copies to `z`, passes both to `g()`
3. **Output**: Two functions print the arrays in identical order—if output differs, the compiler or optimizer is incorrect
4. **Verification**: Byte-identical output from `f()` (global array access) vs `g()` (parameter-based access) proves correct array-to-pointer decay and indexing

## Learning Notes

**Era-specific idioms**:
- K&R function declarations (`g(x, y) int x[][4], *y[]`) were standard in pre-ANSI C; this test verifies the lcc compiler still handles them
- **No ANSI prototypes** — typical of 1990s compilers testing for backwards compatibility
- The separate `f()` (global) vs `g()` (parameter) pattern verifies that the compiler's pointer arithmetic for array indexing is **invariant across scope boundaries**

**Modern contrast**:
- Modern C would use ANSI prototypes: `void g(int x[][4], int *y[3])`
- This test would be replaced by compiler fuzzing (e.g., AFL) or property-based testing in contemporary QA

**Engine connection** (indirect):
- The **qvm bytecode** generated from game/cgame VM code is compiled by *exactly this lcc compiler*
- If lcc mis-generates array indexing, **bot AI pathfinding and game entity arrays would silently corrupt** at runtime
- Thus this test indirectly validates game VM correctness through compiler validation

## Potential Issues

None evident in the test itself. However, the **lcc compiler's handling of array-to-pointer decay is non-trivial** on some platforms (especially those lacking hardware address modes for strided indexing like PPC). The test would fail loudly if:
- Multi-dimensional array indexing calculated offsets incorrectly
- Pointer arithmetic across array dimensions was misaligned
- K&R declaration parsing introduced parameter-binding errors

---

**Note**: This file has **zero architectural coupling** to the runtime Quake III engine. It is a **compiler correctness check**, not an engine component or asset.
