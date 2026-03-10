# code/unix/qasm.h — Enhanced Analysis

## Architectural Role

This file is the **platform-specific assembly glue layer** for Unix/Linux x86 systems, connecting the engine's performance-critical software renderer and audio mixer to hand-optimized x86 assembly code compiled into the same binary. It reflects Quake III's original 1990s heritage: when every CPU cycle mattered, the rendering pipeline (rasterization, texture sampling) and real-time audio mixing were implemented in assembly language. The file serves as the **contract** between C code (which allocates globals and calls C functions) and `.s` assembly translation units (which need symbol names mangled for ELF linkage, access to those globals, and hard-coded byte offsets into C struct layouts).

## Key Cross-References

### Incoming (who depends on this file)
- **Assembly source files** in `code/unix/`: Any `.s` files (e.g., `snd_mixa.s`, `matha.s`, `sys_dosa.s`, `vm_x86a.s`) include this header to gain access to:
  - The `C(label)` macro for ELF symbol decoration
  - All `.extern` declarations for globals defined in C
  - Struct offset constants for field access
- **Renderer globals** declared here feed into software-rasterization paths in `code/renderer/` (legacy spans, edges, surfaces, z-buffer state)
- **Audio mixer globals** declared here feed into `code/client/snd_mix.c`, `snd_dma.c` (assembly-optimized mixing routines)

### Outgoing (what this file depends on)
- **All symbols declared as `.extern`** are defined in C translation units elsewhere:
  - Renderer: `code/renderer/tr_*.c`, `code/client/` (z-buffer, texture, lighting, span/edge lists)
  - Audio: `code/client/snd_dma.c`, `snd_mix.c` (paintbuffer, sample state)
  - Entry points called from assembly: `D_PolysetSetEdgeTable`, `D_RasterizeAliasPolySmooth` (from renderer)
- **Implicit struct-layout dependencies** on:
  - `code/game/q_shared.h`, `code/game/bg_*.h` (plane_t, dnode_t shared types)
  - `code/renderer/tr_local.h`, `code/renderer/tr_public.h` (edge_t, surf_t, espan_t, refdef_t)
  - `code/client/snd_local.h` (channel_t, sfxcache_t, portable_samplepair_t)

## Design Patterns & Rationale

**Symbol name mangling via preprocessor:**  
The `C(label)` macro handles ELF (`label`) vs. a.out/other formats (`_label`) automatically. This avoids maintaining separate `.s` files per platform and lets a single `.s` source assemble on multiple Unix variants.

**Struct offset constants instead of C types:**  
Assembly code cannot use C struct definitions (no type system), so byte offsets are hardcoded as preprocessor constants (`pl_normal 0`, `pl_dist 12`, etc.). This pattern is **fragile but effective** — the comments repeatedly warn "must be kept in sync" because there's no compile-time check. Any C struct change requires manual offset recomputation.

**Conditional assembly directives:**  
The `#ifndef GLQUAKE` block gates software-renderer symbols. This suggests the codebase supported **two rendering paths**: OpenGL (modern, preferred) and software-rasterized (legacy fallback). On modern GL-only builds, none of the software-renderer `.extern` declarations are needed.

**Rationale for hand-assembly paths:**  
In the late 1990s, x86 CPUs (Pentium II era) lacked good compiler vectorization. Critical innermost loops — texture sampling, span rasterization, sample mixing — benefited from hand-tuned register allocation, unrolling, and prefetch ordering. The software renderer was a performance necessity before dedicated GPUs became standard.

## Data Flow Through This File

**Compile/link-time flow (not runtime):**
1. **Preprocessing**: `qasm.h` is `#include`d at the top of `.s` files; macros expand and symbol references are recorded.
2. **Assembly**: The assembler sees `.extern C(paintbuffer)` (after macro expansion), records it as an unresolved external symbol in the `.o` object file.
3. **Linking**: The linker matches `.extern` references against `paintbuffer` defined in `snd_mix.c` and resolves the address.
4. **Runtime**: Assembly code loads the address of `paintbuffer` (or fields within it using hardcoded offsets) and executes memory operations.

**Example: audio mixing**  
Assembly in `snd_mixa.s` accesses `portable_samplepair_t` fields:
```
psp_left   = 0       # sample pair: left channel at +0 bytes
psp_right  = 4       # right channel at +4 bytes
```
At runtime, if a loop iterates over the `paintbuffer` (an array of `portable_samplepair_t`), assembly uses these offsets to directly read/write audio samples at `*(int32_t*)(ptr + psp_left)`.

## Learning Notes

**What studying this file teaches:**
1. **Pre-GPU rasterization architecture**: Quake III still shipped with a software renderer as fallback. The hundreds of `.extern` declarations (`d_pzbuffer`, `colormap`, `edge_p`, `span_p`, `surface_p`) reveal the massive state machinery required — z-buffers, colormaps, span/edge lists, lighting caches — all accessed from tight assembly loops.

2. **Cross-subsystem tight coupling**: This file exposes how the renderer, audio mixer, and platform layer were deeply interwoven. Unlike modern modular engines, every assembly code path directly reads/writes shared globals. There's no abstraction; performance demanded direct memory access.

3. **Idiomatic to this era**: Modern game engines use:
   - SIMD intrinsics (SSE2, AVX) in C/C++ rather than raw assembly
   - Compiler auto-vectorization with careful loop structure
   - Architecture-agnostic languages or JIT compilation
   - Memory-pooling and cache-conscious data layouts rather than scattered globals

4. **Connection to engine concepts**: This file does **not** reflect modern ECS, scene graphs, or deferred rendering. Instead, it reveals the immediate-mode retained-state architecture of a 1990s real-time renderer: global state updated each frame, assembly code reads it sequentially, outputs to framebuffer/soundbuffer.

5. **Platform abstraction boundary**: The file shows that Unix/Linux support was added after the original DOS/Windows ports. The conditional `C()` macro and all the `#ifndef GLQUAKE` gates hint at retrofitting OpenGL onto a codebase originally written for software rasterization.

## Potential Issues

1. **Struct layout synchronization risk**:  
   Comments warn "!!! must be kept the same as in [C header]" but there's no compile-time check. If a C struct (e.g., `channel_t` in `snd_local.h`) is modified, the offset constants here become invalid, causing silent memory corruption at runtime. Modern engines use `offsetof()` macro or compile-time asserts to prevent this.

2. **Dead code in GL-only builds**:  
   All software-renderer `.extern` declarations (several hundred) are unused if `GLQUAKE` is defined. The codebase carries assembly sources that are never compiled or linked on modern systems, increasing maintenance burden.

3. **Platform specificity**:  
   This file is Unix/Linux x86 only. Equivalent files exist for Win32 (`code/win32/`) and macOS, creating three parallel definitions of the same struct offsets. If a struct changes, all three must be updated independently — a classic DRY violation.

4. **Symbol namespace pollution**:  
   All `.extern` symbols are global. There's no scoping; any assembly code can read/write any global (no encapsulation). This made sense for a single monolithic binary but reduces modularity.
