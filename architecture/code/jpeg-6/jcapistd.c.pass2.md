# code/jpeg-6/jcapistd.c — Enhanced Analysis

## Architectural Role
This file implements the standard full-compression API for libjpeg-6, a vendored IJG JPEG library. While the **runtime Quake III engine uses JPEG only for **decompression** (texture loading in the renderer via `tr_image.c`), this compression module exists to support offline **tools** in the build pipeline (likely `q3map` for lightmap/texture processing, or radiant editor plug-ins). The file is intentionally separated from the minimal API (`jcapimin.c`) to prevent bloating transcoding-only applications with full compressor code.

## Key Cross-References

### Incoming (who depends on this file)
- **Tool-time only** (not runtime engine):
  - `q3map/` build toolchain (lightmap/texture writing)
  - `q3radiant/` level editor (if it exports textures)
  - Custom compression utilities in the build system
- Not called by:
  - Renderer (`tr_image.c`) — uses decompression, not compression
  - Client/server game code — no runtime JPEG encoding
  - cgame/game VMs — no bytecode-level JPEG access

### Outgoing (what this file depends on)
- **libjpeg-6 internal hierarchy** (defined elsewhere in `code/jpeg-6/`):
  - `jpeglib.h` — public type definitions (`j_compress_struct`, `JSAMPARRAY`, `JSAMPIMAGE`)
  - `jpegint.h` (via `JPEG_INTERNALS` macro) — private submodule interfaces:
    - `jpeg_comp_master` (vtable: `prepare_for_pass`, `pass_startup`, `call_pass_startup`)
    - `jpeg_c_main_controller` (vtable: `process_data`)
    - `jpeg_c_coef_controller` (vtable: `compress_data`)
    - `jpeg_destination_mgr` (vtable: `init_destination`)
    - `jpeg_error_mgr` (vtable: `reset_error_mgr`)
  - **Defined in companion files**:
    - `jinit_compress_master()` from `jcmaster.c` — bootstrap all submodules
    - `jpeg_suppress_tables()` from `jcparam.c` — mark tables for transmission

## Design Patterns & Rationale

### 1. **Modular Initialization with Vtable Dispatch**
Each call in `jpeg_start_compress` delegates to a submodule via function pointers in `cinfo` (the compression instance struct):
```
(*cinfo->err->reset_error_mgr)()
(*cinfo->dest->init_destination)()
(*cinfo->master->prepare_for_pass)()
```
**Why:** Allows swapping error handlers, destination modules (file, memory buffer, custom), and compression strategies without recompiling. Similar to the renderer's `refexport_t` vtable pattern in this engine.

### 2. **State Machine Enforcement**
`global_state` guards valid transitions:
- Entry: requires `CSTATE_START`
- After init: transitions to `CSTATE_SCANNING` or `CSTATE_RAW_OK`
- Errors on invalid state (e.g., calling `jpeg_write_scanlines` in wrong state)

**Why:** Prevents misuse (e.g., writing data before initialization) and simplifies error reporting.

### 3. **Lazy Header Emission via `call_pass_startup`**
On first call to `jpeg_write_scanlines`/`jpeg_write_raw_data`, the compressor defers frame/scan header output via `pass_startup`, allowing the application to inject COM markers in between:
```c
if (cinfo->master->call_pass_startup)
    (*cinfo->master->pass_startup)(cinfo);
```
**Why:** JPEG spec allows application data (COM, APP0–APP15 markers) between SOI and SOF. This design delays SOF emission to let callers write metadata.

### 4. **Suspension/Resume Architecture**
Data-feeding functions (`jpeg_write_scanlines`, `jpeg_write_raw_data`) return the number of lines actually consumed. If the destination module requests suspension (e.g., output buffer full), the compressor suspends and returns `0`, allowing the application to drain buffers and resume.

**Why:** Enables async/streaming I/O and prevents blocking on slow output channels.

### 5. **Safety-by-Default: `write_all_tables` Parameter**
Rather than emitting abbreviated streams (no Huffman/quantization tables) as a silent default when reusing a `j_compress_struct`, the API forces explicit opt-in:
```c
if (write_all_tables)
    jpeg_suppress_tables(cinfo, FALSE);  /* mark all tables to be written */
```
**Rationale from comments:** Multiple runs from the same compressor object leave `sent_table=TRUE`, so a subsequent run would silently emit abbreviated JPEG (tables omitted), which is **dangerous** if the decoder doesn't have the prior tables. Forcing `write_all_tables=TRUE` as the recommended default prevents this common mistake.

## Data Flow Through This File

1. **Initialization Phase:**
   - Application calls `jpeg_start_compress(cinfo, write_all_tables=TRUE)`
   - File validates state, optionally marks all tables, resets error/destination managers
   - Calls `jinit_compress_master` to instantiate all encoder submodules
   - Calls `prepare_for_pass` to allocate working buffers
   - Sets `next_scanline=0`, `global_state=CSTATE_SCANNING`

2. **Data Feeding Loop:**
   - Application repeatedly calls:
     - `jpeg_write_scanlines(cinfo, scanlines[], num_lines)` for interleaved RGB/YCbCr input, OR
     - `jpeg_write_raw_data(cinfo, data[], num_lines)` for pre-downsampled per-component input
   - Each call:
     - Validates state (`CSTATE_SCANNING` or `CSTATE_RAW_OK`)
     - Updates progress monitor counters (if registered)
     - Triggers `pass_startup` on first call (emits headers, allows COM injection)
     - Routes to internal `process_data` (scanlines) or `compress_data` (raw) vtables
     - Advances `next_scanline` by lines consumed
     - Returns count of lines processed (may be < requested if suspension occurs)

3. **Finalization:**
   - Application calls `jpeg_finish_compress()` (in `jcapimin.c`, not here)
   - Flushes all remaining blocks, emits EOI marker, writes trailers

## Learning Notes

### Idiomatic 1990s C Library Design
- **Stateful objects via `struct` pointers:** The `j_compress_ptr` is the only parameter to most functions; all state lives in the struct. No thread-local state or globals. (Compare: modern libraries use opaque handles or closures.)
- **Vtable-based polymorphism:** Every major subsystem (error handling, destination I/O, compression strategy) is pluggable via function pointers embedded in the struct. No inheritance, but achieves modularity.
- **Explicit suspension/resumption:** No exceptions, no callbacks with context; the caller polls return values. Fits the 1990s ecosystem (DOS, Windows 3.1, embedded systems) where resources were tight.

### Contrast with Modern Engines
- **Modern GPU engines** (Unreal, Unity, Godot) would likely:
  - Load all texture assets in bulk at startup, compress offline, store in engine-native format
  - Avoid runtime JPEG compression entirely; use precomputed mipmaps, atlases, and format conversions
  - Use thread pools for I/O, not suspension-based coroutines
- **Quake III's design** reflects its era: flexible, modular, low-footprint, but requires careful caller discipline (state machine, loop structure)

### Game Engine Concepts
- **Resource pipelines:** This file is part of the texture asset pipeline. The flow is: **source image (any format) → (q3map/tool) JPEG compress → .pak archive → (runtime) JPEG decompress → GPU texture**.
- **Separation of concerns:** Compression (offline, tool-time) and decompression (runtime) are completely separate code paths. The engine **never** compresses at runtime.
- **Modular I/O:** The `jpeg_destination_mgr` vtable is the engine's abstraction over "where does compressed output go?" (file, memory, network). This mirrors modern engine patterns like `IStream` or `ByteBuffer`.

## Potential Issues

### 1. **No Type Safety on `cinfo` Dereference**
Lines like `cinfo->progress->progress_monitor` assume:
- `cinfo->progress != NULL` is checked, but
- `cinfo->master`, `cinfo->main`, `cinfo->coef` are assumed non-NULL after `jpeg_start_compress`

If `jinit_compress_master` fails silently (unlikely, but not guarded), subsequent calls would crash. **Mitigation:** `jinit_compress_master` likely calls `ERREXIT` on failure, which longjmps, preventing further execution.

### 2. **Suspension Semantics Unclear to Caller**
Lines like:
```c
row_ctr = 0;
(*cinfo->main->process_data)(cinfo, scanlines, &row_ctr, num_lines);
cinfo->next_scanline += row_ctr;
return row_ctr;
```
If `process_data` suspends and returns `row_ctr=0`, the caller must call again with the **same** `scanlines` pointer. If the caller frees or reuses the buffer, corruption occurs. **Documentation burden:** The caller must understand suspension semantics; no guard against misuse.

### 3. **Raw Data Requires Exact iMCU Row Count**
```c
lines_per_iMCU_row = cinfo->max_v_samp_factor * DCTSIZE;
if (num_lines < lines_per_iMCU_row)
    ERREXIT(cinfo, JERR_BUFFER_SIZE);
```
`jpeg_write_raw_data` **fails** if you don't pass exactly `lines_per_iMCU_row` (or multiples thereof). Contrast: `jpeg_write_scanlines` is forgiving (silently clamps excess rows). **Education opportunity:** Tool authors must understand MCU geometry or face mysterious errors.
