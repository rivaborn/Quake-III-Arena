# code/jpeg-6/jmemnobs.c

## File Purpose
Provides the Quake III renderer-integrated system-dependent JPEG memory manager backend. It replaces standard `malloc`/`free` with the renderer's `ri.Malloc`/`ri.Free` allocator functions, ensuring JPEG memory operations go through the engine's tracked heap. Backing store (disk temp files) is explicitly unsupported.

## Core Responsibilities
- Implement `jpeg_get_small`/`jpeg_free_small` via `ri.Malloc`/`ri.Free`
- Implement `jpeg_get_large`/`jpeg_free_large` via the same allocator (no distinction between small/large)
- Report unlimited available memory to the JPEG library (`jpeg_mem_available`)
- Unconditionally error out if backing store is ever requested (`jpeg_open_backing_store`)
- Provide no-op init/term lifecycle stubs (`jpeg_mem_init`/`jpeg_mem_term`)

## Key Types / Data Structures
None — this file defines no new types.

## Global / File-Static State
None — all state is managed externally via the renderer's `ri` global (defined in `tr_local.h`).

## Key Functions / Methods

### jpeg_get_small
- Signature: `GLOBAL void * jpeg_get_small(j_common_ptr cinfo, size_t sizeofobject)`
- Purpose: Allocate a small memory block for the JPEG library.
- Inputs: `cinfo` — JPEG context (unused); `sizeofobject` — byte count requested.
- Outputs/Return: `void *` pointer to allocated memory.
- Side effects: Calls `ri.Malloc`, incrementing the renderer's heap usage.
- Calls: `ri.Malloc`
- Notes: `cinfo` is ignored entirely.

### jpeg_free_small
- Signature: `GLOBAL void jpeg_free_small(j_common_ptr cinfo, void *object, size_t sizeofobject)`
- Purpose: Release a previously allocated small block.
- Inputs: `object` — pointer to free; `sizeofobject` — ignored by `ri.Free`.
- Outputs/Return: void
- Side effects: Calls `ri.Free`.
- Calls: `ri.Free`

### jpeg_get_large / jpeg_free_large
- Identical in behavior to `jpeg_get_small`/`jpeg_free_small`; `FAR` qualifier is nominal. Both delegate directly to `ri.Malloc`/`ri.Free`.

### jpeg_mem_available
- Signature: `GLOBAL long jpeg_mem_available(j_common_ptr cinfo, long min_bytes_needed, long max_bytes_needed, long already_allocated)`
- Purpose: Report available memory to JPEG library's virtual array manager.
- Inputs: `max_bytes_needed` — upper bound requested.
- Outputs/Return: Returns `max_bytes_needed` unconditionally, telling the library it can have everything it wants.
- Side effects: None.
- Notes: `max_memory_to_use` limit is intentionally ignored; the assumption is virtual memory is sufficient.

### jpeg_open_backing_store
- Signature: `GLOBAL void jpeg_open_backing_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- Purpose: Called when the JPEG library needs a temp file; always fatal.
- Inputs: Standard JPEG backing store params.
- Outputs/Return: Does not return (calls `ERREXIT`).
- Side effects: Triggers JPEG error handler (`JERR_NO_BACKING_STORE`), which longjmps out.
- Notes: Should never be reached given `jpeg_mem_available` always returns `max_bytes_needed`.

### jpeg_mem_init / jpeg_mem_term
- No-ops. `jpeg_mem_init` returns `0` (setting `max_memory_to_use` to 0). `jpeg_mem_term` does nothing.

## Control Flow Notes
This file is not part of the game frame loop directly. It is invoked during JPEG decompression (e.g., texture loading in `R_FindImageFile`/`R_CreateImage`), which occurs at load time or during asset registration. Memory is allocated per-image load and freed when the JPEG context is destroyed.

## External Dependencies
- `jinclude.h` — platform include shims, `SIZEOF`, `MEMCOPY`, etc.
- `jpeglib.h` — JPEG library types (`j_common_ptr`, `backing_store_ptr`, `ERREXIT`, `JERR_NO_BACKING_STORE`)
- `jmemsys.h` — declares the function signatures this file implements
- `../renderer/tr_local.h` — exposes `extern refimport_t ri`, providing `ri.Malloc` and `ri.Free`
- `ri` (`refimport_t`) — defined elsewhere in the renderer; this file depends on it being initialized before any JPEG operation occurs
