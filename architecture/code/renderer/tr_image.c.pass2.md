# code/renderer/tr_image.c — Enhanced Analysis

## Architectural Role

This file is the **texture subsystem core** of the Quake III renderer DLL, acting as the sole gatekeeper between asset sources (disk files, procedural generation, screenshots) and the GPU. It owns the image cache, orchestrates the multi-stage upload pipeline (resample → gamma → mipmap → GPU), manages the skin registration system (player model surface→shader mapping), and exposes the primary entry point (`R_FindImageFile`) used by the shader system during map/asset load. Every visible texture in the game flows through this file's initialization and caching logic.

## Key Cross-References

### Incoming (who depends on this file)
- **`tr_shader.c`** — Calls `R_FindImageFile` for every texture reference in `.shader` definitions during material parsing
- **Renderer backend (`tr_backend.c`)** — Binds textures via `GL_Bind` (defined in `tr_main.c` but uses image cache)
- **Screenshot system (`tr_cmds.c`)** — Calls `SaveJPG` to encode framebuffer to disk
- **Client / Game VM** — Calls `RE_RegisterSkin(const char *name)` to load player skins (`.skin` files that map surface names to shaders)
- **Console (`cmd.c` dispatch)** — Calls `R_ImageList_f` via command buffer; calls `GL_TextureMode` via cvar update
- **Renderer init (`tr_init.c`)** — Calls `R_InitImages`, `R_SetColorMappings`, `R_DeleteTextures` during startup/shutdown/vid_restart

### Outgoing (what this file depends on)
- **OpenGL state cache (`tr_main.c`)** — `GL_Bind`, `GL_SelectTexture`, `GL_CheckErrors` manage texture binding and error checking
- **Platform gamma (`win_glimp.c`, `linux_glimp.c`)** — `GLimp_SetGamma` pushes gamma LUT to hardware if supported
- **Shader system (`tr_shader.c`)** — `R_FindShader` called by `RE_RegisterSkin` to resolve skin surface shaders
- **Filesystem (`qcommon/files.c`)** — `ri.FS_ReadFile`, `ri.FS_FreeFile` for on-disk asset loading
- **Memory (`qcommon/common.c`)** — `ri.Hunk_Alloc*`, `ri.Malloc`, `ri.Free` for texture allocation
- **Libjpeg-6 (`../jpeg-6/jpeglib.h`)** — Full JPEG encode/decode pipeline with custom in-memory I/O manager

## Design Patterns & Rationale

**Hash Table with Linear Chains** — `hashTable[1024]` + `image_t.next` chain caches loaded images by filename hash. Rationale: Simple O(1) average lookup avoids redundant disk I/O and GPU uploads. No collision-resolution strategy visible, relying on low collision rate for this codebase's asset scale.

**Procedural Built-in Textures** — `R_CreateBuiltinImages` generates critical textures (white, default, fog, dlight) at startup rather than loading from disk. Rationale: Eliminates file I/O for essential assets; ensures predictable availability.

**Modular Upload Pipeline** — `Upload32` separates concerns (resample, format selection, gamma correction, mipmap generation) with independent stages callable by different code paths. Rationale: Allows per-platform optimization (e.g., disable CPU mipmaps on newer GL, use hardware S3TC) without restructuring.

**Lookup-Table (LUT) Caching** — `s_gammatable[256]` and `s_intensitytable[256]` precompute pixel transformations; rebuilt once per cvar change rather than per-pixel. Rationale: ~1KB footprint trades for massive per-texture upload speedup; idiomatic for 2005-era hardware.

**Temporary Hunk Memory** — `ResampleTexture`, `R_MipMap2` allocate intermediate buffers from `ri.Hunk_AllocateTempMemory()` (frame-scoped fast allocator). Rationale: Avoids long-term heap fragmentation and leverages engine's memory model; automatically freed at frame boundary.

**Deferred Debug Visualization** — `mipBlendColors[16][4]` and `R_BlendOverTexture` enable runtime mip-level tinting via `r_colorMipLevels` cvar without altering the upload pipeline. Rationale: Orthogonal feature; no performance cost when disabled.

## Data Flow Through This File

**Initialization → Runtime:**
1. `R_InitImages()` → zeroes `hashTable`, calls `R_SetColorMappings()` (rebuild gamma/intensity LUTs), calls `R_CreateBuiltinImages()` (insert essential synthetic textures)
2. `R_SetColorMappings()` reads cvars, clamps gamma to [0.5, 3.0], optionally pushes to hardware via `GLimp_SetGamma`

**Asset Load Path (triggered by shader system):**
1. `R_FindImageFile(filename, mipmap, picmip, wrapMode)` → check hash table
2. Cache miss → `R_LoadImage()` → dispatches to format-specific loader (LoadTGA, LoadJPG, LoadPCX32, LoadBMP)
3. `R_CreateImage()` → allocate `image_t` from hunk, call `Upload32()`
4. `Upload32()` multi-stage pipeline:
   - Pad/resample to nearest power-of-two via `ResampleTexture` (bilinear filtering)
   - Apply picmip recursively (quarter image width/height per level)
   - Detect optimal internal format: check for all-alpha pixels → RGB; else → RGBA; or use S3TC if device supports
   - Light-scale via `R_LightScaleTexture` (apply gamma/intensity LUTs) if not lightmap
   - Generate mipmap chain via `R_MipMap` or `R_MipMap2` (2×2 box filter)
   - Issue `qglTexImage2D` for each mip level
5. Insert `image_t` into hash chain; return to caller (shader system inserts reference)

**Cvar Updates:**
- `r_gamma`, `r_intensity`, `r_overBrightBits` → trigger `R_SetColorMappings()`, which rebuilds LUTs and pushes gamma to hardware
- `r_textureMode` → triggers `GL_TextureMode()`, which iterates all mipmapped textures in cache and updates their GL_TEXTURE_MIN/MAG_FILTER

## Learning Notes

**Pre-GPU Upload Image Processing** — Unlike modern engines (which cook textures offline to target formats), this engine performs gamma correction, resampling, and mipmap generation at runtime. This trades flexibility (dynamic gamma, no cook pipeline) for CPU overhead and cache pollution. Reflects the original Q3A's art pipeline and hardware era.

**Manual Mipmap Generation** — `R_MipMap` (CPU box filter) or `R_MipMap2` (CPU Gaussian-weighted filter) replace modern `glGenerateMipmap()`. Rationale: Fine-grained control for debug (`r_simpleMipMaps` cvar) and reproducibility across platforms.

**S3TC Assumed Omnipresent** — Code checks `glConfig.textureCompression` but offers no fallback to uncompressed; reflects Q3A's minimum-spec assumption (GeForce3, Radeon 8500+). Modern engines support a menu of formats (DXT, BC4–BC7, ASTC, ETC2, PVRTC).

**Skin File Format** — `RE_RegisterSkin` parses `.skin` files (hardcoded text format mapping surface names to shaders). This is a high-level asset composition pattern. Modern engines use material instances or shader parameter overrides instead.

**Hash Table Over Hash Map** — Uses chaining + linear probing (implicit via `image_t.next`), not open addressing. No growth strategy; assumes hash space (1024 buckets) suffices for the game's asset scale.

## Potential Issues

**Format Fallback Silent Failure** — `R_LoadImage` tries `.tga`, then `.jpg` on TGA failure, then falls back to `LoadPCX32`/`LoadBMP`. If all fail, returns `NULL` with no logged warning; caller (`R_FindImageFile`) substitutes `tr.defaultImage`. Missing textures fail silently, making asset errors hard to debug in complex maps.

**No Image Validation in Loaders** — LoadBMP, LoadTGA do not validate buffer sizes before reading; a malformed file header could cause out-of-bounds reads. JPEG is safer (libjpeg validates).

**ResampleTexture Hardcoded Limit** — Asserts `outwidth <= 2048` without explanation. High-res textures (4K+) would trigger this. Modern patch: increase constant or fail gracefully.

**SaveJPG Lacks Error Handling** — If JPEG compression fails partway through, `term_destination` callback may not be invoked, leaving `hackSize` uninitialized. The allocated buffer could leak. Should check `jpeg_mem_dest` state before freeing.

**No Texture Eviction Policy** — `tr.numImages` only increments; the hash table never evicts stale images even if VRAM is exhausted. Large asset packs could overflow VRAM without warning. Modern engines use LRU or explicit unload calls.
