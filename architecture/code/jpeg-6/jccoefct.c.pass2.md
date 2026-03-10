# code/jpeg-6/jccoefct.c — Enhanced Analysis

## Architectural Role

This file is the **coefficient buffer controller** for the JPEG compression pipeline, sitting between the forward-DCT stage and entropy encoding. It is part of the vendored **IJG libjpeg-6 library**, used in Q3A for JPEG encoding (screenshots, offline map/texture tools). Unlike the renderer's image loading path (`jload.c`), this module is **encoding-side only** and operates in the context of the master JPEG compressor's `j_compress_ptr` state machine.

## Key Cross-References

### Incoming (who depends on this file)
- **Caller:** `jpeg_c_compress` master controller (via indirect method dispatch)
  - Calls `jinit_c_coef_controller()` during compression initialization
  - Calls `coef->pub.start_pass()` (registered as `start_pass_coef`) before each pass
  - Invokes `coef->pub.compress_data()` repeatedly until all iMCU rows consumed (function pointer is one of: `compress_data`, `compress_first_pass`, `compress_output`, depending on pass mode)
- **Context:** Dispatcher is outside libjpeg; in Q3A context, likely called from screenshot/encoding machinery

### Outgoing (what this file depends on)
- **Forward-DCT:** `cinfo->fdct->forward_DCT` — processes sample rows, produces coefficient blocks
- **Entropy Encoder:** `cinfo->entropy->encode_mcu` — consumes MCU blocks, produces compressed bitstream
- **Memory Manager:** `cinfo->mem->access_virt_barray`, `cinfo->mem->request_virt_barray` — allocate/access virtual block arrays for multi-pass buffering
- **Utilities:** `jzero_far`, `jround_up` — memory fill, rounding; error macro `ERREXIT`
- **No direct Q3A dependencies** (part of vendored libjpeg)

## Design Patterns & Rationale

**1. Strategy Pattern (Pass Mode Dispatch)**
- `start_pass_coef()` selects the appropriate work function based on `pass_mode`:
  - `JBUF_PASS_THRU` → `compress_data` (single-pass, no buffering)
  - `JBUF_SAVE_AND_PASS` → `compress_first_pass` (multi-pass: DCT all, buffer, output)
  - `JBUF_CRANK_DEST` → `compress_output` (multi-pass: read from buffer, output)
- This avoids per-frame mode checks; the correct function is selected once per pass.

**2. Virtual Array Abstraction**
- For multi-pass modes, delegates all large block storage to the memory manager's virtual array subsystem (`request_virt_barray`, `access_virt_barray`).
- Allows the JPEG compressor to run on systems with limited RAM by swapping blocks to disk/temp space transparently.
- Single-pass mode uses a small, contiguous MCU-sized buffer to avoid this overhead.

**3. Suspension/Resumption**
- All three work functions return `FALSE` if the entropy encoder stalls (e.g., output buffer full).
- Counters (`iMCU_row_num`, `mcu_ctr`, `MCU_vert_offset`) preserve position; caller retries with same input.
- Single-pass mode re-DCTs stalled MCUs on retry (noted as "a bit inefficient" in comments); multi-pass avoids this by using persistent virtual arrays.

**4. Edge Padding Strategy**
- Right edge: fills right-margin dummy blocks with zero AC entries and replicate the last real block's DC value.
- Bottom edge: creates entire dummy rows with DC values copied from above.
- This **minimizes encoded size** (zeros compress well; repeated DC is more coherent than noise).

## Data Flow Through This File

1. **Initialization (one-time):**
   - `jinit_c_coef_controller()` allocates either a single-MCU buffer (pass-through) or per-component virtual arrays (multi-pass).
   - Registers `start_pass_coef` as the pass initialization method.

2. **Per-Pass Setup:**
   - `start_pass_coef()` resets row counters and selects work function.
   - `start_iMCU_row()` computes how many MCU rows fit in the current iMCU row (1 for interleaved, `v_samp_factor` for non-interleaved).

3. **Single-Pass Flow (`compress_data`):**
   - For each MCU in the current iMCU row:
     - Iterate per-component blocks; call `forward_DCT` on each component's sample block.
     - Fill right/bottom edge dummy blocks (zero padding, DC replication).
     - Call `encode_mcu` to emit encoded data.
   - On suspension, save position and return `FALSE`; caller retries (re-DCT occurs).
   - On completion, increment `iMCU_row_num` and return `TRUE`.

4. **Multi-Pass Flow (`compress_first_pass` → `compress_output`):**
   - **First pass:** DCT all components, store coefficients in virtual arrays, fill dummy blocks **in place**.
   - **Subsequent passes:** read MCU blocks from virtual arrays, construct MCU pointers, call `encode_mcu`.
   - Suspension is safe: virtual arrays persist across retry; no re-DCT needed.

## Learning Notes

**What modern engines do differently:**
- Modern JPEG and video codecs often use **tile-based or streaming DCT** rather than full-image buffering, reducing memory footprint.
- Many now support **hardware acceleration** (GPGPU) for DCT/quantization; libjpeg-6 is pure software.
- **Adaptive quantization** and **perceptual optimization** are common; this file is agnostic to those layers.

**What's idiomatic to this era (1994–1995):**
- Virtual memory abstraction (virtual arrays) reflects the era's resource constraints; modern tools assume sufficient RAM.
- The re-DCT inefficiency on suspension is accepted because suspension is rare in practice.
- **Method pointers** (`coef->pub.compress_data`) are the only polymorphism available in C, predating function tables and VMs.

**Connection to game engine concepts:**
- This is **not** part of the game engine proper; it's a **utility codec library**.
- However, it demonstrates **layered architecture**: DCT, quantization, buffering, and entropy encoding are separate modules with well-defined boundaries.
- The **virtual array abstraction** is analogous to game engine memory managers that defer allocation to a subsystem, enabling swapping and out-of-core techniques.

## Potential Issues

1. **Re-DCT Inefficiency (Noted in Code):**
   - Single-pass mode redoes DCT work on suspension. If the entropy encoder frequently stalls, this could degrade performance noticeably.
   - Mitigation: Most use cases (screenshot saving) have fast output buffers; stalls are rare.

2. **Error Handling Gaps (Minor):**
   - `ERREXIT` is used for buffer mode mismatches, but no validation of `cinfo->num_components` or array bounds in the loops. However, libjpeg assumes these are validated upstream during setup.

3. **No Explicit Padding on Reload:**
   - In multi-pass mode, if `compress_first_pass` suspends mid-component, the next call reprocesses the entire iMCU row. Dummy block construction happens in virtual arrays, so this is safe, but it's less efficient than single-pass suspension handling.

4. **Architectural Dependency Assumption:**
   - The file assumes `cinfo->mem` provides thread-safe or appropriately-synchronized virtual array access. In a multithreaded Q3A context (if enabled), contention on the memory manager could be a bottleneck, though JPEG encoding is typically single-threaded in this engine.
