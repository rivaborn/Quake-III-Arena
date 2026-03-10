# lcc/src/trace.c — Enhanced Analysis

## Architectural Role

This file implements compiler-level **debug instrumentation generation** within the LCC C compiler used to build QVM bytecode. It intercepts function entry and exit points during code generation to emit formatted trace output (via `printf` or custom printer), enabling post-compilation profiling and debugging of compiled game logic (game VM, cgame VM, ui VM). The instrumentation is entirely transparent to source code—activated only via `-t<printer>` compiler flags—and integrates with LCC's event-driven IR generation pipeline.

## Key Cross-References

### Incoming (callers)
- **LCC compiler main (`lcc/src/main.c`)**: Passes `-t` compiler flags to `trace_init()` during compiler initialization
- **LCC IR event system (`events.entry`, `events.returns` global)**: Registers `tracecall` and `tracereturn` as `Apply` callbacks attached to function entry/return IR events
- **Type initialization**: Depends on `type_init()` (from `lcc/src/types.c`) for base type objects (`chartype`, `inttype`, `longtype`, etc.)

### Outgoing (dependencies)
- **LCC IR/codegen**: 
  - Reads `IR` global pointer to check if code generation is active
  - Calls `mkop()`, `tree()`, `walk()` to synthesize and emit IR nodes for trace calls
  - Uses `idtree()` to generate symbol references, `cast()` for type conversions, `rvalue()` for rvalues
  - Accesses symbol attributes: `name`, `u.f.callee[]` (function parameter symbols), `u.c.loc` (static storage locations)
- **Memory/symbol management**:
  - `allocate(size, FUNC)` for heap-allocated format strings
  - `genident(STATIC/AUTO, ...)` to create fresh compiler-generated symbols
  - `mkstr()` to create string constants (null pointer fallback for string pointers)
  - `mkop(ARG, ...)`, `tree(ARG+P, ...)` to construct IR argument nodes
- **Type system**:
  - `unqual()` strips qualifiers from types
  - `promote()` applies standard C integer promotions
  - `typestring(ty, "")` generates human-readable type names
  - `freturn()` extracts return type from function type
  - Base types: `chartype`, `signedchar`, `unsignedchar`, `longtype`, `unsignedlong`, `longdouble`, `unsignedtype`, `inttype`, `voidtype`
- **AST/expression builders**: `addrof()`, `field()`, `pointer()`, `condtree()`, `retype()`, `consttree()`, `optree['+']` (operator dispatch)

## Design Patterns & Rationale

**Event-driven IR instrumentation**: Rather than patching the parser or AST, trace hooks into the IR **code generation** phase via callback registration on `events.entry` / `events.returns`. This keeps instrumentation orthogonal to the frontend and allows `-t` flag to control whether hooks are even attached.

**Type-dispatch formatting**: `tracevalue()` recursively switches on `Type::op` (INT, UNSIGNED, FLOAT, POINTER, STRUCT, UNION, ARRAY), emitting appropriate printf format codes and recursively formatting compound types (structs, arrays). This mimics a type-directed code generator.

**Format string accumulation**: `appendstr()` dynamically expands a single heap-allocated buffer (`fmt`) as the format string grows, avoiding repeated allocations. The buffer pointers (`fp`, `fmtend`) track current position and limit.

**Per-call static counters**: Each traced function gets a unique static counter (`genident(STATIC, ...)`) incremented on each call, allowing trace output to number invocations sequentially—useful for matching entry/exit pairs in logs.

**Frame-local numbering**: `frameno` (local auto variable) stores the counter value for each frame, enabling trace output to uniquely identify which invocation is returning.

## Data Flow Through This File

1. **Init phase** (`trace_init`): 
   - Scans command-line for `-t[printer]` flag
   - If found: creates a symbol for the printer function (`printf` by default)
   - Registers `tracecall` callback on `events.entry` IR event
   - Registers `tracereturn` callback on `events.returns` IR event

2. **Per-function-entry** (`tracecall`):
   - Creates static counter for this function
   - Formats `"funcname#<counter>(<param1>=<val1>, ...) called\n"` into `fmt` buffer
   - Recursively calls `tracevalue()` for each parameter to append format codes and IR argument nodes
   - Calls `tracefinis()` to emit the printf call

3. **Per-function-return** (`tracereturn`):
   - Formats `"funcname#<frameno> returned [<val>]\n"` (including return value if not void)
   - Calls `tracefinis()` to emit printf

4. **Finalization** (`tracefinis`):
   - Terminates format string
   - Creates a string constant from `fmt`
   - Appends the format-string symbol as final (rearmost) argument to the IR argument tree
   - Calls `walk(calltree(...))` to emit the complete printf call IR node(s)
   - Resets state for next trace instrumentation

## Learning Notes

**LCC IR as middle-end**: This file demonstrates LCC's IR as a true **intermediate representation**—high-level enough that `tracecall` can programmatically build printf-style calls (with recursive type formatting) but low-level enough to directly emit into codegen. Compare to modern compilers: LLVM IR, GCC's GIMPLE, or JVM bytecode all allow similar IR-level instrumentation.

**Recursive type formatting is idiomatic**: The switch-on-`Type::op` + recursion pattern (for STRUCT, ARRAY, POINTER-to-char) matches how compilers decompose types. Modern engines might use visitor patterns, but the direct switch is typical of 1990s C compiler design.

**No runtime type information**: Trace output is **entirely computed at compile time**—no RTTI, vtables, or reflection needed. All type formatting is baked into the generated printf calls. Contrast with languages like Java/C# that embed type info and formatters at runtime.

**Manual symbol/scope management**: The code directly manipulates symbol tables (`addlocal()`, `genident()`), level counters, and storage classes (STATIC, AUTO, GLOBAL). Modern compiler frontends abstract these via symbol-table manager objects; here they're global state.

## Potential Issues

**Buffer overrun on format string**: If `appendstr()` is called with data that never null-terminates (malformed input), the `while ((*fp++ = *str++) != 0)` loop will overrun. However, all callers in this file pass string literals or controlled buffers, so the risk is low unless integrated incorrectly elsewhere.

**Type name truncation**: `typestring()` generates human-readable names (e.g., `"struct point"`). If a deeply nested or parameterized type produces a very long name, it could overflow the format buffer before the recursive formatting completes. No explicit size guard is present—depends on `appendstr()` expanding on demand.

**Frame number aliasing**: If a function is called recursively, all recursive frames share the same `frameno` local variable (same address). The trace output will show the same frame number for all depths. This is likely by design (to track a specific call's lifetime), but could be confusing in deep recursion or re-entrant code.

**Printer symbol assumptions**: Code assumes the printer function (default `printf`) is already declared or will be linked. If not available, linker will fail. The `printer->defined = 0` suppresses definition-checking, trusting that the symbol will be resolved externally.
