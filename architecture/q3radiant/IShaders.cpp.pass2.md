# q3radiant/IShaders.cpp — Enhanced Analysis

## Architectural Role

This file implements the shader/texture lookup interface for the **Q3Radiant level editor**. It provides the editor's texture cache and on-demand loading mechanism—a client-side concern entirely separate from the runtime renderer's shader system. The function bridges the editor's project configuration (texturepath) with its viewport material preview, enabling texture browsing and assignment during brush/surface painting.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant UI layer** calls `QERApp_TryTextureForName` when users:
  - Browse the texture palette and select a shader
  - Paint textures onto brushes in the viewport
  - Preview shader appearance in the 3D editor view
- The function is exposed via the editor's public plugin interface (`IShaders.h`)

### Outgoing (what this file depends on)
- **g_qeglobals** global state: reads `d_qtextures` linked list (editor's texture cache) and `d_project_entity` (project metadata)
- **Filesystem functions**: `ValueForKey` (read project entity key-value pairs), `QE_ConvertDOSToUnixName` (path normalization)
- **Image loading**: `LoadImage` (low-level pixel buffer allocation), `Texture_LoadTGATexture` (texture object creation)
- **Shader metadata**: `SetNameShaderInfo` (populate shader definition from file)
- **Logging**: `Sys_Printf` (debug output)

## Design Patterns & Rationale

**Layered Caching with Format Fallback:**  
The function implements a two-tier lookup: in-memory cache (fast path via linked-list scan) followed by filesystem I/O (slow path with TGA→JPG fallback). This reflects mid-2000s editor design where texture assets were duplicated: shaders as multi-pass renderer definitions *and* simple flat textures for viewport preview. Modern editors collapse these into a single runtime format.

**Manual Texture Lifecycle:**  
`d_qtextures` is a global linked list manually maintained by the editor. The function allocates new `qtexture_t` nodes on first load and assumes they persist for the entire editing session. No garbage collection or eviction policy is visible here, indicating textures are only freed on editor shutdown.

**Pragmatic Unsafe Code:**  
The function exhibits defensive programmers working around limitations:
- `sprintf` constructs a full path without bounds checking
- Direct character array manipulation (`filename[nLen-3] = 'j'`) replaces the extension
- Comments acknowledge technical debt ("this is dirty")

## Data Flow Through This File

```
User selects texture in editor
       ↓
QERApp_TryTextureForName(name)
       ↓
[Cache hit?] → YES → return cached qtexture_t*
       ↓ NO
Construct filename from project's texturepath
       ↓
Try LoadImage(path.tga) → [Success?] → YES → wrap in qtexture_t
       ↓ NO
Try LoadImage(path.jpg) → [Success?] → YES → wrap in qtexture_t
       ↓ NO
Return NULL (texture not found)
```

**State mutation:** On cache miss, the function allocates a new `qtexture_t` node and inserts it into `g_qeglobals.d_qtextures` linked list (implicit via `Texture_LoadTGATexture`).

## Learning Notes

**Editor vs. Runtime Separation:**  
This file illustrates a design principle often overlooked: editors and runtime engines have *different* asset pipelines. Q3Radiant's texture system is purely for viewport feedback and doesn't directly feed into the renderer's shader compilation. Runtime shaders are parsed from `.shader` files during game initialization; editor textures are simple rasterized previews loaded on-demand.

**Format Multiplicity in Early 2000s:**  
The TGA→JPG fallback reflects the era's disk-space constraints and hardware limitations. Modern engines ship textures in compressed formats (DXT, BC) and load them platform-specifically. Here, raw pixel buffers are loaded and assumed compatible across Windows/Linux.

**Idiomatic Globals and Singletons:**  
`g_qeglobals` is a monolithic editor state struct. Modern UI frameworks use dependency injection or scoped component state; this code assumes a single project is loaded at a time with a global singleton managing all resources.

## Potential Issues

1. **Buffer Overflow Risk** (lines 59–60, 70):  
   Fixed `char filename[1024]` and `cWork[1024]` arrays. A `texturepath` key longer than ~1000 bytes or a texture `name` with long path components will cause stack smashing.

2. **Unsafe String Manipulation** (lines 68–72):  
   Replaces file extension by direct character assignment without bounds checking or validation that the original filename is long enough.

3. **Memory Leak on Partial Failure** (line 79):  
   If `Texture_LoadTGATexture` succeeds but `SetNameShaderInfo` fails, the allocated `pPixels` is freed but the `qtexture_t` node leaks into the cache.

4. **Missing Error Propagation** (lines 77–81):  
   The function returns NULL if no texture is found, but callers cannot distinguish between "texture doesn't exist," "I/O error," or "corrupt file." The `Sys_Printf` call on success is asymmetric (no error logging).
