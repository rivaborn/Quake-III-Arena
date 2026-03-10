# lcc/cpp/hideset.c — Enhanced Analysis

## Architectural Role

This file implements **macro-expansion state tracking** for the LCC C preprocessor. Hidesets prevent infinite macro recursion by recording which identifiers have already been expanded in the current expansion context. They are a critical component of the preprocessor's macro-application safety mechanism—every macro invocation adds identifiers to the current hideset, and the preprocessor checks this before re-expanding an identifier. This is central to implementing correct C preprocessor semantics per the C standard.

## Key Cross-References

### Incoming (who depends on this file)
- **lcc/cpp/macro.c** — calls `newhideset()`, `unionhideset()`, `checkhideset()` to manage hidesets during macro argument substitution and expansion phases
- **lcc/cpp/cpp.c** — top-level preprocessor driver that initializes hidesets via `iniths()` at startup
- **lcc/cpp/lex.c** or token processor — likely checks `checkhideset()` before expanding macro identifiers

### Outgoing (what this file depends on)
- **lcc/cpp/cpp.h** — provides `Nlist` struct definition (symbol table entry for macro names)
- **Standard C library** — `malloc()`, `realloc()`, `memmove()`, `fprintf()`
- **lcc utility layer** — `domalloc()` (arena allocator) and `error()` (fatal error handler)

## Design Patterns & Rationale

**1. Index-based referential transparency**  
Hidesets are stored in a global `hidesets[]` array and referred to by integer indices, not pointers. This allows:
- Hidesets to be passed/stored/compared without deep-copying the underlying `Nlist*` arrays
- Safe serialization/transmission if needed (unlikely in a preprocessor, but architecturally sound)

**2. Sorted insertion with linear scan**  
`inserths()` maintains **sorted order** across `Nlist*` pointers (ascending address order via `*shs < np` pointer comparison). This enables:
- O(n) deterministic search in `checkhideset()` — no hash table overhead
- Deduplication: if `*shs == np` already exists, `np` is not re-inserted
- Canonical representation: two hidesets with the same elements in the same order will have identical memory layout

**3. Lazy hideset deduplication**  
`newhideset()` first scans all existing hidesets to find an exact match before allocating a new one (lines 47–51). This reduces memory fragmentation and enables fast equality checks via pointer identity.

**4. Hard size limit with silent failure**  
If a new hideset would exceed `HSSIZ` (32 elements), `newhideset()` returns the **original hideset index unchanged** (line 53). This is a degradation, not a failure—the preprocessor continues but may not correctly prevent all infinite recursion if hidesets overflow. The rationale: stack-based macro expansion depths rarely exceed 32 in practice.

**5. Dynamic array growth**  
The `hidesets[]` global array grows by factor `3/2 + 1` (geometric growth) when capacity is exhausted, standard memory pool practice.

## Data Flow Through This File

```
Input:  macro identifier (Nlist *np), current hideset index (int hs)
   ↓
Process:
   1. checkhideset() — O(n) linear search for membership
   2. newhideset() — build candidate array, deduplicate against existing hidesets
      └─> inserths() — sorted insertion, avoiding duplicates
   3. unionhideset() — merge two hidesets via repeated newhideset() calls
Output: new hideset index (int) or original index if overflow
```

**Key state transitions:**
- **Expansion start**: Create new hideset by adding current macro name to existing set → `newhideset(hs, current_macro)`
- **Nested macro**: Union the parent's and argument's hidesets → `unionhideset(hs1, hs2)`
- **Prevent recursion**: Before expanding identifier, check `checkhideset(hs, np)` → skip if true

## Learning Notes

**For a developer studying this preprocessor:**

1. **Canonical symbol table integration** — `Nlist` pointers serve as globally-unique identifiers; no string comparisons needed. This is idiomatic to hand-written C compiler tooling (pre-hash-table era).

2. **Pointer-address-based ordering** — Sorting hidesets by `Nlist*` address creates a canonical order independent of identifier name. This is a micro-optimization for deterministic deduplication but is brittle if the symbol table is ever relocated.

3. **No balanced tree** — A modern implementation might use a hash set or tree for O(1)–O(log n) membership tests, but the fixed 32-element limit makes O(n) linear search acceptable for the expected case.

4. **Global mutable state** — The `hidesets` array is a singleton, global to the entire preprocessor session. This reflects the monolithic architecture of hand-written compiler frontends (no context structs).

5. **Connection to macro algorithm** — This file implements only the **data structure**; the macro-expansion logic that *uses* hidesets (e.g., "don't expand `X` if `X` is in the current hideset") lives elsewhere (likely `macro.c`). This is a clean separation of concerns.

## Potential Issues

- **Silent overflow (line 53):** If a hideset reaches 32 entries, further additions return the original hideset. A macro expansion nesting deeper than 32 levels might then incorrectly expand an already-expanded identifier, causing infinite recursion. In practice, this is extremely rare and acceptable as a hard limit for robustness.

- **Pointer-address-based sorting:** If `Nlist` symbols are ever reallocated (arena compaction, symbol table resize), pointer order changes, potentially breaking deduplication. Code should add assertions to verify stability.

---

**Role in broader toolchain:** This hideset system is a **compile-time safety mechanism** for the offline LCC compiler. It has no runtime-engine role; its correctness is essential only for the QVM bytecode generation pipeline during level/mod compilation.
