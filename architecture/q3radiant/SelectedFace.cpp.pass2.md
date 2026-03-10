# q3radiant/SelectedFace.cpp — Enhanced Analysis

## Architectural Role

This file implements a thin plugin interface layer for the Q3Radiant level editor, enabling external texture manipulation tools (DLLs like TexTool) to read/write selected face geometry and texture properties. It acts as a data bridge between the editor's internal face representation and third-party plugins that need access to brush primitive texturing. The file is idiomatic of late-1990s plugin architecture—exporting WINAPI-convention functions with no state, relying entirely on global editor state (`g_qeglobals`, `g_ptrSelectedFaces`).

## Key Cross-References

### Incoming (who depends on this file)
- **Texture tool plugins** (e.g., TexTool.dll, mentioned in code comments) dynamically load and call `QERApp_GetQeglobalsHGLRC`, `QERApp_GetFaceInfo`, `QERApp_SetFaceInfo`, `QERApp_GetTextureSize`
- Plugin manager / DLL loader in Q3Radiant core discovers and invokes these exported functions

### Outgoing (what this file depends on)
- **Editor globals** (`g_qeglobals`, `g_ptrSelectedFaces`, `g_ptrSelectedFaceBrushes`) — shared editor state
- **Editor functions**: `Brush_Build()` (geometry rebuild after face modification), `Sys_UpdateWindows()` (UI refresh), `Sys_Printf()` (logging)
- **Type definitions** from `radiant/*.h`: `face_t` (map face), `brush_t` (brush entity), `_QERFaceData` (plugin-facing struct), `winding_t`, `brushprimit_texdef_t`

## Design Patterns & Rationale

**Plugin Interface Pattern:** WINAPI-decorated functions exported from DLL allow external tools to discover and call into editor without source linkage. This was the standard approach for extensibility in the 1990s–2000s before plugin frameworks matured.

**Global State Dependency:** Rather than passing editor context through parameters, the code accesses `g_qeglobals` and array collections (`g_ptrSelectedFaces`) directly. This reflects the monolithic editor architecture where all state is globally visible—simplistic but tight coupling.

**Mode Gating:** Both `SetFaceInfo` and `GetFaceInfo` check `g_qeglobals.m_bBrushPrimitMode` and warn if called outside brush primitive mode. This suggests the plugin interface was designed for a specific editor mode; outside that mode, face data structures may differ (e.g., legacy texture def format vs. brush primitive texdef).

## Data Flow Through This File

1. **Read path (Get):**
   - Plugin calls `QERApp_GetFaceInfo()`
   - Retrieves first selected face from `g_ptrSelectedFaces[0]`
   - Copies texture name, plane points (3 verts), brush primitive texdef, and full winding geometry into plugin-allocated buffer
   - Returns success (1) or failure (0)

2. **Write path (Set):**
   - Plugin modifies face data and calls `QERApp_SetFaceInfo()`
   - Updates texture name via `selFace->texdef.SetName()`
   - Copies plane points and brush primitive texdef back into editor face
   - Calls `Brush_Build()` to recompute brush geometry (likely normals, planes, BSP tree)
   - Calls `Sys_UpdateWindows()` to refresh viewport

3. **GL context pass-through:**
   - `QERApp_GetQeglobalsHGLRC()` returns the editor's OpenGL rendering context—allows plugins to render directly into the viewport

## Learning Notes

**Idiomatic Q3Radiant Pattern:** This exemplifies how late-1990s level editors extended functionality: a core application with a global state table, DLL-based plugins that hook via exported functions, and direct global access. Modern engines use dependency injection, event busses, or scripting VMs instead.

**Brush Primitives (Q3A-Specific):** The repeated checks for `m_bBrushPrimitMode` and the dual handling of `brushprimit_texdef` vs. legacy `texdef` reflects Q3A's shift to *brush primitives*—a rotation-invariant texture coordinate system that survives brush transformations better than legacy UV offsets. This file only works in that mode; it's a historical artifact of the editor's modernization mid-project.

**Memory Unsafe Patterns:**
- `strcpy(pFaceData->m_TextureName, selFace->texdef.name)` — classic buffer overflow risk if texture name exceeds field size
- Direct `memcpy` of variable-length `winding_t` using offset-of-array trick: `size = (int)((winding_t *)0)->points[...];` — assumes fixed struct layout and exact knowledge of winding size
- Unchecked pointer casts: `reinterpret_cast<face_t*>(g_ptrSelectedFaces.GetAt(0))` assumes the collection is type-erased but known to contain `face_t*`

## Potential Issues

- **No bounds checking on texture name:** `strcpy` can overflow if texture name is long
- **Fragile winding copy:** Assumes `winding_t` struct layout is stable and that `face_winding->numpoints` is valid; no size validation before `memcpy`
- **Silent failure on empty selection:** Both Set and Get return 0 without warning if no faces selected; callers must check return values
- **Mode-specific contract:** Code will silently fail (or warn and return 0) if called outside brush primitive mode; plugins that don't handle this will appear broken
- **No undo/redo integration visible:** `Brush_Build` and `Sys_UpdateWindows` are called directly; unclear if this participates in the editor's undo system
