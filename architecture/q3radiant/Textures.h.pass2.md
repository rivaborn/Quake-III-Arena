# q3radiant/Textures.h — Enhanced Analysis

## Architectural Role

This header defines the **level editor's texture asset management subsystem**. Unlike the runtime renderer (which compiles and caches shader definitions at game startup), the editor must dynamically load, display, and apply textures to geometry on-demand as designers browse and paint surfaces. The header bridges editor UI (texture browser, properties panels) with the underlying texture/shader loading pipeline and plugin architecture.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant editor UI modules** (`TextureBar.cpp`, `SurfaceDlg.cpp`, `TexEdit.cpp`, `TexWnd.cpp`) — depend on `Texture_*` display and enumeration functions
- **Brush/face painting** (`Brush.cpp`, `SELECT.cpp`) — calls `Texture_SetTexture` when applying textures to selected surfaces
- **Shader editor** (`ShaderEdit.cpp`) — calls `LoadShaders`, `ReloadShaders` to refresh shader definitions after edits
- **Plugin system** (`PlugIn.cpp`) — calls `Texture_LoadFromPlugIn` to inject vendor/tool-specific texture loaders

### Outgoing (what this file depends on)
- **Shader/material loading infrastructure** — `LoadShaders`, `ReloadShaders` likely delegate to shader-parsing code (not in this header)
- **Filesystem layer** — underlying functions read `.shader`, `.tga`, `.jpg` files from disk
- **OpenGL/renderer wrappers** — texture display in the 3D viewport and texture browser
- **Plugin ABI** (`IPluginTexdef` forward declaration) — extensibility point for custom texture providers
- **In-memory texture cache** (global state) — `Texture_ForName` retrieves from or populates cache

## Design Patterns & Rationale

**Lazy Loading with Caching:** `Texture_ForName(const char *name, bool bReplace, bool bShader, bool bNoAlpha, bool bReload, bool makeShader)` demonstrates per-option lazy loading — textures are decoded and cached only when first requested, with flags controlling behavior (reload on demand, suppress alpha, auto-create fallback shader).

**State Machine for UI Navigation:** `Texture_StartPos`/`Texture_NextPos` implement stateful iteration over cached textures, enabling the texture browser to enumerate assets without allocating an array — classic pagination pattern in resource-constrained 2000s GUI code.

**"Inuse" Tracking:** `Texture_ClearInuse`, `Texture_ShowInuse` maintain a secondary pass to identify which textures are actually referenced in the current map, enabling memory cleanup (`Texture_FlushUnused`) and visual highlighting in the UI.

**Dual Shader/Texture Paths:** The `bShader` flag in `Texture_ForName` reflects the editor's dual mode — assets can be accessed as raw image files (`.tga`, `.jpg`) or as shader definitions (`.shader` with imagery references), matching the game's own texture lookup logic.

**Plugin Extensibility:** The `IPluginTexdef` parameter in `Texture_SetTexture` and the separate `Texture_LoadFromPlugIn` function allow third-party plugins (e.g., 3D art packages, procedural texture tools) to inject custom texture sources — architectural evidence of the editor's modular design.

## Data Flow Through This File

1. **Startup:** `Texture_Init` (optionally `bHardInit=false` for soft reload) loads the base shader database and initializes the texture cache.
2. **User Interaction (Texture Browser):**
   - `Texture_ShowDirectory(int menunum)` populates a UI list for a specific shader directory
   - User selects a texture → `Texture_ForName` retrieves/caches it
   - `Texture_StartPos`/`Texture_NextPos` iterate the cache for UI rendering
3. **Painting Surfaces:**
   - Designer selects a texture and clicks a brush face
   - `Texture_SetTexture(texdef_t*, brushprimit_texdef_t*, ...)` applies the texture definition to the face
   - Optional `IPluginTexdef` handler is invoked if a plugin has registered custom behavior
4. **Cleanup/Reload:**
   - `Texture_FlushUnused` frees cached but unreferenced textures
   - `Texture_Flush(bReload)` optionally clears and re-initializes the entire cache
   - `LoadShaders`/`ReloadShaders` re-parse `.shader` files (e.g., after external edit)

## Learning Notes

**Editor-vs-Runtime Decoupling:** This header exemplifies the separation of concerns in Quake III: the **editor** is a completely independent Win32/MFC application with its own rendering pipeline and asset management, unrelated to the runtime engine in `code/`. A modern engine would use the same shader/texture path for both editor and game; Q3 duplicates logic because the editor predates the current shader system.

**Flags-Based Configuration:** The proliferation of boolean parameters (`bReplace`, `bShader`, `bNoAlpha`, `bReload`, `makeShader`, `bFitScale`, `bSetSelection`, `bLinked`) reflects pre-C++20 C practices — no enums, no struct packing. Modern code would use a config struct or builder pattern.

**Forward Declaration of Plugin API:** The `class IPluginTexdef;` forward declaration signals COM-style plugin ABI — the actual `IPluginTexdef` vtable is defined elsewhere, and callers never need to know its full definition, enabling binary-stable plugin loading.

**Texture Pool Abstractions:** Commented-out structs (`texdef_t`, `texturewin_t`, `qtexture_t`) show evolution; the modern code likely uses different internal representations, but the header preserves the old interface for backward compatibility or as documentation.

## Potential Issues

- **No memory bounds on texture cache:** `Texture_FlushUnused` suggests ad-hoc cache eviction; large projects could exhaust memory if the cache grows unbounded.
- **Stale plugin references:** `Texture_LoadFromPlugIn` passes raw `LPVOID` — no validation that the plugin is still loaded or the pointer valid; a plugin unload crash is possible.
- **Mismatch between editor and game shaders:** `LoadShaders` parses `.shader` files from disk at edit time, but the game may have different shader compilation (e.g., platform-specific fallbacks, `bReload` flags); designers may paint textures that don't render correctly at runtime.

---

*This file is best understood in the context of q3radiant as a standalone Win32 MFC application predating modern modular game editors.*
