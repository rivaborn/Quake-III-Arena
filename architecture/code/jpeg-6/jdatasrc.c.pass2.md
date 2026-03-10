# code/jpeg-6/jdatasrc.c — Enhanced Analysis

## Architectural Role

This file implements the **decompression data source interface** that bridges raw JPEG memory buffers (from the renderer's texture loader) to the IJG JPEG decompressor core. It is a vendored, minimal adaptation of the IJG reference implementation—replacing stdio I/O with direct memory-to-memory copying. It sits at the critical **renderer → libjpeg** boundary for all texture decompression during level and model loading, with no path to the game logic or network stack.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** — calls `jpeg_stdio_src` (misleading name; actually memory-based) to initialize JPEG decompression for raw texture bytes loaded from the virtual filesystem
- **`code/jpeg-6/jdcomapi.c`** and other IJG core files — invoke all five callback methods (`init_source`, `fill_input_buffer`, `skip_input_data`, `term_source`) during `jpeg_read_header`, scanline reads, and cleanup
- **`code/renderer` (overall)** — depends on a working JPEG pipeline; jdatasrc is part of the initialization contract for `jpeg_create_decompress(cinfo)` + `jpeg_stdio_src(cinfo, data)`

### Outgoing (what this file depends on)
- **`code/jpeg-6/jpeglib.h`** — provides `jpeg_source_mgr` interface definition, `j_decompress_ptr`, `JPOOL_PERMANENT`, and the memory allocator hook (`cinfo->mem->alloc_small`)
- **`code/jpeg-6/jinclude.h`** — platform-specific macros (`SIZEOF`, `memcpy`)
- **Memory allocator** — calls `(*cinfo->mem->alloc_small)()` for permanent pool allocation (set up by `jpeg_create_decompress`)
- **`jpeg_resync_to_restart`** — library default restart-marker recovery callback (used for error resilience)

## Design Patterns & Rationale

**Why memory-based, not stdio?** The renderer loads textures asynchronously from the virtual filesystem (`ri.FS_ReadFile`) into memory; copying again from a `FILE *` would waste cycles and buffer complexity. Direct buffer input is simpler for a game engine that already stages file I/O.

**Permanent allocation across multiple images:** The struct and intermediate buffer are allocated once (`JPOOL_PERMANENT`) and reused for sequential textures. This trades memory efficiency (one 4KB buffer per decompress context) for initialization cost amortization. Matches the IJG reference design but assumes the same `j_decompress_ptr` is not mixed with different source managers (enforced by the caller).

**Stateless callbacks:** The library calls back into `fill_input_buffer` repeatedly; the state machine is implicit—each call just copies the next chunk and advances the source pointer (`src->infile`). No bookkeeping of EOF or total size.

**No suspension support:** Unlike the reference stdio version (which comments extensively on suspension for streaming), this version always returns `TRUE` from `fill_input_buffer`. The renderer has the entire texture in RAM, so suspension is unnecessary and would complicate the initialization contract.

## Data Flow Through This File

```
Renderer loads texture via FS_ReadFile()
  ↓
(raw JPEG bytes in memory at pointer P)
  ↓
renderer calls: jpeg_stdio_src(cinfo, P)
  ├─ [first time only] allocate my_source_mgr struct + 4KB buffer
  └─ wire up callback function pointers
  ↓
renderer calls: jpeg_read_header(cinfo, ...)
  ↓
IJG core calls: init_source(cinfo)
  └─ sets start_of_file = TRUE
  ↓
[scanline decompression loop]
  ↓
IJG core calls: fill_input_buffer(cinfo) [0+ times]
  ├─ memcpy(4KB from src->infile to src->buffer)
  ├─ advance src->infile by 4KB
  └─ return TRUE
  ↓
IJG core calls: skip_input_data(cinfo, nbytes) [0+ times for APPn markers]
  ├─ call fill_input_buffer() if we skip beyond current buffer
  └─ update next_input_byte / bytes_in_buffer
  ↓
[decompression complete]
  ↓
IJG core calls: term_source(cinfo)
  └─ no-op
  ↓
renderer calls: jpeg_destroy_decompress(cinfo)
```

## Learning Notes

- **Minimal stdlib:** This is pure JPEG machinery—no game engine concepts (collision, physics, networking) leak in. Good separation of concerns.
- **Buffer-size constant:** `INPUT_BUF_SIZE = 4096` is a historical choice from early Unix fread() efficiency studies; modern L1 caches and DMA make it suboptimal for large textures, but changing it requires recompilation and would change decompression timing.
- **Idiomatic IJG:** The callback pattern and permanent allocation strategy are directly from the IJG reference (`example.c`). The vendored code preserves this verbatim, making it easy to patch or upgrade libjpeg.
- **Modern engines:** Would likely use `libjpeg-turbo` (faster SIMD decompression) or load JPEGs at build time, converting to DDS or KTX. Quake III loads and decompresses on demand, trading CPU for memory flexibility.
- **No error semantics:** The comments warn about overread on EOF, but the code has no bounds check. The assumption is that `src->infile` always points to valid data and texture sizes are known. Caller responsibility.

## Potential Issues

1. **Unbounded memcpy in `fill_input_buffer`**: Copies exactly `INPUT_BUF_SIZE` bytes unconditionally. If the texture is smaller than 4KB or on the last refill, this will read past the end of the buffer. The caller (`tr_image.c`) must ensure the buffer is padded or the JPEG decompressor stops early enough. This is a common IJG gotcha documented in the reference source.

2. **No EOF signaling:** Unlike the stdio version (which detects feof), this version has no way to detect end-of-file. If the JPEG is truncated, `fill_input_buffer` will read garbage. Error recovery relies on IJG's restart marker logic or the decompressor hitting an internal size limit.

3. **Name mismatch:** Function is `jpeg_stdio_src` but takes a memory pointer, not a `FILE *`. This is retained from the original IJG API but is confusing for new maintainers.

---
