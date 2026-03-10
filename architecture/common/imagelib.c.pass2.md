# common/imagelib.c — Enhanced Analysis

## Architectural Role

This file is the **offline asset I/O foundation** serving the entire build-tool ecosystem (q3map, bspc, q3radiant). It decouples image format handling from tool-specific code, providing a unified 8-bit and 32-bit image interface. As a pure utility library with no runtime role, it trades strict format support (only legacy 2D formats) for simplicity and compile-time integration into multiple independent executables without shared library dependencies.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map** — texture loading during BSP compilation; calls `Load256Image` / `Load32BitImage` to read source textures for shader preprocessing
- **bspc** — minimal image use (likely for visualization or debug output only; cross-reference context not provided but build tool consistency suggests limited scope)
- **q3radiant** — editor viewport textures and material previews; calls dispatch functions for all user-facing image operations (load materials, preview skyboxes)
- **code/renderer (runtime)** — **does NOT call imagelib**; uses `jpeg-6` directly for texture loading; imagelib is strictly offline-only
- **code/game, code/cgame, other runtime VMs** — **never access imagelib**; all image I/O happens pre-execution on the host

### Outgoing (what this file depends on)
- **cmdlib.h** (`common/cmdlib.c`) — `LoadFile`, `SaveFile`, `Error`, `ExtractFileExtension`, `Q_stricmp`, byte-order conversion macros (`BigShort`, `BigLong`, `LittleShort`, `LittleLong`)
- **Standard C** — `stdio.h` (FILE I/O, fgetc, fopen, fread), `stdlib.h` (malloc, free), `string.h` (memset, memcpy)
- **No renderer, qcommon, or game dependencies** — complete isolation from runtime engine

## Design Patterns & Rationale

### Extension-Driven Format Dispatch
`Load256Image` / `Load32BitImage` / `Save256Image` use `ExtractFileExtension` + `Q_stricmp` to select codecs by filename. This avoids passing format enums through tool interfaces and allows tools to work polymorphically with heterogeneous asset sources.

### Manual Chunk Parsing (IFF)
`LoadLBM` parses the FORM/BMHD/CMAP/BODY chunk structure without a generic IFF deserializer. This reflects the era's pragmatism: special-cased code is smaller and faster than a reusable framework. The global `bmhd` singleton is a quirk—tools expect to query dimensions after load without separate metadata passing.

### In-Memory Decompression Buffers
Unlike the renderer (which streams from pak3 ZIP archives), offline tools load entire files into memory (`LoadFile`), decompress in-place, and hand the result to callers. This trades memory for simplicity and is acceptable because build tools are not interactive or frame-rate-critical.

### Limited Compression Support
Only PackBits RLE (LBM) and ZSoft RLE (PCX) are supported; TGA and BMP are uncompressed or minimal. This reflects their era: texture compression (S3TC, etc.) was absent. Modern engines would use PNG or DDS with library dependencies; q3map avoids them for portability.

## Data Flow Through This File

```
Load Path:
  filename
    ↓
  LoadFile (cmdlib) → in-memory buffer
    ↓
  Format-specific parser (LoadLBM/LoadPCX/LoadBMP/LoadTGA)
    ├─ validate header
    ├─ decompress if needed (LBMRLEDecompress, PCX RLE)
    └─ allocate output buffer, populate from decompressed data
    ↓
  Return pointers to caller (pixels + optional palette)
    
Save Path:
  Raw pixel buffer + palette + filename
    ↓
  Format-specific encoder (WriteLBMfile/WritePCXfile/WriteTGA)
    ├─ allocate working buffer
    ├─ write header
    ├─ encode/compress payload
    └─ free working buffer
    ↓
  SaveFile (cmdlib) → disk
```

**Key detail**: Palette is always 768 bytes (256 × RGB); 32-bit formats skip it. TGA read produces RGBA (4 bytes/pixel); 8-bit formats produce index buffers. This asymmetry reflects the engine's dual-mode texturing: indexed for level geometry, RGBA for UI/sprites.

## Learning Notes

### Historical Context
This code is authentically early-2000s: manual memory management, format support frozen at a snapshot (LBM/PCX from Amiga/DOS eras, BMP from Windows 3.1, TGA from SGI), and no abstraction frameworks. Modern engines use libpng/libjpeg/DirectXTex and treat image I/O as a pluggable subsystem.

### Idiomatic Patterns
- **Chunk alignment**: `Align()` padding to 2-byte boundaries reflects IFF specification and Amiga memory requirements—unnecessary on x86 but preserved for format fidelity.
- **Byte-order macros**: `BigShort`, `LittleShort` appear throughout; the codebase was written to be endian-portable (even though by 2005 little-endian x86 dominated).
- **Error as control flow**: `Error()` is a `longjmp`-based exit, not a recoverable exception. Invalid files or buffer overruns abort the tool immediately.

### No Runtime Path
Critical insight: **imagelib never runs on player machines**. All image loading happens offline during map/asset authoring. The runtime engine (`code/renderer`) has a completely separate image pipeline (JPEG-only via `code/jpeg-6`; BSP textures pre-baked into pak3 archives). This separation of concerns is why imagelib can afford to be simple, non-optimized, and format-frozen.

## Potential Issues

1. **PCX RLE Overrun** (commented FIXME in `LoadPCX`): The decompressor may exceed buffer bounds on the last scanline due to malformed run-length codes. Truncation to buffer size prevents crashes but silently corrupts the last rows. Modern code would validate file integrity before decompression.

2. **Global `bmhd` State**: After `LoadLBM`, dimensions are stored in the global `bmhd` struct. Callers must query `bmhd.w` and `bmhd.h` immediately before calling another `Load*` function, or dimensions are lost. Thread-unsafe and fragile if tools ever parallelize asset loading.

3. **No Palette Validation**: 768-byte palette buffers are copied blindly. No check for required palette presence in indexed formats (e.g., PCX missing trailer palette is silently accepted as `NULL`). Tools tolerate this because they control source assets; real image files would require stricter validation.

4. **BMP Top-Down Assumption**: The code assumes negative height means top-down bitmap and flips rows. Non-standard BMP variants may not conform, but Q3 level tools are guaranteed to produce compatible files.
