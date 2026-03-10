# q3radiant/isurfaceplugin.h — Enhanced Analysis

## Architectural Role

This header defines a **bidirectional plugin contract** between Q3Radiant's core editor and third-party surface property plugins. It enables plugins to intercept and customize texture definition workflows across three data types: brush faces (`face_t`), texture slots (`qtexture_t`), and Bézier patches (`patchMesh_t`). The two GUID-identified function tables establish a two-way communication boundary: `QERPlugSurfaceTable` exposes editor services to plugins; `QERAppSurfaceTable` allows plugins to query editor state and trigger updates. This is a **tool-specific interface** entirely outside the runtime engine — it has no dependency path into `code/`.

## Key Cross-References

### Incoming (who depends on this file)
- Q3Radiant plugin loader/manager (not in provided context, but implied in `q3radiant/PlugInManager.cpp`)
- Any surface property plugin DLL implementing `QERPlugSurfaceTable_GUID` and registering via `GetProcAddress`
- Core Radiant code that manages surface dialogs, texture application, and map serialization (`SurfacePlugin.cpp`, `Undo.h`, map I/O)
- Radiant's `pData` void-pointers in `face_t`, `qtexture_t`, `patchMesh_t` are cast to `IPluginTexdef*` via `GETPLUGINTEXDEF` macro

### Outgoing (what this file depends on)
- **Radiant core types** (not in this header but referenced): `face_t`, `qtexture_t`, `patchMesh_t`, `texturewin_t`
- **Radiant's printf infrastructure**: `PFN_QERAPP_MAPPRINTF` for serializing custom texdef data to `.map` files
- **Radiant global state** accessible via `QERAppSurfaceTable` callbacks (patch selection, texture window, etc.)

## Design Patterns & Rationale

1. **COM-Style Versioning (IUnknown-lite)**
   - GUID-based interface discovery avoids symbol binding
   - Struct's `m_nSize` field allows safe extension (new function pointers appended without breaking old plugins)
   - Reference counting (`IncRef`/`DecRef`) decouples plugin lifetime from Radiant's memory management

2. **Strategy Pattern (Type Polymorphism)**
   - Single `IPluginTexdef` interface with separate methods for `face_t`, `qtexture_t`, `patchMesh_t`
   - Rationale: editor UI is unaware which data type the plugin is bound to; plugin must handle all three
   - Contrast with modern plugin systems that might use inheritance or trait objects

3. **Factory + Allocation Contract**
   - Three allocators (`PFN_QERPLUG_TEXDEFALLOC`, `_QTEXTUREALLOC`, `_PATCHALLOC`) with caller ownership
   - Radiant initializes each `IPluginTexdef` with a back-pointer to the data it's bound to
   - Tradeoff: plugins must manage object lifecycle vs. simplicity of direct allocation

4. **Adapter/Wrapper (pData Pattern)**
   - `void*` slots in Radiant's core types hold plugin-opaque state
   - `GETPLUGINTEXDEF` macro casts at point of use — minimal coupling
   - Rationale: plugins extend Radiant's data model without modifying core source

## Data Flow Through This File

**Plugin Initialization:**
```
Radiant loads plugin DLL 
→ queries QERPlugSurfaceTable_GUID 
→ caches function pointers (DoSurface, ByeByeSurfaceDialog, etc.)
```

**Surface Edit Workflow:**
```
User selects face/texture/patch in editor
→ Radiant calls PFN_QERPLUG_TEXDEFALLOC( entity_ptr )
→ Plugin allocates IPluginTexdef, stores back-pointer to entity
→ Plugin's ParseTexdef() reads custom key-values from .map or UI
→ User modifies surface properties via custom dialog (m_pfnDoSurface)
→ m_pfnUpdateSurfaceDialog refreshes UI after external changes
```

**Map Serialization:**
```
Radiant saves .map file
→ calls plugin's WriteTexdef( MapPrintf )
→ plugin writes custom texdef fields via supplied callback
→ plugin controls exact formatting/syntax
```

**Patch-Specific Flow:**
```
User edits patch mesh
→ separate ParsePatchTexdef / WritePatchTexdef paths
→ allows different storage format than brush/texture texdefs
```

## Learning Notes

- **Tool vs. Engine Split**: Q3's architecture cleanly separates the offline level editor (with plugin extensibility) from the runtime engine (monolithic, no plugin contract). This is a late-1990s design choice; modern engines blur this boundary.
  
- **Outdated Comment** (`miptex_t` marked as WAL-format-only) suggests this codebase was retrofitted from Quake II tooling; Q3 never used `.WAL` textures at runtime.

- **Reference Counting**: The `IncRef`/`DecRef` pattern is idiomatic for pre-STL C++ plugin systems (circa 1999) but today would use `std::shared_ptr`. Shows age of codebase.

- **Macro-Based Casting** (`GETPLUGINTEXDEF`) is defensive: avoids `static_cast<>()` inline, making the unsafe type coercion explicit and easy to grep. Good practice for plugin boundaries.

- **Bidirectional Interface**  is unusual for modern plugin systems (which are typically unidirectional). Radiant chose this to give plugins full inspection/control over editor state, accepting the coupling cost. Enables plugins like deformers or terrain tools that need to query what's selected.

## Potential Issues

1. **No Version Field**: `m_nSize` alone doesn't encode a version number. If a new function is appended and an old plugin loads, it will read uninitialized memory past `m_nSize`. A safer pattern is `{ int size, int version, ... }`.

2. **WINAPI Hardcoded**: All function pointers use `WINAPI` calling convention, limiting to Windows x86. Unix/Linux ports of Radiant would fail to load plugins or require recompilation with platform-specific macros.

3. **No Error Propagation**: Methods like `Hook()`, `ParseTexdef()` return `void`. If binding fails, there's no way to signal an error to Radiant. Modern versions would return `bool` or an error code.

4. **Commented-out `IPluginQTexture`**: Suggests either incomplete implementation or abandoned feature. Unclear why one texture interface was removed and `IPluginTexdef` kept.

5. **Weak Copy Semantics**: `Copy()` returns a new `IPluginTexdef`, but shallow vs. deep copy is unspecified. Plugins must infer behavior.
