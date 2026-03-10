# lcc/src/string.c — Enhanced Analysis

## Architectural Role

This file implements the **string interning (pooling) subsystem for the LCC compiler**—a self-contained C compiler used offline to compile Quake 3 game code into QVM (Quake Virtual Machine) bytecode. String interning is a foundational compiler optimization that ensures each unique string literal or identifier is stored exactly once in memory, reducing both memory footprint and enabling fast identity-based string comparison (pointer equality). This is essential during parsing and symbol-table construction where the same identifier appears repeatedly.

## Key Cross-References

### Incoming (who depends on this file)
- **All LCC compiler source files** (`lcc/src/*.c`) that participate in lexical analysis, parsing, and symbol management implicitly call `string()`, `stringn()`, or `stringd()` whenever they acquire new identifiers or string constants from the source being compiled.
- **No runtime engine dependency**: Unlike botlib or renderer subsystems, LCC is a pure offline tool with no link-time relationship to `code/` runtime. The compiled QVM output (cgame, game, ui) uses interned strings only indirectly (as constants baked into the compiled bytecode).

### Outgoing (what this file depends on)
- **`allocate(n, PERM)`** — low-level zone allocator from LCC's memory system; used to reserve chunks (4KB + `len + 1` bytes) when the string pool exhausts its current buffer.
- **`NELEMS()` macro and `PERM` constant** — defined elsewhere in LCC infrastructure; `PERM` denotes permanent (non-freeable) heap allocation typical of a single-pass compiler.
- **C standard library assertions** (`assert(str)`).

## Design Patterns & Rationale

**Hash Table with Chaining (1024 buckets)**
- Uses a fixed-size modulo-power-of-two hash table to avoid dynamic resizing complexity.
- Collision chain via `p->link` pointer handles hash collisions.

**Scatter Array for Hash Distribution**
- The 256-entry `scatter[]` table of pseudo-random 32-bit values is the key technique: rather than naive character-by-character summation, each byte of the input string is mapped through `scatter[*(unsigned char)*end++]` before left-shift accumulation.
- This **avoids poor clustering** that naive hashing would exhibit (e.g., anagrams would hash identically); scatter ensures small changes in input produce large hash changes.

**Chunked Allocation Strategy**
- Rather than allocate one `malloc()` per string, the code reserves a 4 KB buffer and carves individual strings from it.
- Dramatically reduces allocator fragmentation and syscall overhead during parsing, where strings are interned at high frequency.
- Once a chunk is exhausted, a new one is allocated; the old chunk is never freed (hence `PERM` lifetime).

**Deduplication via Full String Comparison**
- After hash lookup, the code performs byte-by-byte comparison (`while (*s1++ == *s2++)`) to confirm a real match, not just a hash collision.
- This correctness-first approach is correct but slightly inefficient; modern compilers sometimes cache lengths or use memcmp.

## Data Flow Through This File

1. **Entry**: Parser calls `string("identifier")` or `stringn(buf, len)` with newly-parsed text.
2. **Hashing**: Left-rotate accumulator, mixing in scattered byte values for even distribution.
3. **Lookup**: Walk collision chain at `buckets[h]` comparing lengths and full strings.
4. **Hit**: Return existing `p->str` pointer immediately (deduplication achieved).
5. **Miss**: Allocate from current chunk (or reserve new 4 KB chunk if needed), copy bytes, append null terminator, insert at head of collision chain, return new pointer.
6. **Exit**: Compiler holds returned pointer in symbol table or constant table; all occurrences of that string share the same memory address.

## Learning Notes

- **Classic compiler pattern**: String interning is ubiquitous in language implementations (Python, Java, many C/C++ compilers). This implementation is textbook—hash table + chaining + scattered hash function.
- **Scatter array idiom**: The pseudo-random lookup table is an elegant way to scramble input bytes without expensive multiplication or prime operations, common in 1990s compiler design where CPUs had slow arithmetic.
- **Memory profiling insight**: A single-pass compiler like LCC never frees strings (PERM allocation), relying on process exit to clean up. This is safe but would be inappropriate for a long-running interpreter (e.g., Python or JavaScript runtimes must free unused interns).
- **No concurrency**: No synchronization primitives; LCC is single-threaded and sequential.

## Potential Issues

- **No overflow protection in hash calculation**: The line `h = (h<<1) + scatter[...]` can overflow silently; some implementations use `(h<<1) ^ scatter[...]` or explicit modulo in the loop to avoid undefined behavior.
- **Scatter array size mismatch**: If the input string contains bytes with all 256 possible values, the scatter array is large but fixed. No bounds check, assuming input is well-formed ASCII/UTF-8.
- **No hash table resize**: With a fixed 1024-bucket table, pathological input (many hash collisions) would degrade to O(n) lookup per string. Production compilers often grow the table dynamically, but for Q3A's modest codebase (game, cgame, UI VMs), this is not a practical concern.
