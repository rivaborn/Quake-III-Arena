# code/bspc/l_mem.h — Enhanced Analysis

## Architectural Role

This header provides the custom memory allocation interface for **BSPC**, an offline batch-processing tool (not runtime engine code). BSPC compiles BSP maps into AAS navigation data; as a standalone command-line utility, it has no per-frame lifecycle or streaming constraints, allowing a simple fixed-allocation strategy. The memory subsystem is isolated from the runtime engine's dual-hunk architecture (in `qcommon`), reflecting BSPC's role as a separate tool that can reuse botlib's AAS pipeline without runtime overhead.

## Key Cross-References

### Incoming (who depends on this file)
- All BSPC source files throughout `code/bspc/` call `GetMemory`, `GetClearedMemory`, and `FreeMemory` for heap allocation
- Files like `aas_create.c`, `aas_store.c`, `aas_file.c` allocate temporary and persistent AAS geometry structures
- Debug build callers receive the stringified allocation size via macro injection when `MEMDEBUG` is enabled

### Outgoing (what this file depends on)
- Implementation entirely self-contained in `code/bspc/l_mem.c` (no external subsystem calls)
- No dependencies on `qcommon`, renderer, or runtime engine subsystems
- No includes in this header itself

## Design Patterns & Rationale

**Compile-time debug mode** (conditional macro substitution):
- When `MEMDEBUG` is undefined (production), calls are direct function pointers with minimal overhead
- When defined, macros inject `__FILE__` and `__LINE__` metadata for leak detection without adding parameters to production code
- The `#undef MEMDEBUG` at line 28 indicates debug tracing is **disabled by default** in release builds

**Minimal interface**:
- Only `alloc` / `alloc-zero` / `free` / `query` primitives — no resizable pools, arenas, or defragmentation
- Appropriate for batch tools: BSPC processes one map, allocates structures, compiles, frees, exits. No streaming or runtime reallocation needed

**Signature inconsistency** (`int` vs `unsigned long`):
- Minor design artifact: `GetClearedMemory(int size)` vs `GetMemory(unsigned long size)` suggests these functions evolved independently or from different libraries

## Data Flow Through This File

1. **Compilation phase**: BSPC tool binary is built; `l_mem.h` is included by all compilation units needing allocation
2. **Map compilation**: When processing a `.bsp` file:
   - Map/BSP data structures allocated via `GetMemory` / `GetClearedMemory`
   - AAS areas, reachability data, and clusters allocated progressively
   - Temporary structures (`aas_create.c` temp faces/nodes) use same allocator
3. **Finalization**: `FreeMemory` releases all blocks; `TotalAllocatedMemory` confirms final cleanup
4. **Output**: Compiled `.aas` file written; process exits (all pointers invalidated)

No inter-frame state or garbage collection — straightforward lifetime matching.

## Learning Notes

**Why BSPC has its own memory layer**:
- BSPC is a **standalone offline tool**, not linked into the runtime engine
- botlib also has `l_memory.c`, but BSPC's version exists because BSPC is compiled independently
- BSPC *does* reuse botlib's AAS geometry algorithms via `code/bspc/be_aas_bspc.c`, which adapts botlib's AAS code to use BSPC's allocator (via `AAS_InitBotImport` providing import struct)

**Idiomatic to this era**:
- Modern engines use typed allocators (arena, pool, growing heap), custom alignment, and fragmentation profiling
- Q3's approach is bare-bones: single malloc-like interface, no segregation by lifetime or access pattern
- The opt-in debug metadata is a pragmatic early 2000s compromise — no overhead unless `-DMEMDEBUG` flag is set during compilation

**Batch-tool design**:
- No real-time constraints; memory coherence is irrelevant
- No virtual memory/paging considerations (all allocations fit in process heap)
- No per-entity or per-frame pools — just a global free-list underneath

## Potential Issues

None clearly inferable from this header alone. The signature inconsistency (`int` vs `unsigned long`) is minor and unlikely to cause problems in practice (both are bounds-checked in implementation). The disabled debug mode means allocation leaks in BSPC are **only detectable if the code is recompiled with `-DMEMDEBUG`**, which may go unnoticed in standard release builds.
