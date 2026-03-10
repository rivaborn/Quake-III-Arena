# lcc/src/profio.c — Enhanced Analysis

## Architectural Role

This file is part of the **LCC compiler toolchain** (an offline infrastructure component, not runtime engine code). It provides profiling data serialization and query services for analyzing execution traces of compiled QVM bytecode. The `prof.out` format it parses captures per-file execution point counts and function call graphs, enabling post-mortem analysis of bot AI, game logic, and cgame behavior without instrumentation overhead during live gameplay.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler profiler frontend** (implied by file structure; specific callers not visible in provided cross-reference excerpt)
- Part of offline tooling suite alongside `q3map/`, `q3radiant/`, and `bspc/` tools

### Outgoing (what this file depends on)
- Standard C library (`stdio.h`, `qsort`, string operations)
- `c.h` header (from `lcc/src/c.h`) — LCC's own compilation unit definitions
- Memory allocation infrastructure inherited from LCC (`NEW` macro, `newarray`, `string()`, `PERM` arena)

## Design Patterns & Rationale

**Profile-as-data structure:** The code builds an in-memory representation of profile results into a hierarchy:
- **Files** → **Functions** (with execution counts) → **Callers** (call sites)
- Each tier is a linked list sorted by execution metrics (location for functions, count+location for callers)

**Why this design:**
- Reflects the logical containment in source code (file→function hierarchy matches C scoping)
- Sorted traversal enables deterministic output and fast binary-search queries
- Linked-list construction avoids upfront allocation; data-driven by prof.out contents

**Query optimization:** `findcount()` and `findfunc()` cache the last file lookup (`static struct file *cursor`), exploiting temporal locality—profiles typically query consecutive points from the same source file.

**Binary search on execution points:** The `counts` array within each file is sorted by (y, x) coordinates—line and column—enabling O(log n) lookup via `findcount()` despite the 2D coordinate space.

## Data Flow Through This File

1. **Input (prof.out format):**
   - Header: count of files, functions, execution points
   - File table: names keyed by index
   - Function records: name, file index, location (x,y), call count, caller info
   - Point records: file index, location, execution count

2. **Processing in `gather()` (called repeatedly by `process()`):**
   - Parse file list into `filelist` linked list (deduplicating by pointer identity using `findfile()`)
   - Parse function records → call `afunction()` to insert into per-file function list
   - Parse caller edges → call `acaller()` to append to callee's caller list
   - Parse execution points → call `apoint()` to populate per-file count arrays

3. **Output (in-memory data structure):**
   - `filelist`: global linked list of `struct file`
   - Each file has `funcs` (sorted linked list) and `counts` array (later qsorted by location)
   - Each function has `callers` (sorted linked list of call sites)

4. **Query interface:**
   - `findcount(file, x, y)`: binary-search execution count at source location
   - `findfunc(name, file)`: linear-search function call count

## Learning Notes

- **LCC's memory model:** Uses arena allocation (`PERM`) for offline tools; no free/cleanup overhead matters here
- **Pointer-based string interning:** The code uses `p->name == name` (pointer equality, not `strcmp`) because strings come from a shared pool via `string()` function; this is a classic compiler toolchain pattern
- **Two-phase sorting:** Functions are sorted during insertion (`afunction`), but execution points are collected then batch-sorted at end (`process`); trade-off between insertion cost and query frequency
- **Linked-list manipulation idiom:** The `for (r = &ptr; *r && ...; r = &(*r)->link)` pattern (pointer-to-pointer iteration) is idiomatic in C for sorted insertion without special cases
- **Modern contrast:** A modern profiler might use hash tables, but linked lists + binary search over sorted arrays reflect 1990s compiler toolchain constraints

## Potential Issues

- **1-indexed file references:** The `afunction()` and `apoint()` calls use `files[f-1]` to convert 1-indexed file numbers from prof.out to 0-indexed array; off-by-one errors would be silent if file counts don't match
- **Pointer-identity string dedup:** The `p->name == name` comparisons assume the same string object is reused; different interning could break function lookup
- **No bounds checking on coordinates:** `findcount()` assumes x/y are within valid ranges; malformed prof.out with out-of-bounds locations could cause false negatives
- **Unbounded file count:** `assert(nfiles < NELEMS(files))` uses a fixed-size stack array (64 files); profiles with more files will silently truncate
