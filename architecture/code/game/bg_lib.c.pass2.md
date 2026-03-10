# code/game/bg_lib.c — Enhanced Analysis

## Architectural Role

This file is a **VM-only libc stub layer** that sits at the boundary between Q3VM module isolation and the underlying platform. Every VM-compiled module (game, cgame, ui) gets its own copy of bg_lib linked at compile time via LCC's linker. It exists because the LCC compiler (used to build QVM bytecode) emits implicit calls to standard C library functions, but Q3's VM architecture intentionally isolates each module with no access to the host's libc. By providing these stubs directly, the engine avoids either external linker dependencies (impossible in a sandbox) or forcing each game logic file to reimplement `strlen` locally.

Under **native (non-VM) builds**, only `qsort`, `memmove`, `rand`/`srand`, and numeric conversion functions are compiled; VM-specific stubs like `strcpy`, `sprintf` are excluded (marked `#if defined(Q3_VM)`), allowing native code to use the platform's optimized libc.

## Key Cross-References

### Incoming (who depends on this file)
- **game VM** (`code/game/*.c`): Uses `qsort` (ai_dmq3.c entity sorting), string functions (entity parsing in g_spawn.c), numeric conversion (g_items.c weapon configs), `vsprintf` (console output)
- **cgame VM** (`code/cgame/*.c`): Uses `qsort` (cg_scoreboard.c), string functions, math (`tan` in cg_effects.c for projectile arcs), `rand` (particle effects)
- **ui VMs** (`code/q3_ui/` and `code/ui/`): Use string/numeric functions for menu config parsing, `qsort` for server browser sorting
- **Compiler/linker artifact**: LCC emits implicit calls to `strlen`, `strcpy`, `memcpy` during code generation — these symbols must resolve at link time

### Outgoing (what this file depends on)
- **q_shared.h** (included): Provides `Q3_VM` macro (controls conditional compilation), `size_t`, `va_list`, `M_PI`, `qtrue`/`qfalse` type definitions
- **bg_lib.h** (implicitly via q_shared.h): May define `cmp_t` for Q3_VM builds
- **No engine dependencies**: Notably does NOT call back into qcommon, renderer, or server — pure utility code

## Design Patterns & Rationale

**Pattern 1: Dual-purpose compilation via `#if defined(Q3_VM)`**
- VM builds compile full suite of stubs (strings, math, I/O)
- Native builds skip VM-only code, relying on platform libc for performance-critical operations
- **Rationale**: Avoids code duplication while keeping VM modules self-contained

**Pattern 2: LCC/Q3VM calling convention adaptation in `vsprintf`**
```c
#ifdef __LCC__
  va_list argptr;    // LCC: va_list is int*, advance by 1 per call
#else
  double d;          // Native: floats advance by 2 per %f
  va_list = (int*)&d + 2;
#endif
```
- **Rationale**: LCC's ABI differs from standard C ABIs; the file hardcodes this knowledge

**Pattern 3: Bentley-McIlroy 3-way quicksort with goto-loop instead of recursion**
- Uses tail recursion optimization (`goto loop`) to reduce stack depth
- Median-of-3 pivot selection; ninther for large arrays; fallback to insertion sort for small/nearly-sorted runs
- **Rationale**: Portable, proven algorithm; stack efficiency critical in embedded VM contexts

**Pattern 4: Table-driven lookup disabled (`#if 0` wrapped code)**
- `sintable[1024]`, `acostable[]`, pre-computed `sin`/`cos`/`acos` functions are all compiled out
- Only `tan` (via `cos`/`sin` ratio) remains live
- **Rationale**: Suggests these were early optimizations later superseded by better methods or more RAM budgets

## Data Flow Through This File

1. **Incoming**: Each VM module gets a self-contained copy at link time; LCC compiler inserts implicit calls during codegen
2. **Transformation**:
   - String ops: character-by-character mutation (strcpy, strlen)
   - Math: table lookup (disabled) or inline computation (`tan`)
   - I/O: format string parsing into token stream (`vsprintf`, `sscanf`)
   - Numeric: character→integer/float conversion with side effects (`_atoi` advances pointer)
3. **Outgoing**: Results written back through caller-supplied buffers; no state persisted across calls except `randSeed`

## Learning Notes

**Idiomatic patterns a developer studying Q3 learns:**
- **VM isolation**: Q3's security model relies on VMs having zero access to host processes; utility libraries must be self-contained
- **LCC conventions**: The `%f` handling in `vsprintf` is a micro-example of how Q3 developers had to adapt to non-standard compiler ABIs
- **Pointer-advance string parsing** (`_atoi`, `_atof`): Idiomatic to Quake token parsing; seen throughout `qcommon/cmd.c`, `client/cl_parse.c`. These functions return a value AND advance the input pointer, enabling clean token loops without manual cursor management
- **No dynamic allocation**: Unlike modern game engines with custom allocators, Q3's utility layer operates purely on stack and caller-owned buffers
- **Lean feature set**: Surprisingly minimal feature coverage (no wide chars, no `%g` formatting, no `*` width specifiers). This reflects Q3's target: deterministic, predictable gameplay logic with no I18N

**Modern engines vs. Q3:**
- Modern engines provide full-featured allocators and formatting libraries to VM guests; Q3 assumes the VM is mostly self-sufficient
- Most modern VMs (WASM, Lua, etc.) rely on host OS libc; Q3's approach is idiosyncratic to its sandbox design

## Potential Issues

1. **Buffer overflow in `vsprintf`**: No bounds checking on output buffer. Callers must size buffers correctly. Acceptable given tight control over VM module sources, but reflects pre-security-hardening design.
2. **Incomplete numeric parsing**: `atof` does not handle scientific notation (`1e-3`), limiting config file expressiveness (though Quake doesn't use it).
3. **`sscanf` return value always 0**: Acknowledged as "really crappy" in first-pass; callers cannot distinguish parse success from failure. Works only if inputs are known-good (e.g., from BSP entity strings).
4. **Thread-unsafe `randSeed`**: If any bot AI runs on a worker thread (not the case in original Q3, but potential issue in engines using async physics), the PRNG state would race.
