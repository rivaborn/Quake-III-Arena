# code/renderer/tr_font.c

## File Purpose
Implements the font registration and rendering system for Quake III Arena's renderer. It supports both runtime TrueType rasterization via FreeType 2 (compile-time opt-in via `BUILD_FREETYPE`) and the standard path of loading pre-rendered glyph bitmaps and atlas textures from disk.

## Core Responsibilities
- Load pre-rendered font `.dat` files and associated TGA atlas images from `fonts/`
- Cache up to `MAX_FONTS` registered fonts to avoid redundant loads
- Register glyph shader handles via `RE_RegisterShaderNoMip` for each loaded font
- (When `BUILD_FREETYPE`) Rasterize TrueType glyphs using FreeType, pack them into 256×256 GL texture pages, and optionally write `.dat`/`.tga` output files
- Provide endian-safe binary deserialization helpers (`readInt`, `readFloat`) for `.dat` files
- Initialize and shut down the FreeType library (`R_InitFreeType`, `R_DoneFreeType`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fontInfo_t` | struct (defined elsewhere) | Per-font data: glyph array, scale, name |
| `glyphInfo_t` | struct (defined elsewhere) | Per-glyph metrics and shader handle |
| `poor` | union | Byte-reinterpretation union for endian-safe `float` reads |
| `FT_Library` | typedef (FreeType, `#ifdef`) | FreeType library instance |
| `FT_Face` | typedef (FreeType, `#ifdef`) | FreeType font face |
| `FT_Bitmap` | struct (FreeType, `#ifdef`) | Rasterized glyph bitmap |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `registeredFontCount` | `int` | static (file) | Count of currently cached fonts |
| `registeredFont[MAX_FONTS]` | `fontInfo_t[6]` | static (file) | Cache of loaded font descriptors |
| `ftLibrary` | `FT_Library` | global (`#ifdef BUILD_FREETYPE`) | Singleton FreeType library handle |
| `fdOffset` | `int` | static (file) | Current byte offset into `fdFile` during `.dat` parsing |
| `fdFile` | `byte *` | static (file) | Pointer to raw `.dat` file buffer during parsing |

## Key Functions / Methods

### RE_RegisterFont
- **Signature:** `void RE_RegisterFont(const char *fontName, int pointSize, fontInfo_t *font)`
- **Purpose:** Primary entry point. Loads font data by point size; checks cache first, then tries pre-rendered `.dat` file, then falls back to FreeType rasterization if compiled in.
- **Inputs:** `fontName` — path to `.ttf` (FreeType path only); `pointSize` — requested size; `font` — output struct.
- **Outputs/Return:** Fills `*font` in place; registers into `registeredFont[]`.
- **Side effects:** Calls `R_SyncRenderThread()`; allocates/frees heap via `Z_Malloc`/`Z_Free`; calls `ri.FS_ReadFile`/`ri.FS_FreeFile`; registers GL textures and shaders; may write `.dat` and `.tga` files if `r_saveFontData->integer`.
- **Calls:** `R_SyncRenderThread`, `Q_stricmp`, `ri.FS_ReadFile`, `readInt`, `readFloat`, `RE_RegisterShaderNoMip`, `RE_ConstructGlyphInfo` (FT), `R_CreateImage` (FT), `RE_RegisterShaderFromImage` (FT), `WriteTGA` (FT), `ri.FS_WriteFile` (FT).
- **Notes:** Cache key is `fonts/fontImage_<pointSize>.dat`, not the font name. `glyphScale` is computed as `(72/dpi) * (48/pointSize)`, normalizing to a 48-point baseline. FreeType path packs glyphs row-first into 256×256 pages; overflow triggers a new image.

### R_InitFreeType
- **Signature:** `void R_InitFreeType()`
- **Purpose:** Initializes FreeType library and resets font cache count.
- **Side effects:** Sets `ftLibrary` (FT); zeroes `registeredFontCount`.
- **Notes:** Called during renderer init. Always resets the font count regardless of `BUILD_FREETYPE`.

### R_DoneFreeType
- **Signature:** `void R_DoneFreeType()`
- **Purpose:** Shuts down FreeType and clears font cache count.
- **Side effects:** Calls `FT_Done_FreeType`; nulls `ftLibrary`; zeroes `registeredFontCount`.

### readInt / readFloat
- **Purpose:** Sequential little-endian (PPC-swapped) reads from `fdFile` at `fdOffset`.
- **Notes:** `readFloat` uses the `poor` union for type-punning; PPC path byte-swaps all four bytes.

### R_GetGlyphInfo / R_RenderGlyph / RE_ConstructGlyphInfo / WriteTGA
- All `#ifdef BUILD_FREETYPE` only.
- `R_GetGlyphInfo` extracts floored/ceiled metrics from a `FT_GlyphSlot`.
- `R_RenderGlyph` allocates and renders an outline glyph into a new `FT_Bitmap`.
- `RE_ConstructGlyphInfo` places one glyph into a 256×256 packing buffer; returns `NULL`-slot signal via `xOut/yOut = -1`.
- `WriteTGA` writes a raw 32-bit TGA via `ri.FS_WriteFile` with BGR swap.

## Control Flow Notes
- `R_InitFreeType` is called during renderer startup (`R_Init`), `R_DoneFreeType` during shutdown.
- `RE_RegisterFont` is called from the UI/cgame modules during asset registration (not per-frame).
- No per-frame code exists in this file.

## External Dependencies
- `tr_local.h` — renderer internals: `image_t`, `ri`, `R_SyncRenderThread`, `R_CreateImage`, `RE_RegisterShaderNoMip`, `RE_RegisterShaderFromImage`, `r_saveFontData`
- `qcommon/qcommon.h` — `Z_Malloc`, `Z_Free`, `Com_Memset`, `Com_Memcpy`, `Com_sprintf`, `Q_stricmp`, `Q_strncpyz`
- `fontInfo_t`, `glyphInfo_t`, `GLYPHS_PER_FONT`, `GLYPH_START`, `GLYPH_END` — defined elsewhere (likely `game/q_shared.h` or `qcommon/qfiles.h`)
- FreeType 2 headers (`ft2/freetype.h`, etc.) — only when `BUILD_FREETYPE` is defined; not shipped in release builds
