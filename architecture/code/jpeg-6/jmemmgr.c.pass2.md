# code/jpeg-6/jmemmgr.c — Enhanced Analysis

## Architectural Role
This file implements the memory manager for the vendored IJG libjpeg-6 decoder, which is consumed exclusively by the renderer's texture-loading subsystem (`code/renderer/tr_image.c` → `jload.c`). It provides pool-based allocation and optional disk-backed virtual arrays to handle image decoding within bounded memory limits, enabling texture loading of images larger than available RAM via backing-store swapping.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** (via `tr_init.c`) calls JPEG decoder during texture-load pipeline (See Architecture: "Load, resample, gamma-correct, mipmap, and upload all textures")
- **`jload.c`** in this same directory wraps the decoder entry point and invokes this memory manager
- All `jpeg_*` allocation/deallocation syscalls (e.g., `jpeg_get_small`, `jpeg_mem_available`) defined here are consumed by every JPEG operation during decompression

### Outgoing (what this file depends on)
- **System-dependent layer** (`jmemsys.h`, e.g., `jmemnobs.c` or `jmemansi.c`): provides actual malloc/free, memory limits (`jpeg_mem_available`), and backing-store I/O (`jpeg_open_backing_store`)
- **Error dispatch** via `ERREXIT`/`ERREXIT1` macros (likely defined in `jerror.h` via `jinclude.h`)
- **Zero-fill utility** `jzero_far` for pre-zeroing virtual array regions

## Design Patterns & Rationale

**Pool-lifetime segregation** (`JPOOL_PERMANENT` vs `JPOOL_IMAGE`): Reflects JPEG library's multi-pass scan architecture—permanent structures (Huffman tables, quantization) survive the image; image-specific buffers can be freed after decompression. This enables sequential image decoding without restart overhead.

**Never-null allocation contract**: Entire design assumes `alloc_*` never returns NULL; failures invoke `out_of_memory()` which calls `ERREXIT` (longjmp into engine error handler). This simplifies decoder logic and matches 1990s JPEG library philosophy of immediate hard-exit on allocation failure. Modern engines would prefer error codes or fallbacks.

**Virtual array backing store**: The deferred realization pattern (`request_virt_*` → `realize_virt_arrays`) allows the decoder to ask for large buffers speculatively, then decide at realization time how many rows fit in memory. Rows exceeding the in-memory window are swapped to disk. **Critical observation**: This design was essential when target systems (embedded devices, mid-1990s PCs) had severely constrained RAM but abundant disk. On modern systems with gigabytes of RAM, backing-store swaps would be catastrophic performance events if triggered during frame rendering.

**Slop-based fragmentation reduction**: The dual-slop strategy (first pool vs. subsequent pools) reflects tuning for typical JPEG workload patterns. First image decode needs large buffers; subsequent images can reuse space. Values (1600 permanent, 16000/5000 image) are embedded constants—no adaptive sizing.

## Data Flow Through This File

1. **Initialization** (`jinit_memory_mgr`): Engine calls during JPEG object creation; reads optional `JPEGMEM` environment variable to override memory policy; sets up pool lists, calls system-dependent `jpeg_mem_init`.

2. **Allocation phase**: Decoder calls `alloc_sarray`/`alloc_barray` to allocate row-buffers for intermediate data. Row pointers allocated from "small" pool (fast reuse), row data from "large" pool (one-per-request). Virtual arrays registered but not realized yet.

3. **Realization** (`realize_virt_arrays`): Called once before scanning begins. Queries `jpeg_mem_available`, allocates in-memory buffer, opens backing store file if needed. **Data enters** from decoder's request queue; rows are written to file or held in memory depending on available budget.

4. **Access phase** (`access_virt_*`): During scanning, decoder accesses strips. If requested strip is outside current in-memory window, old data flushed to backing store, new data loaded via `do_*array_io`.

5. **Teardown** (`free_pool`, `self_destruct`): Closes backing-store files, frees all pools, resets `cinfo->mem = NULL`.

## Learning Notes

**Era-specific constraints visible here**:
- Alignment union pattern (`ALIGN_TYPE dummy`) is pre-C99 portable alignment before `_Alignof`/`alignas`
- Manual fragmentation management (slop halving on retry) reflects systems where malloc was unpredictable
- Environment variable override suggests deployment uncertainty (unknown memory limits)
- Backing-store design is diagnostic of 1990s embedded/console engineering trade-offs (disk cheap, RAM scarce)

**Differences from modern engines**:
- Modern JPEG decoders (e.g., libjpeg-turbo) assume sufficient RAM; no backing-store support
- No error codes; hard-exit-on-failure is incompatible with event-driven engines
- No streaming/progressive decode support visible; assumes full image in memory
- No thread-safety; single-threaded JPEG decode only

**Connection to game engine concepts**:
- This is a subsystem-isolation example: JPEG library's memory model is completely orthogonal to the engine's hunk/zone allocators (see Architecture: "`qcommon` provides zone and hunk memory allocation"). The engine loads the JPEG decoder as a DLL with its own memory context, creating a hard boundary.
- No ECS or scene-graph concepts apply; pure data-structure allocation.

## Potential Issues

**Critical performance hazard**: If `jpeg_mem_available` returns a value that causes `realize_virt_arrays` to open a backing store during frame rendering, disk I/O during texture load would cause frame-rate hitches. This was acceptable in 1995; it's a bug in a 2000+ engine. **Mitigation**: The `JPEGMEM` environment variable is meant to be set conservatively (well below system RAM) to avoid this scenario, but it's a human-configuration issue, not engine-enforced.

**System-dependent binding**: This file depends on three system-dependent entry points (`jpeg_get_small/large`, `jpeg_mem_available`, `jpeg_open_backing_store`) whose implementations are not visible in the first-pass code. If `jmemnobs.c` returns misleading memory estimates, allocation failures are silent (fail-and-retry loop with slop halving), potentially leading to allocation thrashing and slowdown rather than clear errors.

**MAX_ALLOC_CHUNK** (not shown in excerpt): This constant is a silent upper bound on single-allocation size. If exceeded by any single decode, `out_of_memory` is called. The constant's value is critical but not visible here; mismatches between limit and typical JPEG sizes cause hard failures with no fallback.
