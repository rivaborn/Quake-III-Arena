# common/imagelib.h — Enhanced Analysis

## Architectural Role

This header defines the **image I/O façade for the offline asset pipeline**. It sits at the tool-side boundary, consumed exclusively by `q3map/`, `bspc/`, and `q3radiant/` — never by the runtime engine. The file is a compatibility layer bridging multiple legacy texture formats (LBM, PCX, TGA) into a uniform memory model: heap-allocated pixel buffers + optional palettes. Its design reflects Q3A's heterogeneous 1990s–2000s asset workflow, where artists used different paint/modeling tools that produced different formats.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/** (BSP compiler): Texture loading during lightmap compilation and shader parsing; likely calls `Load256Image`/`LoadTGA`/`Load32BitImage` to resolve texture paths in BSP geometry
- **q3radiant/** (level editor): Preview and in-editor texture display; calls loaders when opening maps or changing textures
- **bspc/** (BSP→AAS offline compiler): May reference indirectly through shared q3map code or not at all if it focuses purely on geometry/navigation
- **Offline utility tools**: Any standalone tools for batch texture conversion or processing

### Outgoing (what this file depends on)
- **Standard C library** (`stdio.h`, `stdlib.h` implicit): `fopen`/`fread`/`malloc` for I/O and memory allocation
- **common/cmdlib.h**: Type definitions (`byte` type likely defined here; implied by function signatures)
- **Platform-specific file I/O**: The implementation (not shown) calls platform-level file APIs via `common/cmdlib.c` or equivalent
- **No explicit memory allocator**: Functions don't specify `Hunk_Alloc` vs. `malloc`. Likely `malloc` for tools (vs. qcommon's hunk for runtime)

## Design Patterns & Rationale

### **Format Dispatcher Pattern**
The dual-level interface (specific loaders + generic wrappers) decouples call sites from format knowledge:
```c
Load256Image(name, pixels, palette, w, h)  // Dispatch by .lbm / .pcx extension
  → LoadLBM(...) or LoadPCX(...) internally
```
This reduces coupling in tools (q3map doesn't hardcode format per file) and allows format additions without changing callers.

### **Raw Pointer Output Convention**
```c
void LoadLBM(const char *filename, byte **picture, byte **palette)
```
Using `byte **` (out-pointer) rather than returning `struct { byte *pic; byte *pal; }` reflects **C89 era conventions** and matches qcommon's style (cf. `MSG_*` syscall patterns). Callers assume responsibility for freeing both allocations separately.

### **Format-Specific Metadata Variance**
- LBM/PCX: Encode width/height internally; palette always 256 entries × 3 bytes (RGB)
- TGA: Width/height are explicit parameters; pixel format inferred from file (can be 8/24/32-bit)
- `Load32BitImage`: Abstracts format choice for 32-bit RGBA; likely maps `.tga` → `LoadTGA`, falls back for other formats

This **loose typing** (no enums, no explicit format descriptors) was typical of early 2000s build tools prioritizing simplicity over type safety.

### **In-Memory Parsing (LoadTGABuffer)**
Unique to TGA: the ability to parse from a memory buffer rather than disk suggests tools sometimes pre-load images into memory (e.g., from a `.pak` or for batch processing) before parsing. This is a pragmatic tool-time optimization absent in runtime (which streams from disk).

## Data Flow Through This File

```
User (q3map/q3radiant/etc.)
  ↓
Load*Image() / Load*File()  [header declares only]
  ↓ [implementation calls platform I/O, malloc, format-specific decoder]
  ├→ Heap: pixel buffer (malloc'd)
  ├→ Heap: palette buffer (malloc'd, if 8-bit)
  └→ Output: dimensions, pitch info (via out-pointers)
  
User responsible for:
  ├→ free(pixels)
  ├→ free(palette)  [if not NULL]
  └→ Process or save elsewhere
```

**Key assumption**: No error indication in function signature (void return). Callers must either:
1. Trust the image exists and is valid
2. Check for NULL pointers post-call (likely pattern, though unspecified)
3. Use global `last_error` state (common in tool-era code)

## Learning Notes

### **Era-Specific Idioms**
1. **No dynamic structs**: Modern C APIs might return `image_t *` with metadata embedded; here, caller juggles multiple allocations manually. Reflects pre-C99 conservatism.
2. **Format diversity**: LBM (Deluxe Paint), PCX (ZSoft legacy), TGA (Targa). Modern engines standardize on PNG/JPG. Q3A's support reflects artist workflows of the time.
3. **8-bit paletted textures still present**: Even in a 32-bit engine, Q3A could cache 8-bit paletted textures in VRAM. Modern engines dropped this entirely.

### **Separation of Concerns (Tool vs. Runtime)**
This file is **never** included by `code/qcommon`, `code/client`, `code/server`, or `code/renderer`. The runtime engine likely has a separate `tr_image.c` (in renderer/) that:
- Loads textures at runtime (only TGA/JPG formats shipped in `.pk3` archives)
- Allocates from the GPU-resident hunk, not malloc
- Integrates with the OpenGL state machine

The strict separation shows **Q3A's architecture clarity**: offline tools and runtime are different programs with different constraints.

### **Game Engine Programming Concepts**
- **No ECS or scene graph**: imagelib is purely I/O; it knows nothing of textures as game objects. Caller must wire loaded pixels into a `gltexnum_t` or shader binding.
- **Lazy binding**: Images are loaded on-demand, not precompiled into archives. The mapper decides which images to reference; the editor/compiler resolves them at tool time.

## Potential Issues

1. **No error reporting**: A missing or corrupted image file produces a void return. Callers relying on error checks will get dangling pointers or silent failures. Likely mitigated by logging during load, but **not enforced by the interface**.

2. **Unspecified memory semantics**: Caller must know to `free()` allocations. No ownership container or RAII-style cleanup. Risk of leaks if caller forgets or if control flow exits early.

3. **Palette assumptions for 8-bit**: `LoadLBM` and `LoadPCX` return `byte **palette` implicitly as 256×3 bytes. No field documenting this; caller must assume. A malformed file could violate this.

4. **No bit-depth indication**: `LoadTGA` and `LoadTGABuffer` return `byte **pixels` for any TGA bit depth. Caller must infer from width/height whether data is 24-bit or 32-bit; no enum return value specifies format.

---

**Size estimate:** ~300 tokens (excluding this markdown wrapper). Core insights: format-dispatcher pattern, strict tool-vs-runtime separation, era-specific C idioms, and lack of formal error handling.
