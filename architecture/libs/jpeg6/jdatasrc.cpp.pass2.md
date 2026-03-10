# libs/jpeg6/jdatasrc.cpp — Enhanced Analysis

## Architectural Role

This file implements a **memory-buffer data source adaptor** for the vendored IJG libjpeg-6 decompression library. It bridges the JPEG decompressor's abstract source-manager interface to in-memory JPEG data, enabling the renderer's texture loading pipeline to decompress JPEG images from loaded files without intermediate streaming abstractions. Although named `jpeg_stdio_src()`, the implementation sources JPEG data directly from a pre-loaded memory buffer—this is a key design choice that avoids file I/O overhead during renderer-driven texture setup.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** (implied via `code/jpeg-6/jload.c`) — Renderer texture loader calls libjpeg decompression routines, which in turn invoke the `init_source` → `fill_input_buffer` callback chain registered by this module
- **libjpeg internal decompression pipeline** — All JPEG decompression calls routed through `jpeg_stdio_src()` setup will use the function pointers registered here

### Outgoing (what this file depends on)
- **Memory allocator from JPEG library context (`cinfo->mem`)** — All allocations routed through `(*cinfo->mem->alloc_small)()` with `JPOOL_PERMANENT` lifetime, ensuring buffers survive across multiple image loads from the same JPEG context
- **No other subsystem dependencies** — Entirely self-contained; data source logic is independent of the Q3A engine core

## Design Patterns & Rationale

**Plugin/Strategy Pattern:** JPEG library defines an abstract `jpeg_source_mgr` vtable; this module implements one concrete strategy (memory-buffer sourcing). The library is indifferent to the source's nature, allowing swappable implementations without recompilation.

**Permanent Buffer Lifetime:** The comment at `jpeg_stdio_src()` entry justifies keeping source manager and input buffer allocated permanently: *"If we discarded the buffer at the end of one image, we'd likely lose the start of the next one."* This supports the Q3A use case of decompressing multiple texture JPEG images from a single opened source context, avoiding repeated allocate/free cycles.

**Dummy Implementation for Skip:** The `skip_input_data()` function implements a simple loop-and-refill strategy rather than using `fseek()`. This is deliberate—the comments note `fseek()` doesn't work on pipes, and for in-memory buffers, the performance penalty is negligible. The implementation assumes `fill_input_buffer()` will never return FALSE (suspension not supported), simplifying bot-level logic.

## Data Flow Through This File

```
[JPEG-compressed buffer in memory]
           ↓
[jpeg_stdio_src() setup]
  ├─ Allocate my_source_mgr + JOCTET buffer (permanent pool)
  ├─ Register function pointers (init, fill, skip, term)
  └─ Store infile pointer to compressed source
           ↓
[Decompression loop (driven by libjpeg)]
  ├─ init_source() — mark start_of_file=TRUE
  ├─ fill_input_buffer() — memcpy from infile[pos] into buffer
  │                         advance infile pointer
  │                         set pub.next_input_byte + bytes_in_buffer
  ├─ skip_input_data() — advance next_input_byte within buffer
  │                       or refill if skip exceeds buffer
  └─ term_source() — no-op cleanup
           ↓
[Decompressed image pixels → texture upload]
```

**Key invariant:** The input `infile` pointer is a linear memory buffer; `fill_input_buffer()` assumes sequential reads. The function increments the source pointer by `INPUT_BUF_SIZE` (4096 bytes) on each refill, copying that chunk into the reusable `buffer` region.

## Learning Notes

**How Q3A integrates JPEG:** The texture loader (`tr_image.c`) reads JPEG files into memory, then hands the buffer to `jpeg_stdio_src()` → `jpeg_decompress_image()`. This avoids libjpeg's built-in `FILE*`-based source; instead, the entire JPEG is buffered, then decompressed in one pass. This was common in early-2000s engines to enable seamless asset loading and streaming.

**Memory pool strategy (JPOOL_PERMANENT):** The JPEG library allocates from pools with explicit lifetimes. Choosing `JPOOL_PERMANENT` ensures the source manager survives across multiple calls to `jpeg_read_header()` / `jpeg_decompress_image()` on the same `j_decompress_ptr`, avoiding reallocation overhead. Modern engines often use arena allocators; Q3A's two-pool design (`JPOOL_TEMPORARY` vs `JPOOL_PERMANENT`) is an early variant of that idea.

**Why not `FILE*`?** The comment header mentions "reading JPEG data from a file (or any stdio stream)" but the actual implementation reads from memory. This is intentional: by the time the renderer loads a texture, the file is already in the virtual filesystem (`FS_ReadFile`) as a buffer. Avoiding a second layer of I/O buffering (JPEG's FILE* layer + Q3A's virtual FS) kept decompression fast on 2000-era hardware.

## Potential Issues

1. **Buffer Overflow in `fill_input_buffer()`:** The function blindly copies `INPUT_BUF_SIZE` bytes from `src->infile` without bounds checking. If the source buffer is smaller than the requested size, this is undefined behavior. The renderer must ensure the JPEG file buffer is large enough; no runtime guard prevents truncated JPEG data from overrunning the read window.

2. **Suspension Not Supported:** Both `fill_input_buffer()` (always returns TRUE) and `skip_input_data()` assume data is available. If the JPEG library ever calls these expecting a FALSE return for suspension/backpressure, the engine will deadlock or corrupt state. The comments in `skip_input_data()` acknowledge this assumption: *"note we assume that fill_input_buffer will never return FALSE"*—this works only if the entire JPEG is pre-buffered.
