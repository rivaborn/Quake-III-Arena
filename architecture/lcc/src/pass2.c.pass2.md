# lcc/src/pass2.c — Enhanced Analysis

## Architectural Role

`pass2.c` is the **IR-to-backend interface layer** of the LCC C compiler, which is the **sole compiler toolchain** used to compile Quake III's QVM bytecode modules (cgame, game, ui, q3_ui). This file bridges the compiler's intermediate representation (produced by pass1) to a pluggable backend system, translating generic IR nodes (types, symbols, code trees) into backend-specific assembly or bytecode. As a tool-phase component, it operates entirely offline during asset compilation and has no runtime presence in the shipped engine.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compilation pipeline** (`lcc/src/main.c`, `pass1.c`): Calls `interface()` dispatcher for each IR item during backend invocation
- **Backend abstraction** (`rcc.h`): Consumes backend vtable (`Interface IR`) to emit final code; backend exports via `GetBotLibAPI` pattern (in LCC context)

### Outgoing (what this file depends on)
- **IR definitions** (`rcc.h`): Enum kinds (`rcc_Export_enum`, `rcc_Type_enum`, `rcc_Function_enum`, etc.)
- **LCC type/symbol infrastructure** (from pass1): btot (base-to-type), constant(), findlabel(), addlocal(), intconst()
- **Backend vtable** (`IR->export`, `IR->global`, `IR->defconst`, `IR->function`): Abstract interface to code emission
- **Scope/codegen globals** (external): `cfunc`, `labels`, `level`, `codelist`, `swap` (endianness), `voidtype`, `inttype`, `voidptype`

## Design Patterns & Rationale

**Visitor over IR nodes** (`visit()` function):  
Recursively descends `rcc_node_ty` tree, reconstructing engine-level `Node` AST from flattened IR. Implements tree pattern-matching via `rcc_*_enum` discriminators — idiomatic for compiler IRs of this era (pre-SSA, AST-like IR).

**UID (unique identifier) indirection**:  
Types and symbols are referenced by integer UIDs rather than direct pointers, enabling:
- Serialization/transmission of IR without relocatable pointers
- Lazy materialization via `uid2type()` / `uid2symbol()` with memoization in `itemmap[]`
- Recursive type support (structs containing pointers to themselves) via early `itemmap` assignment before recursive `uid2type()` calls

**Dispatcher table (`doX[]`)**:  
Empty dispatch table with all entries zeroed, then populated statically. This is likely a **placeholder for dynamic dispatch** or **selective backend support** — not all `rcc_*` kinds need handlers in all backends.

**Lazy symbol/type materialization**:  
`uid2type()` and `uid2symbol()` check `itemmap[uid]` first; on cache miss, deserialize from `items[]` array, populate `itemmap`, and free the source structure. This amortizes deserialization cost and keeps memory footprint low during backend traversal.

## Data Flow Through This File

1. **IR ingestion** (Frontend → `pass2.c`):
   - LCC's pass1 produces `rcc_interface_ty` items (Export, Global, Function, etc.)
   - Items queued for dispatch via `interface()` dispatcher

2. **Symbol/type materialization**:
   - `uid2symbol()` / `uid2type()` lazily deserialize from UID references into engine-level `Type` / `Symbol` objects
   - `itemmap[]` acts as both cache and arena

3. **Code tree visitation**:
   - `doForest()` extracts node list from Interface item
   - `visit()` recursively transforms `rcc_node_ty` → `Node` AST
   - Each node kind mapped to backend operator codes (CNST, ASGN, CALL, etc.)

4. **Backend code emission**:
   - Completed `Node` trees and symbol/type descriptors passed to backend vtable
   - `IR->function()`, `IR->defconst()`, etc. emit target code
   - Backend-specific code emission is completely opaque to `pass2.c`

## Learning Notes

**Why separate QVM compilation?**  
Quake III's VM architecture requires deterministic bytecode across all platforms (to enforce server-side rule consistency). Using a dedicated LCC compiler (rather than native C toolchains) ensures **portable, reproducible QVM builds**. This is distinct from modern engines that compile shader/script code at load-time.

**Why UID indirection?**  
In the 2000s, compiler IR was often linearized for transmission/caching (relevant for distributed builds or pre-compiled IR databases). UIDs allow IR to remain serializable without pointer fixups. Modern compilers (LLVM, GCC) use in-memory AST graphs directly.

**Recursive type handling**:  
Notice the pattern in `uid2type()` for `rcc_STRUCT_enum` / `rcc_UNION_enum`:
```c
itemmap[uid] = ty;  /* recursive types */
```
This is assigned **before** recursing into fields, enabling self-referential types (e.g., linked-list nodes). Modern type systems use explicit recursive type constructors; here it's implicit in the memoization.

**Code forest vs. tree**:  
`doForest()` collects multiple root-level trees into a single `Gen` pseudo-node. This likely reflects an IR that separates declarations (forests) from control flow (trees within functions).

## Potential Issues

- **Empty dispatch table (`doX[]`)**: All entries are zeroed; if the backend calls an unsupported kind, it crashes with a NULL dereference. This may be intentional (fail-fast for unsupported IR), but no bounds check or assertions guard against out-of-range kinds.
- **Scope assumptions in `uid2symbol()`**: Asserts `p->scope != CONSTANTS && p->scope != LABELS`, but no explicit error path if violated — would silently construct invalid symbols.
- **Double-free risk**: `uid2symbol()` and `uid2type()` both `free(items[uid])` after materialization; if a UID is visited twice (should not happen with correct IR), double-free would occur. Relies on caller discipline.
