# code/renderer/tr_image.c

## File Purpose
This file implements the renderer's complete image management system for Quake III Arena, handling loading, processing, uploading, and caching of all game textures. It supports BMP, PCX, TGA, and JPEG formats, manages OpenGL texture objects, and owns the skin registration system.

## Core Responsibilities
- Load raw image data from disk in multiple formats (BMP, PCX, TGA, JPEG)
- Resample, mipmap, and gamma/intensity-correct images before GPU upload
- Upload processed pixel data to OpenGL via `qglTexImage2D`
- Cache loaded images in a hash table to avoid redundant loads
- Create and manage procedural built-in textures (dlight, fog, default, white)
- Manage skin (`.skin` file) registration and lookup
- Build gamma/intensity lookup tables used during texture upload

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `image_t` | struct (defined in tr_local.h) | Represents a single GPU texture: dimensions, GL name, mipmap flags, hash chain |
| `BMPHeader_t` | struct | Parsed BMP file header for LoadBMP |
| `textureMode_t` | struct | Associates a GL filter mode name string with min/mag filter enums |
| `my_destination_mgr` | struct | Custom libjpeg destination manager for in-memory JPEG compression |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_intensitytable` | `byte[256]` | static | LUT mapping input intensity to scaled intensity |
| `s_gammatable` | `unsigned char[256]` | static | LUT mapping input value to gamma-corrected value |
| `gl_filter_min` | `int` | global | Current GL minification filter; default `GL_LINEAR_MIPMAP_NEAREST` |
| `gl_filter_max` | `int` | global | Current GL magnification filter; default `GL_LINEAR` |
| `hashTable` | `image_t*[1024]` | static | Hash table of all loaded `image_t` records, keyed by filename |
| `mipBlendColors` | `byte[16][4]` | file | Debug colors blended over each mip level when `r_colorMipLevels` is set |
| `hackSize` | `int` | static | Byte count written by JPEG compressor's `term_destination` callback |

## Key Functions / Methods

### R_InitImages
- **Signature:** `void R_InitImages(void)`
- **Purpose:** Initializes the entire image subsystem at renderer startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Zeroes `hashTable`; calls `R_SetColorMappings` to build LUTs; calls `R_CreateBuiltinImages`.
- **Calls:** `R_SetColorMappings`, `R_CreateBuiltinImages`
- **Notes:** Must be called before any image can be loaded or referenced.

### R_SetColorMappings
- **Signature:** `void R_SetColorMappings(void)`
- **Purpose:** Rebuilds `s_gammatable` and `s_intensitytable` from current cvars; also configures `tr.overbrightBits` and pushes gamma to hardware.
- **Inputs:** Reads `r_gamma`, `r_intensity`, `r_overBrightBits`, `glConfig`.
- **Outputs/Return:** None; updates global LUT arrays and `tr` fields.
- **Side effects:** Calls `GLimp_SetGamma` if hardware gamma supported; clamps `r_gamma` cvar to 0.5–3.0.
- **Calls:** `ri.Cvar_Set`, `GLimp_SetGamma`

### R_CreateImage
- **Signature:** `image_t *R_CreateImage(const char *name, const byte *pic, int width, int height, qboolean mipmap, qboolean allowPicmip, int glWrapClampMode)`
- **Purpose:** Sole constructor for `image_t` objects; allocates from hunk, binds a GL texture, uploads pixel data, and inserts into hash table.
- **Inputs:** Name string, raw RGBA pixel buffer, dimensions, mip/picmip/wrap flags.
- **Outputs/Return:** Pointer to new `image_t`.
- **Side effects:** Increments `tr.numImages`; calls `Upload32` which issues `qglTexImage2D`; modifies `hashTable`.
- **Calls:** `GL_Bind`, `Upload32`, `GL_SelectTexture`, `qglTexParameterf`, `generateHashValue`

### R_FindImageFile
- **Signature:** `image_t *R_FindImageFile(const char *name, qboolean mipmap, qboolean allowPicmip, int glWrapClampMode)`
- **Purpose:** Primary external entry point; returns cached image if already loaded, otherwise loads from disk and calls `R_CreateImage`.
- **Inputs:** Filename and texture parameter flags.
- **Outputs/Return:** `image_t*` or `NULL` on failure.
- **Side effects:** May allocate and upload a new texture; tries uppercase extension fallback on failure.
- **Calls:** `generateHashValue`, `R_LoadImage`, `R_CreateImage`, `ri.Free`

### Upload32
- **Signature:** `static void Upload32(unsigned *data, int width, int height, qboolean mipmap, qboolean picmip, qboolean lightMap, int *format, int *pUploadWidth, int *pUploadHeight)`
- **Purpose:** Core upload pipeline: power-of-two resize, picmip downscale, format selection (RGB/RGBA/S3TC), light scaling, mipmap chain generation, and final `qglTexImage2D` calls.
- **Inputs:** Pixel buffer, dimensions, processing flags.
- **Outputs/Return:** Selected internal format and actual upload dimensions via out-params.
- **Side effects:** Allocates/frees temp memory; issues multiple `qglTexImage2D` calls for mip levels; sets GL filter params.
- **Calls:** `ResampleTexture`, `R_MipMap`, `R_LightScaleTexture`, `R_BlendOverTexture`, `qglTexImage2D`, `qglTexParameterf`, `GL_CheckErrors`
- **Notes:** Uses `goto done` to skip mip generation for non-mipmapped textures.

### R_LoadImage
- **Signature:** `void R_LoadImage(const char *name, byte **pic, int *width, int *height)`
- **Purpose:** Dispatches to the correct format loader based on file extension; `.tga` falls back to `.jpg` if TGA load fails.
- **Calls:** `LoadTGA`, `LoadJPG`, `LoadPCX32`, `LoadBMP`

### R_CreateBuiltinImages
- **Signature:** `void R_CreateBuiltinImages(void)`
- **Purpose:** Generates all engine procedural textures at startup: default, white, identity-light, scratch (×32), dlight, fog.
- **Side effects:** Populates `tr.defaultImage`, `tr.whiteImage`, `tr.identityLightImage`, `tr.scratchImage[]`, `tr.dlightImage`, `tr.fogImage`.

### SaveJPG / LoadJPG
- **Purpose:** Encode a screen buffer to JPEG on disk; decode a JPEG file from disk to RGBA in memory.
- **Notes:** Uses embedded libjpeg-6 with a custom in-memory destination manager (`my_destination_mgr` / `hackSize`). `SaveJPG` is used by the screenshot system.

### RE_RegisterSkin
- **Signature:** `qhandle_t RE_RegisterSkin(const char *name)`
- **Purpose:** Loads and parses a `.skin` file (or treats a non-`.skin` name as a single shader), returning a handle.
- **Side effects:** Allocates `skin_t` and `skinSurface_t` from hunk; calls `R_FindShader`.

## Control Flow Notes
- **Init:** `R_InitImages` → `R_SetColorMappings` + `R_CreateBuiltinImages` called during `R_Init`.
- **Per-asset load:** `R_FindImageFile` is called by the shader system during map/model load; it checks the hash, loads from disk, and uploads to GPU once.
- **Shutdown:** `R_DeleteTextures` deletes all GL texture objects and zeroes `tr.images`.
- **Screenshot path:** `SaveJPG` is called from the backend screenshot command, outside the normal load pipeline.

## External Dependencies
- `tr_local.h` — `image_t`, `tr`, `glConfig`, `glState`, `ri` (refimport), all renderer cvars
- `../jpeg-6/jpeglib.h` — libjpeg-6 compression/decompression API (included with `JPEG_INTERNALS` defined)
- OpenGL via QGL wrappers: `qglTexImage2D`, `qglTexParameterf`, `qglDeleteTextures`, `qglBindTexture`, `qglActiveTextureARB`
- `GL_Bind`, `GL_SelectTexture`, `GL_CheckErrors`, `GLimp_SetGamma` — defined in other renderer files
- `R_FindShader`, `R_SyncRenderThread` — defined in `tr_shader.c` / render thread code
- `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.Malloc`, `ri.Free`, `ri.Hunk_Alloc`, `ri.Hunk_AllocateTempMemory`, `ri.Hunk_FreeTempMemory`, `ri.Error`, `ri.Printf` — engine import table
