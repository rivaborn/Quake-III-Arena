# libs/pak/pakstuff.cpp — Enhanced Analysis

## Architectural Role

This file is a **tools-only offline utility library** for managing Quake III Arena package archives (both binary PAK and ZIP-based PK3 formats). It supports the map compiler (q3map), level editor (q3radiant), and bot compiler (bspc) in reading assets and content from packaged files during development. It has **zero runtime role**—the shipped engine never links or calls this code; the runtime filesystem is handled entirely by `code/qcommon/files.c`.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map** (map compiler): loads textures and shader definitions from PK3/PAK during BSP compilation
- **q3radiant** (level editor): asset browser, shader/texture loading from packages
- **bspc** (bot AAS compiler): may load entity/light data from pakfiles during preprocessing
- (Possibly other offline tools in the build chain)

### Outgoing (what this file depends on)
- **unzip.c/h** (`code/qcommon/`) for ZIP archive format handling (PK3 support)
- **str.h** (libs/) for `Str` string utility class
- **Standard library**: `stdio.h`, `stdlib.h`, `string.h`, Windows-specific `io.h`

## Design Patterns & Rationale

1. **Global File Handle Pool** (`pakfile[16]`, `pakdir`, `pakdirptr`)  
   Simple but brittle: fixed array of 16 open file pointers. No lifetime tracking or RAII; relies on manual `pakopen` flag. Reflects late-1990s C practices.

2. **Template Linked Lists** (`StrPtr<T>`, `PtrList<T>`, `ZFileList`, `StrList`, `PK3List`)  
   Type-safe containers built from scratch (no STL). Used for dynamic tracking of PK3 files and texture paths. Each node owns its data (`~PK3FileInfo()` deletes `m_pName`).

3. **Exhaustive Magic Number Detection** (`ReadMagic()`)  
   ~20 file formats recognized by byte signatures: IWAD, PWAD, PACK, WAD2, BSP, MODEL, SPRITE, WAV, AU, PBM/PGM/PPM (both ASCII and raw), BMP, GIF, PCX. Hard-coded, not extensible—typical for a 2005 closed tool chain.

4. **Memory Pooling with Block Alignment** (`__qblockmalloc` rounds up to 4096-byte boundaries)  
   Reduces fragmentation for repeated small allocations; assumes memory is cheap, cleanup is manual.

5. **Path Manipulation Utilities** (extract dir/name/base/extension, DOS↔Unix conversion)  
   Portable path handling before std::filesystem; accommodates both `/` and `\` separators.

## Data Flow Through This File

```
User requests pak load
  ↓
OpenFileReadMagic() → ReadMagic() → file type detected
  ↓
Type-specific handler (PAK vs PK3/ZIP)
  ↓
pakdir/pakdirptr populated with directory entries
  ↓
Cache maintained in g_PK3Files (PK3List) and g_PK3TexturePaths (StrList)
  ↓
Tools query by name/path; file seek/read on demand
```

Global state (`g_bPK3`, `g_strBasePath`, `HavePakColormap`) persists across calls, avoiding repeated initialization. Early return on error sets `pakopen = false`.

## Learning Notes

- **Tools ≠ Engine**: This codebase reflects the offline build pipeline, completely separate from shipped runtime (`qcommon/files.c`). Tools had no performance constraints; runtime had strict memory budgets.
- **Archive Format Flexibility**: Supporting both PAK (id's binary format) and PK3 (ZIP) simultaneously was a pragmatic choice circa 2001–2005—PK3 was easier for modders; PAK was id's original standard.
- **Pre-STL C++**: No `std::vector`, `std::map`, or exception handling. Manual memory, global state, and linked-list templates. Common for legacy game engines where compile-time overhead and binary size mattered.
- **Magic Number Pattern**: Robust for a fixed set of known formats, but inflexible. Modern equivalents would use file extension + content validation or pluggable format handlers.

## Potential Issues

1. **No Thread Safety**: Global state (`pakfile[]`, `g_bPK3`, `dirhead`) is unguarded. q3radiant (multi-threaded UI) could corrupt if multiple threads open pakfiles simultaneously.
2. **Fixed Array Limits**: `pakfile[16]` silently fails if >16 files opened. No overflow detection.
3. **Memory Leak Risk**: `__StrDup()` and `new []` in template destructors assume caller cleanup. No smart pointers; manual `delete []` required.
4. **Path Separator Fragility**: Code checks for both `/` and `\`, but behavior is inconsistent across platforms (Unix never produces `\`; hardcoded DOS logic may be dead code on Linux).
