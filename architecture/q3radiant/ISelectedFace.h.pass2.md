# q3radiant/ISelectedFace.h — Enhanced Analysis

## Architectural Role

This header defines a COM-style plugin interface (`_QERSelectedFaceTable`) that bridges the Q3Radiant editor core to external plugins (especially texture/surface manipulation tools). It exposes read-write access to the currently selected face's texture properties, geometry, and lighting configuration. The GUID-based discovery mechanism (`QERSelectedFaceTable_GUID`) allows plugins to negotiate this interface at runtime without hard dependencies on the editor's internal structures.

## Key Cross-References

### Incoming (who depends on this file)
- **TexTool plugin** and other surface-manipulation plugins query the editor for this interface via GUID and invoke its function pointers to read/write selected face state
- Plugin entry points (`PlugIn_GetData`) likely use the GUID to discover this vtable from the editor's plugin system
- Code in `q3radiant/SelectedFace.cpp` implements the actual state storage and likely populates this vtable

### Outgoing (what this file depends on)
- Indirect dependency on **qtexture_t** type (defined in renderer/tr_local.h or q3radiant's internal texture header) — used by `PFN_TEXTUREFORNAME` return type
- Indirect dependency on **_QERFaceData** struct (likely in q3radiant headers) — face geometry, texinfo, shader references
- Indirect dependency on **winding_t** (geometry library, likely in common/polylib.h) — polygon vertex list for brush face boundaries
- Indirect dependency on **texdef_t** and **brushprimit_texdef_t** (shader system types) — texture coordinate mapping

## Design Patterns & Rationale

**COM-style GUID interface**: The static GUID enables plugins to discover this interface by type at runtime. This decouples the plugin from the editor's internal object model—plugins don't need to know the implementation, only the stable GUID.

**Virtual function table (vtable)**: Function pointers allow the editor to swap implementations without plugin recompilation. The `m_nSize` field (first member) is a versioning strategy: newer editor versions can extend the struct while old plugins still work by checking size.

**Asymmetric read/write**: `GetFaceInfo`/`SetFaceInfo` bracket mutations. This allows the editor to validate or log changes; `SetTexture` is a separate path for batch texture operations (likely more efficient than individual face updates).

**Type erasure**: The `void* pPlugTexdef` parameter in `Select_SetTexture` avoids exposing internal shader data structures to plugins—plugins pass opaque handles back to the editor.

## Data Flow Through This File

1. **Plugin initialization**: Plugin queries editor for `QERSelectedFaceTable_GUID` → obtains vtable pointer
2. **Read path**: Plugin calls `GetFaceInfo()` → retrieves current face's geometry, texinfo, shader name → plugin modifies locally
3. **Write path**: Plugin calls `SetFaceInfo()` → editor validates, updates brush data, marks document dirty
4. **Texture binding**: Plugin calls `TextureForName()` → editor loads/caches qtexture_t asset → plugin uses for preview
5. **Batch update**: Plugin calls `Select_SetTexture()` with new texdef → editor applies to all selected faces atomically

## Learning Notes

**Early plugin architecture**: This reflects pre-2000s editor plugin design (GUIDs + vtables without modern C++ abstractions). Modern editors use introspection, trait systems, or message dispatch instead.

**No per-face locking**: The interface assumes single-threaded editor access. No mention of undo/redo bookending—plugins are expected to call `SetFaceInfo` synchronously while the editor is idle.

**Incomplete encapsulation**: The TODO comment acknowledges that mixing "selected face" operations with generic texture operations in one interface is a code smell. Texture queries (`GetTextureSize`, `TextureForName`) would better belong in a dedicated texture query interface.

**Winding assumption**: The comment `winding_t is assumed to have MAX_POINTS_ON_WINDING allocated` delegates buffer management to the caller. Modern APIs would return owned heap data or use iterators.

## Potential Issues

- **No error handling**: Function pointers return `int` or `void`; no indication of success/failure. Callers have no way to know if `TextureForName` loaded a fallback texture.
- **Type safety**: `void* pPlugTexdef` parameter in `Select_SetTexture` breaks type checking. A malformed pointer causes undefined behavior.
- **Stale pointers**: If a plugin caches a `qtexture_t*` from `TextureForName` and the editor reloads textures, the pointer becomes invalid—no invalidation callback.
- **Versioning mismatch**: The `m_nSize` field only helps if plugins check it; malformed struct layouts (ABI incompatibility) could silently corrupt memory if overlooked.
