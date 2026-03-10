# q3radiant/GLInterface.cpp — Enhanced Analysis

## Architectural Role

This file implements the **plugin drawing hook system** for Q3Radiant's viewport windows. It acts as a registry that allows optional plugin modules (like texture-editing tools) to inject custom rendering callbacks into the 2D/3D views without modifying the core editor. The system uses MFC's `CPtrArray` for dynamic hook management and reference counting (`IncRef`/`DecRef`) to ensure safe lifecycle management of plugin objects. This exemplifies the editor's extensibility layer—distinct from the runtime engine, part of the RadiantDoc/MainFrm/ChildFrm UI hierarchy.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main UI loop** (`MainFrm.cpp`, `ChildFrm.cpp`, viewport handlers): calls `DrawPluginEntities(VIEWTYPE)` once per frame for each window type (XY, camera, Z view)
- **Plugin DLLs** (e.g., TexTool, surface plugins): call `QERApp_HookXYGLWindow()` during plugin init and `QERApp_UnHookGLWindow()` on shutdown
- **Face selection system**: the global `g_ptrSelectedFaces` (populated elsewhere) is read by `QERApp_ISelectedFace_GetTextureNumber()`

### Outgoing (what this file depends on)
- **MFC types**: `CPtrArray` (dynamic array from Windows MFC; not in the runtime engine)
- **Radiant types**: `face_t` structure (map editor entity; `d_texture->texture_number` field), `IGLWindow` interface (plugin callback vtable)
- **Platform**: `Sys_Printf()` for debug output (shared platform layer)
- **Global editor state**: `g_ptrSelectedFaces` (selected faces array, populated by brush/face selection code)

## Design Patterns & Rationale

**Observer/Hook Pattern**:
- Plugin modules register themselves as subscribers to viewport rendering events
- The `CPtrArray l_GLWindows` acts as a broadcast registry; `DrawPluginEntities()` is the "notify all" call
- Rationale: Allows TexTool and similar plugins to render overlays (texture preview, deformation grids) without fork-and-modify of core viewport code

**Reference Counting**:
- `IncRef()`/`DecRef()` on hook add/remove suggests `IGLWindow` is a COM-like interface or custom ref-counted object
- Prevents premature deletion of plugin objects while draw callbacks are pending

**Linear Iteration for Dispatch**:
- The loop in `DrawPluginEntities()` is simple and sufficient for a small number of plugins (typically 0–2 in practice)
- No hash table, no priority ordering—plugins draw in registration order

## Data Flow Through This File

1. **Initialization**: Plugin calls `QERApp_HookXYGLWindow(pGLW)` → stored in `l_GLWindows`, ref count incremented
2. **Per-Frame Rendering**: Main viewport loop calls `DrawPluginEntities(vt)` with current view type (XY/camera/Z)
3. **Hook Dispatch**: `DrawPluginEntities()` iterates `l_GLWindows` and invokes `IGLWindow::Draw(vt)` on each
4. **Cleanup**: Plugin calls `QERApp_UnHookGLWindow(pGLW)` → removed from array, ref count decremented
5. **Texture Query**: UI or plugins can call `QERApp_ISelectedFace_GetTextureNumber()` to get the active face's texture ID

## Learning Notes

**Editor-Specific Patterns**:
- This file demonstrates how a professional level editor (from id Tech 3 era) exposes a stable plugin API
- The reference-counting + interface design influenced later editor extensibility frameworks (Unreal, CryEngine)
- Unlike the runtime engine (which is VM-based for game mods), the editor uses direct DLL loading and COM-style vtables

**Historical Context**:
- The TODO comment (`add support for camera view, Z view ... (texture view?)`) reveals the code was written before full multi-view support was implemented
- This is a "quick hack" (per the comments), suggesting rapid prototyping during active development circa 2000

**MFC/Win32 Patterns**:
- Uses MFC `CPtrArray` (Windows-only, dynamic array)—not portable to other platforms
- `WINAPI` calling convention is explicit (Windows-only DLL interface)
- No cross-platform abstraction, typical of late-1990s Windows-centric tools

## Potential Issues

- **Linear Search**: `UnHookGLWindow()` uses linear scan; becomes O(n) with many plugins (unlikely in practice, but poor scaling)
- **No Bounds Checks**: `l_GLWindows.GetAt(i)` can return null or invalid pointer if array was modified during iteration; no guard against this
- **Global State**: `l_GLWindows` is not thread-safe; if the main viewport thread and a plugin thread both call Hook/Unhook, races occur
- **Incomplete API**: The TODO suggests camera and Z views were not yet hooked; if only XY was implemented, 3D viewport plugins could not render
- **Texture Number Fallback**: `QERApp_ISelectedFace_GetTextureNumber()` returns `0` if no face is selected; the comment `"++timo hu ? find the appropriate gl bind number"` indicates uncertainty about the correct return value for edge cases
