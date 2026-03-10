# libs/jpeg6/jdhuff.cpp — Enhanced Analysis

## Architectural Role

This file implements the entropy decoding layer of the JPEG decompression pipeline, responsible for the lowest-level bitstream parsing. It sits in the **vendored IJG libjpeg-6 library**, which is consumed **exclusively by the Renderer** (`code/renderer/tr_image.c`) for texture asset loading at initialization time, never in the hot per-frame rendering path. This isolation means the file has zero impact on engine subsystem coupling or runtime performance—it is pure external codec infrastructure that the renderer treats as a black box.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer** (`code/renderer/tr_image.c`): Calls the public libjpeg API (not this file directly); decoder created via `jpeg_create_decompress`
- **Other libjpeg modules** (`jdphuff.c`, `jdapistd.c`): Call public functions defined here:
  - `jpeg_fill_bit_buffer`: Refill bit buffer from input stream
  - `jpeg_huff_decode`: Decode a single Huffman symbol
  - `jpeg_make_d_derived_tbl`: Build Huffman lookup tables
- Nowhere else in `code/` calls JPEG functions directly; texture loading is entirely renderer-owned

### Outgoing (what this file depends on)
- **libjpeg internal headers** (`jinclude.h`, `jpeglib.h`, `jdhuff.h`): Type definitions, macros, memory allocator interface
- **Memory allocator** (`cinfo->mem->alloc_small`): Standard IJG abstraction (not engine memory)
- **Error/warning macros** (`ERREXIT1`, `WARNMS`): IJG's error handling, not engine's `Com_Error`
- **Bit-buffer I/O abstraction** (`state->cinfo->src->fill_input_buffer`): Source manager, not direct filesystem
- **No dependencies on engine subsystems** (qcommon, server, renderer backend, etc.)

## Design Patterns & Rationale

**1. Input Suspension Support**
The entire file revolves around graceful input suspension. The `bitread_perm_state` / `savable_state` dual structure allows backtracking to MCU boundaries if the source buffer is exhausted mid-block. This was critical in the 1990s for streaming decompression on memory-constrained systems; the comments explicitly document this burden.

**2. Lookahead Optimization Tables**
`jpeg_make_d_derived_tbl` precomputes `look_nbits[256]` and `look_sym[256]` tables to decode up to `HUFF_LOOKAHEAD` bits in a single table lookup, avoiding bit-by-bit loop unrolling. This is a classic speed/space tradeoff for variable-length code decoding.

**3. Marker Stuffing De-escaping**
Lines 299–328 implement the JPEG spec's 0xFF escape mechanism: when a 0xFF byte appears in the bitstream, a following 0x00 is stuffed and must be discarded; any other byte is a restart marker. This is handled inline during bit refilling, not as a separate pass.

**4. Register-Heavy Code**
Extensive use of `register` hints and local variable hoisting (e.g., `next_input_byte`, `bytes_in_buffer` copied from state at function entry, updated locally) reflects 1990s compiler optimization practices. Modern compilers ignore `register`, but the pattern shows intent for tight loops.

## Data Flow Through This File

**Inbound:** Compressed JPEG bitstream → `state->cinfo->src->next_input_byte` (managed by source manager)

**Processing:**
1. `jpeg_fill_bit_buffer`: Accumulate bytes into `get_buffer` (32-bit rolling window), handling marker de-stuffing
2. `jpeg_huff_decode`: Fetch min-bits from buffer, then bit-by-bit read until code matches a Huffman symbol
3. `decode_mcu`: Outer loop assembles an MCU (Minimum Coded Unit, typically 8×8 block) using DC/AC Huffman tables
4. DC coefficient: decoded as differential (difference from prior DC); converted to actual value via `state.last_dc_val[ci]`
5. AC coefficients: run-length encoded (skip count in high nibble, value magnitude in low)
6. Coefficients placed in zigzag order in output block; **caller must zero the block beforehand**

**Outbound:** Raw DCT coefficients (JCOEF) → `MCU_data[blkn][k]` (caller-provided buffer); state updated only on successful MCU completion

## Learning Notes

**1. Idiomatic to this era / different from modern engines:**
- **No malloc during decode**: Static buffers and stack allocation only; the entire state machine fits in ~200 bytes
- **No abstraction creep**: `HUFF_DECODE` macro is inlined everywhere, not factored into functions; performance > maintainability
- **Graceful degradation**: If bitstream is corrupted (marker hit early), stuff zeros and emit a warning; no hard error, image completes
- **Manual SIMD-free bit manipulation**: Bit loops written explicitly; modern decoders would vectorize de-RLE or use SIMD FFTs

**2. Modern contrast:**
- Contemporary engines (Unreal, Unity) often use libvpx, HEIF, or WebP hardware-accelerated decoders
- This file's suspension model is obsolete in streaming contexts (HTTP range requests handle buffer refills)
- The lookahead tables are micro-optimizations; modern branch prediction may make linear scan cheaper

**3. Theoretical grounding:**
- Huffman decoding algorithm (Figure F.16, JPEG spec) is textbook; the code is nearly a direct transliteration
- DC differential encoding (F.2.2.1) exploits locality: consecutive DC values are typically close
- Restart markers (`process_restart`) enable error recovery and parallel decoding in later standards (JPEG2000, WebP)

## Potential Issues

**1. Buffer Overrun on Corrupted Input**
Line 342: `if (l > 16)` catches garbage input, but earlier code at line 327 assumes `htbl->pub->huffval` is valid. A malformed table could cause out-of-bounds dereference. However, this is pre-validated in `start_pass_huff_decoder` (line 102), so risk is low in practice.

**2. Sign Extension Assumption**
The `HUFF_EXTEND` macro (lines 381–391) assumes arithmetic right-shift for sign extension. On platforms with logical right-shift (rare but possible), negative coefficients would corrupt. Modern code would use explicit conditional logic or a lookup table.

**3. Single-Threaded State**
The `huff_entropy_decoder` struct is not thread-safe; concurrent calls to `decode_mcu` on the same instance will corrupt state. The engine's texture loader is single-threaded at load time, so this is not a runtime risk, but it reflects the library's original design (single-pass, sequential decode).
