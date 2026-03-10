# code/jpeg-6/jchuff.c — Enhanced Analysis

## Architectural Role

`jchuff.c` implements the **baseline JPEG entropy encoder**, one half of a two-encoder design (the other being `jcphuff.c` for progressive JPEG). This module sits between the DCT coefficient quantizer and the output file/buffer manager, implementing the final compression stage. Within Quake III's renderer, JPEG texture loading routes through the libjpeg-6 decompression path, but this encoder is used when capturing gameplay screenshots or creating custom textures during offline tools/map compilation. The module demonstrates IJG's suspension-aware architecture, critical for the engine's streaming/demand-paced output where the destination buffer can become full mid-frame.

## Key Cross-References

### Incoming (who depends on this file)

- **`jcapimin.c`** (IJG baseline encoder init) — calls `jinit_huff_encoder()` during compressor setup; wires `start_pass_huff` to `entropy→pub.start_pass` vtable
- **`jcphuff.c`** (progressive JPEG encoder) — **shares** `jpeg_make_c_derived_tbl()` and `jpeg_gen_optimal_table()` via `jchuff.h` extern declarations; runs parallel to this module with identical table-building logic but different MCU encoding strategy
- **Renderer/offline tools** — indirectly; any code that needs to write JPEG output (e.g., screenshotting, custom texture baking) links the full `libjpeg-6` stack, including this encoder

### Outgoing (what this file depends on)

- **`jpeglib.h` / `jinclude.h`** — core types (`j_compress_ptr`, `JHUFF_TBL`, `c_derived_tbl`), memory manager vtable (`cinfo→mem→alloc_small`), error macros (`ERREXIT`, `ERREXIT1`)
- **`jchuff.h`** — declares `jpeg_make_c_derived_tbl()` and `jpeg_gen_optimal_table()` for shared use with `jcphuff.c`
- **`jpeg_natural_order` array** — defined elsewhere (typically `jutils.c`); maps 8×8 block indices to zigzag traverse order per JPEG Sec. F.1.2.1
- **`jpeg_destination_mgr` vtable** — called indirectly via `dump_buffer()` → `(*dest→empty_output_buffer)(cinfo)` when output buffer fills
- **Symbol frequency tables** — (if `ENTROPY_OPT_SUPPORTED`) allocated from `JPOOL_IMAGE` and filled during statistics pass, then consumed by `jpeg_gen_optimal_table()` to generate optimal DHT entries

## Design Patterns & Rationale

### Suspension-Aware State Snapshots
The `savable_state` + `working_state` pattern is IJG's elegant answer to **output suspension**: when `emit_byte()` fills the destination buffer and `dump_buffer()` returns `FALSE`, the MCU encoding can safely abort without corrupting persistent state. Only on complete MCU success does `ASSIGN_STATE()` commit working variables back to `entropy→saved`. This allows the caller to:
1. Refill the output buffer
2. Resume the next `encode_mcu_huff()` call without re-entering the same MCU

This is critical for **framerate stability** — the engine can pace buffer drains (e.g., flush to disk) without halting the main loop.

### Dual-Path Routing (Statistics vs. Encoding)
`start_pass_huff()` branches on `gather_statistics`:
- **TRUE** (statistics mode): allocate/zero count arrays, wire `encode_mcu_gather` and `finish_pass_gather` → after scan, `jpeg_gen_optimal_table()` consumes counts to build optimal DHT
- **FALSE** (encoding mode): build `c_derived_tbl` lookup tables from predefined DHTs, wire `encode_mcu_huff` and `finish_pass_huff`

This supports **two-pass compression** (scan image for symbol frequencies → generate optimal tables → re-encode with optimized tables), if `ENTROPY_OPT_SUPPORTED` is defined.

### Fast Huffman Lookup
`jpeg_make_c_derived_tbl()` pre-computes two flat arrays per Huffman table:
- `ehufco[256]` — code value indexed by symbol
- `ehufsi[256]` — code size (bits) indexed by symbol

Avoids tree traversal in the hot path (`emit_bits()`); symbols with no code get `ehufsi[s]=0`, which `emit_bits()` flags as a fatal error if attempted.

### Bit-Level Output with Stuffing
`emit_bits()` maintains a 24-bit accumulator (`put_buffer`) to shift incoming codes and flush whole bytes. When an emitted byte is `0xFF`, a `0x00` stuffing byte follows (JPEG spec), handled transparently by the `emit_byte()` macro. This avoids false sync markers.

## Data Flow Through This File

1. **Init**: `jinit_huff_encoder()` allocates the encoder object, NULLs all table pointers.
2. **Start-of-scan**: `start_pass_huff(gather_statistics)` either:
   - Allocates stat arrays (gather mode), or
   - Calls `jpeg_make_c_derived_tbl()` for each table to populate `ehufco`/`ehufsi` (encode mode)
3. **Per-MCU**:
   - `encode_mcu_huff()` copies entropy state to `working_state`, calls `encode_one_block()` for each 8×8 block
   - `encode_one_block()` emits DC difference as (nbits_symbol, nbits_value) via `emit_bits()`, then AC symbols as (run/nbits_symbol, nbits_value) pairs
   - `emit_bits()` shifts bits into 24-bit buffer, flushes `≥8` bits to `emit_byte()`, which writes to output and calls `dump_buffer()` if needed
   - On suspension (FALSE return), state is rolled back; on success, state commits to `entropy→saved`
4. **Restart**: If `restart_interval > 0`, `emit_restart()` flushes bit buffer, writes `0xFF 0xD0+n` restart marker, resets DC predictions
5. **End-of-scan**: `finish_pass_huff()` flushes remaining bits (fill with 0xFF... to complete final byte)

---

## Learning Notes

### Why This Pattern Matters to Modern Engines
- **Suspension-aware design**: Today's game engines (Unreal, Unity) use async task graphs; jchuff's explicit suspension handling was ahead of its time for **composable, pausable I/O**.
- **Dual-encoder branching**: Shows how to support multiple compression profiles (baseline vs. progressive) with **shared core logic** (`jpeg_make_c_derived_tbl`). Modern engines do this with trait systems or capability flags.
- **Inline SIMD readiness**: The bit-shifting and byte-packing loops in `emit_bits()` are hand-optimized for CPU efficiency; modern compilers can auto-vectorize these, but the original design was tuned for 1990s compilers.

### Idiomatic to 1990s JPEG Encoding
- **Hand-rolled bit accumulator**: Before SIMD was mainstream, bit-level operations were carefully tuned. Today, AVX2/NEON provides native bit-shuffle, but this code is portable.
- **Table-driven symbol lookup**: Every symbol → (code, size) is O(1) array dereference. This was the standard before Huffman-tree JIT compilation became feasible.
- **Restart markers for error resilience**: The periodic `0xFF 0xDn` markers allow decoders to resync if packets are lost; critical for streaming over unreliable networks (pre-HTTPS era).

### Architectural Debt / Quirks
- **Compiler workaround (`ASSIGN_STATE` macro)**: Line 45–54 conditionally compiles manual struct member assignment for broken C compilers. This was necessary for portability in the mid-1990s but is dead code on modern systems.
- **Magic number `0xF0` and `0x00`**: ZRL (zero-run-length) and EOB (end-of-block) symbols are hardcoded rather than computed from Huffman table structure. This works because they're standardized in JPEG, but it couples the encoder to the JPEG standard tightly.

## Potential Issues

1. **Integer overflow in bit accumulation** (lines 308–310): `put_bits += size` could theoretically overflow if a caller passes `size > 32768`, but the contract states `size ≤ 16` per JPEG section F.1.2, and `emit_bits()` asserts `size != 0`. Assumption is documented but not runtime-checked.

2. **Buffer underrun in `huffsize[p]` traversal** (lines 228–244): If `lastp` is computed correctly from loop count, but bit array indexing `htbl→bits[l]` could be malformed (e.g., all zeros), `lastp` could be 0, causing the second loop to never execute. No assertion validates `lastp > 0` or table well-formedness.

3. **Silent symbol loss if code length > 16** (line 285): JPEG limits code lengths to 16 bits; `jpeg_gen_optimal_table()` is supposed to trim longer codes, but if called with a malformed table that has 17+ bit symbols, this encoder will accept and store them, leading to non-standard output that may not decode. No validation.

4. **Restart counter logic** (lines 490–495): If `restarts_to_go` wraps (becomes 0xFFFFFFFF after decrement), undefined behavior. Assumption is that `restart_interval` is set correctly and restart cycles never exceed 8 consecutive MCUs per restart marker. Safe in practice but relies on caller discipline.
