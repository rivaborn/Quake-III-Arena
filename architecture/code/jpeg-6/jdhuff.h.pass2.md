# code/jpeg-6/jdhuff.h — Enhanced Analysis

## Architectural Role

This header is a micro-library within the renderer's JPEG texture loading pipeline. The renderer loads JPEG textures via `tr_image.c` → `jload.c` (IJG entry point) → `jdhuff.c`/`jdphuff.c` (sequential/progressive decoders). This file provides the **Huffman decoding infrastructure** shared by both decoder variants—the canonical code tables, the bit-level extraction machinery, and the fast lookahead optimization that makes Huffman decode practical at texture-load time (~95% of codes hit the 8-bit lookahead cache). The entire JPEG codec is confined to this vendored `code/jpeg-6/` directory; it has no bidirectional dependencies with the rest of the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** (indirectly): The renderer's image loader calls `jload.c` (IJG public API), which instantiates the decoders.
- **`code/jpeg-6/jdhuff.c`**: Sequential Huffman decoder—direct consumer of `d_derived_tbl`, `BITREAD_*` macros, `CHECK_BIT_BUFFER`, `GET_BITS`, `HUFF_DECODE`, and out-of-line `jpeg_huff_decode`.
- **`code/jpeg-6/jdphuff.c`**: Progressive Huffman decoder—same set of consumers.
- **`code/jpeg-6/jdapimin.c` / `jdapistd.c`**: IJG public API entry points that initialize decoders and call into `jdhuff.c`/`jdphuff.c`.

No runtime engine code (server, game VM, client, botlib, etc.) directly references this file.

### Outgoing (what this file depends on)
- **`code/jpeg-6/jpeglib.h`**: Type definitions (`j_decompress_ptr`, `JHUFF_TBL`, `JOCTET`, `boolean`, `INT32`, `UINT8`, `JPP()` macro for function declarations).
- **`code/jpeg-6/jdhuff.c`**: Defines the three extern functions declared here (`jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`).
- **`code/jpeg-6/jdphuff.c`**: Also calls these three externs (progressive variant of the same machinery).

All data flows within the vendored `jpeg-6/` subtree; no outbound calls to qcommon, renderer, or platform layer.

## Design Patterns & Rationale

### 1. **Two-Level Fast/Slow Dispatch**
The `HUFF_DECODE` macro implements a **lookahead fastpath + slow fallback**:
- **Fast path** (~95% of cases): 8-bit lookahead directly indexes `look_nbits[1<<8]` and `look_sym[1<<8]` lookup tables. Zero branching, zero loops—pure table lookup and bit shift.
- **Slow path** (~5% cases): Codes longer than 8 bits; invoke out-of-line `jpeg_huff_decode` which walks the canonical min/max tables manually. Allows suspension on buffer underrun.

This pattern is still standard in modern codecs (H.265, AV1) because it dominates execution time—Huffman decode is the single biggest CPU consumer in JPEG decode.

### 2. **Register-Level State Handoff (bitread_perm_state ↔ bitread_working_state)**
Why two separate state structures?
- **`bitread_perm_state`**: Persistent across MCU boundaries. Holds `get_buffer`, `bits_left`, `printed_eod` in the struct.
- **`bitread_working_state`**: Per-MCU working copy. Same fields **cached in registers** (`BITREAD_LOAD_STATE` macro), plus source/byte-count/marker fields.

Rationale: On 1990s CPUs (Pentium, Alpha, PowerPC), keeping hot variables in registers was critical. The `register` keyword hints to the compiler to keep `get_buffer` and `bits_left` in registers throughout the inner decode loop. The `BITREAD_LOAD_STATE`/`BITREAD_SAVE_STATE` macros perform the hand-off at MCU boundaries, amortizing register spillage cost.

Modern CPUs make this less critical (registers are now managed by the compiler), but the pattern shows **micro-optimization discipline** for streaming bitwise operations.

### 3. **Suspension Protocol**
Both `jpeg_fill_bit_buffer` and `jpeg_huff_decode` return `boolean` / `int` to signal suspension on buffer underrun:
- On `FALSE` return, decode halts; the persistent state is saved; the source manager refills the buffer asynchronously (e.g., from disk).
- On next frame, `BITREAD_LOAD_STATE` resumes from the saved state.

This enables **progressive decode and stream processing** without buffering entire images—crucial for embedded systems or streaming video.

### 4. **Canonical Huffman Code Tables**
The `d_derived_tbl` struct holds `mincode[17]` and `maxcode[17]` arrays—a canonical representation:
- `mincode[k]` = smallest code value of length `k` bits (left-aligned in the `INT32` bit space).
- `maxcode[k]` = largest code value of length `k` bits (or -1 if none).
- `maxcode[17]` is a sentinel to ensure the slow-path loop terminates.

**Why canonical?** JPEG specifies Huffman tables as symbol→code-length mappings (DC/AC tables in the `DHT` marker). The encoder assigns code values in increasing order per length. A canonical decoder pre-computes min/max to enable binary search or range-check dispatch (used in `jpeg_huff_decode`).

## Data Flow Through This File

```
JPEG Bitstream (bytes from source manager)
    ↓
jpeg_fill_bit_buffer()
    • Reads JOCTET bytes into get_buffer (INT32 32-bit register)
    • Tracks bits_left (number of unused bits)
    • Inserts dummy zeros at EOI for graceful termination
    ↓
CHECK_BIT_BUFFER() macro (inline, checks bits_left >= nbits)
    ↓
GET_BITS() / PEEK_BITS() / DROP_BITS() macros (inline bit masking)
    ↓
HUFF_DECODE() macro
    ├─ Fast path (95%): look_nbits[next 8 bits] → decoded symbol
    └─ Slow path (5%):  jpeg_huff_decode() walks mincode/maxcode/valptr
        ↓
        Decoded symbols (0–255, or -1 on suspend)
        ↓
        Back to caller (jdhuff.c, jdphuff.c) as MCU coefficient values
```

**State persistence across MCU boundaries:**
- After each MCU, `BITREAD_SAVE_STATE` writes `get_buffer` and `bits_left` back to `bitread_perm_state`.
- On suspension, the source manager refills; `BITREAD_LOAD_STATE` resumes the next MCU.

## Learning Notes

### Idiomatic to This Era (1990s C Codec Development)

1. **Macro-heavy for inlining**: Modern C compilers have `inline` and aggressive IPA, but in the early 1990s, macros were the only reliable way to force inlining of hottest paths. The `BITREAD_STATE_VARS`, `CHECK_BIT_BUFFER`, `GET_BITS` macros replace what would now be `static inline` functions.

2. **Register keyword as a hint**: The `register` keyword on `get_buffer` and `bits_left` is a strong hint to the compiler. Modern compilers ignore this (they profile and decide), but it documents intent: "these variables live on the CPU, not in memory."

3. **Lookahead cache optimization**: This technique predates modern entropy coders (CABAC, range coding), which are adaptive. Static Huffman tables allowed offline computation of the 8-bit lookahead cache—a win that still applies to JPEG and PNG.

4. **Suspension/resumption for streaming**: This pattern became standard for progressive image formats and real-time video codecs. The ability to pause mid-MCU and resume later enabled early MPEG streaming systems.

5. **Canonical Huffman representation**: Still used in H.264, H.265, VP9, AV1 because it avoids storing full codebook tables and enables efficient range-check decode.

### Modern Equivalents

- **Lookahead optimization** → SIMD Huffman decode (e.g., Intel's fast H.264 decoder), or specialized hardware (Nvidia NVDEC).
- **Macro inlining** → `static inline` functions with compiler-driven inlining analysis.
- **Register hints** → None; the compiler manages all registers via register allocation.
- **Suspension protocol** → Streaming interfaces in modern video APIs (e.g., `VkVideoDecodeAV1SessionCreateInfoKHR` supports incremental frame decoding).

### Connections to Game Engine Concepts

**Texture loading** is the critical path in a game engine's main-thread startup or streaming. Quake III uses deferred texture loading via `BeginStreamedFile` / `StreamedRead` (see `code/client/client.h` and `code/win32/win_main.c`), which pairs well with JPEG's suspension protocol—the renderer can kick off a load, interleave physics/AI, and resume when data arrives. The lookahead optimization ensures Huffman decode doesn't become a bottleneck during those interleaved frames.

## Potential Issues

**None obvious from code inspection alone**, but architectural context suggests:

1. **No bounds checking on look_nbits/look_sym indexing**: The `PEEK_BITS(HUFF_LOOKAHEAD)` can produce any 8-bit value (0–255), and the macro directly indexes `look_nbits[look]` without validation. **Risk**: Malformed Huffman table pointers could cause out-of-bounds access. The JPEG spec and `jpeg_make_d_derived_tbl` must guarantee the lookup tables are always 256 entries. *Mitigation*: JPEGs are usually trusted (from disk or trusted servers), but a fuzzing harness on `jload.c` would catch this.

2. **Register keyword on structures**: The `bitread_working_state *state` parameter is not `register`-qualified, so the macro de-references it repeatedly (`state.get_buffer`, `state.next_input_byte`, etc.). On architectures with slow memory (e.g., Alpha, early PowerPC), this could add latency. Modern CPUs with speculative load pipelining hide this. *Non-issue in practice on modern targets*, but shows the code was tuned for 1995-era hardware.

3. **No explicit overflow protection on bit shifts**: The `#define GET_BITS(nbits)` macro assumes `nbits` is in range [1, 15] per JPEG spec. Passing `nbits > 31` would yield undefined behavior (left-shift out of bounds). **Risk**: Malformed input or misuse could crash. *Mitigation*: All call sites in `jdhuff.c` / `jdphuff.c` respect this range; no fuzzing risk unless the macros are exposed as public API (they're not).
