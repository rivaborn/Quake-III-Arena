# code/jpeg-6/jmemdos.c — Enhanced Analysis

## Architectural Role

This file is a **vendored third-party memory subsystem** (IJG libjpeg-6) providing MS-DOS-specific backing store for JPEG decompression. While not part of the core Quake III engine, it is consumed exclusively by the renderer's texture-loading pipeline (`code/renderer/tr_image.c` → `code/jpeg-6/jload.c` → memory manager). The file demonstrates a legacy three-tier fallback pattern (XMS extended memory → EMS expanded memory → DOS files) to allow JPEG decompression of large images within the severe RAM constraints of early-1990s DOS systems.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/jpeg-6/jload.c`**: Calls `jpeg_mem_init()` at startup and invokes the JPEG library's decompression entry points, which internally allocate/deallocate via `jpeg_get_small`, `jpeg_get_large`, etc.
- **`code/renderer/tr_image.c`**: Indirectly; loads JPEG textures through `jload.c`, triggering on-demand backing store via `jpeg_open_backing_store()`
- **JPEG library internals** (`jpeglib.h`, `jmemmgr.c`): The file defines platform-specific hooks that the portable IJG core invokes at load time

### Outgoing (what this file depends on)
- **Platform assembly stubs** (`jmemdosa.asm`): Provides low-level DOS I/O and XMS/EMS driver calls; critical boundary between C and real-mode/protected-mode code
- **Standard libc**: `malloc()`, `free()`, `getenv()`, `sprintf()`, `fopen()`, `fclose()`, `remove()` — all standard C runtime
- **Compiler-specific far-heap routines**: Turbo C's `farmalloc()` / `farfree()` or MSVC's `_fmalloc()` / `_ffree()`; tied to memory model
- **IJG JPEG library error/trace macros** (`jerror.h` via `jpeglib.h`): `ERREXIT`, `ERREXITS`, `TRACEMSS`, `TRACEMS1`

## Design Patterns & Rationale

**Why this code exists:** The IJG JPEG library is portable C; this file is the **platform-specific glue layer** (one of several: `jmemnobs.c` for simple systems, `jmemansi.c` for ANSI C, etc.). The design trades off portability for performance and feature richness.

**Why three backing stores?** DOS had three disjoint memory pools:
- **Conventional memory (640K)**: DOS file swap (slowest, but universally available)
- **Extended memory (XMS, V2.0)**: Fast, but required protected-mode context switch (expensive on 286)
- **Expanded memory (EMS/LIM, V4.0)**: Fast on 286/386, slower on newer CPUs due to page-mapping overhead

The code tries XMS first (favoring 386+ machines), falls back to EMS (good for 286), then to files. This is **hardware-era optimization** baked in at compile time.

**Far pointers and misaligned structs:** The `EMSspec` union with byte-offset macros (`FIELD_AT`, `SRC_PAGE`, etc.) is defensive against buggy compilers that over-align struct fields. Rather than risk relocation, the code accesses EMS spec fields byte-by-byte, sacrificing readability for guaranteed correctness.

**Odd-byte handling in XMS:** The `ODD()` macro and recursive read/write of the final byte in `read_xms_store`/`write_xms_store` work around an XMS V2.0 restriction: the driver's EMB-move operation requires even-length transfers. This is a microoptimization for a known hardware limitation.

## Data Flow Through This File

```
JPEG decompression request (texture load)
    ↓
jpeg_mem_init() [once per session]
  └─→ Reset next_file_num, report max memory
    ↓
Allocate small objects
  jpeg_get_small() → malloc()  [near heap, fast]
    ↓
Allocate large objects (when needed)
  jpeg_get_large() → far_malloc()  [far heap, slower]
    ↓
Query available memory
  jpeg_mem_available() → max_memory_to_use - allocated
    ↓
[If backing store needed → exceeds max_memory_to_use]
jpeg_open_backing_store()
  └─→ Try open_xms_store()   [success → wire XMS read/write/close]
      └─→ else try open_ems_store()  [success → wire EMS read/write/close]
          └─→ else open_file_store()  [always succeeds or ERREXIT]
                └─→ select_file_name() → TMP/TEMP env-var + counter
                    ├─→ Probe for collision with fopen()
                    └─→ Create jdos_open() → DOS file handle
    ↓
Read/write backing store as needed
  read_*_store(offset, count) / write_*_store(offset, count)
    ↓
jpeg_mem_term() [on session shutdown]
  └─→ Optional _fheapmin() [MSVC-specific large-block reclaim]
    ↓
close_*_store() [on backing store cleanup]
  └─→ Deallocate XMS/EMS block or remove temp file
```

## Learning Notes

**What a developer learns:**
1. **Vendored libraries require platform shims**: JPEG is portable; this file adapts it to DOS constraints. Modern engines (with SIMD, shader compilation, etc.) likewise need platform-specific wrappers.
2. **Real-mode memory model complexity**: The `far_malloc`/`far_free` macros and `void FAR *` pointers are artifacts of 16-bit x86 segmentation. Modern flat-address-space systems don't need this; studying it illuminates why 32-bit (and later 64-bit) architectures won.
3. **Hardware-specific optimization**: The XMS-before-EMS choice is explicit policy, not a generic algorithm. Performance depends on CPU generation. Modern engines would profile at runtime.
4. **Defensive coding for broken compilers**: The `EMSspec` byte-offset macros are not elegant C; they're a pragmatic workaround for struct alignment bugs in 1990s-era compilers. Today's compilers are more standards-compliant, making this obsolete.
5. **Fallback chains are reliable**: The code tries the fastest path first, then degrades gracefully. This pattern (fast GPU path, slow CPU fallback) is still idiomatic.

**Idiomatic to this era:**
- Assembly-language stubs for low-level I/O and driver calls (now the job of the OS kernel)
- Explicit memory model annotations (`FAR`, `NEED_FAR_POINTERS`)
- Static `next_file_num` counter for collision avoidance (modern systems use `mkstemp()` or UUID)
- Compile-time feature gates (`XMS_SUPPORTED`, `EMS_SUPPORTED`) instead of runtime detection
- Direct environment-variable parsing for configuration

## Potential Issues

1. **Thread-safety:** The file-static `next_file_num` and `xms_driver` are not protected by locks. On a hypothetical multi-threaded renderer, concurrent JPEG decompressions could race on `select_file_name()`, generating duplicate file names. (Quake III is single-threaded except for optional SMP renderer, so this is not a practical issue, but worth noting.)

2. **Assumptions about struct alignment:** The code assumes 2-byte-or-better alignment (`/* struct fields aligned on greater than 2-byte boundaries */`). Compilers with `-fpack-struct=4` or Pentium-P6 (8-byte alignment) could break EMS access.

3. **Hard-coded `DEFAULT_MAX_MEM` (300 KB):** If JPEG texture decoding for a large image exceeds 300 KB working space, backing store is invoked. On DOS machines with limited conventional RAM, this could thrash heavily. Modern systems with ample RAM never hit this path.

4. **Environment-variable dependency:** `TMP` and `TEMP` env-vars control temp file location. If unset or pointing to read-only storage, `select_file_name()` will silently use `"."`, potentially filling the current directory. No error feedback to the user.

5. **XMS/EMS driver contract not verified:** The code assumes the driver correctly implements XMS V2.0 or EMS V4.0 specs. A buggy driver (or one claiming to be present but not) could cause silent memory corruption. The `AX != 1` checks catch some errors but not all.
