# q3radiant/IShaders.h — Enhanced Analysis

## Architectural Role

This file defines a **plugin interface contract** for the q3radiant level editor's shader and texture resource management subsystem. It bridges the editor's plugin layer to the runtime shader/texture loading infrastructure, allowing plugins to query shader definitions and load texture images for map editing and preview. The interface follows a Windows COM-style GUID-based discovery pattern, decoupling the editor's core systems from plugin implementations while providing safe "try-load" semantics that fail gracefully rather than crash on missing assets.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant plugins** (accessed via `QERShadersTable_GUID` discovery): request shader and texture data during editor operations
- **q3radiant UI components** (texture browser, surface dialog, material picker): populate live preview and asset lists using `m_pfnTryTextureForName`
- **IMessaging.h / plugin manager**: passes the populated `_QERShadersTable` vtable to plugin instances at load time

### Outgoing (what this file depends on)
- **Renderer subsystem** (likely `code/renderer/tr_shader.c`, `tr_image.c`): implements the two function pointers; performs actual shader parsing and texture file I/O
- **Shader definition files** (`.shader` scripts in `scripts/` .pk3): source data loaded by `PFN_TRYSHADERFORNAME`
- **Texture image files** (`.tga`, `.jpg` in asset directories): raw image files accessed by `PFN_TRYTEXTUREFORNAME`

## Design Patterns & Rationale

**GUID-Based Virtual Interface Table** — Mimics Windows COM's discovery model. Plugins call a plugin-manager entry point with the GUID constant; the manager returns a populated `_QERShadersTable` struct. This decouples the plugin binary from the editor's internal DLL without requiring static linking or forward declarations.

**"Try" Semantics** — Both function pointers return `NULL` on failure rather than throwing exceptions or logging errors. This allows plugins to gracefully degrade: "shader doesn't exist in editor's current assets → fall back to a default representation." Notably absent: no error callback or logging hook, which is a significant limitation noted in the header's TODO comments.

**Two-Tier Asset Abstraction** — `TryShaderForName` loads the *computed* shader state (potentially synthesizing defaults), while `TryTextureForName` loads only the raw image file. This mirrors the runtime renderer's split between shader definitions and their texture dependencies, but the comments suggest this API was incomplete and the distinction somewhat artificial for editor purposes.

## Data Flow Through This File

1. **Plugin initialization**: Plugin calls plugin manager with `QERShadersTable_GUID` → editor returns vtable containing function pointers
2. **Shader asset query**: Plugin calls `m_pfnTryTextureForName("textures/base_wall/concrete")` → renderer loads from `.pk3` or disk → returns `qtexture_t*` or `NULL`
3. **Shader definition query**: Plugin calls `m_pfnTryShaderForName(...)` (currently disabled/unimplemented per comment) → would have loaded `.shader` script definitions
4. **Editor preview/UI**: UI components iterate through loaded shaders via the vtable; populate texture browser, apply to selected faces

## Learning Notes

**Plugin vs. Runtime Design Divergence** — Unlike the runtime engine (where shader parsing is internal to the renderer DLL), the editor exposes shader loading as a discrete plugin service. This reflects the editor's need to support third-party tool plugins (e.g., custom material editors, terrain tools, shader generators) that must access editor assets independently.

**Incomplete/Evolutionary API** — The extensive TODO/NOTE comments indicate this interface was still in flux (circa Q3A release 2000). The disabled `m_pfnTryLoadShader` and unimplemented `m_pfnTryShaderForName` suggest the developers intended richer shader-definition access but deferred it, possibly due to the shader code being mid-refactor. Modern editors (and later id engines) would unify these into a single asset-load abstraction.

**Absence of Asset Enumeration** — Unlike modern editors (which typically provide `GetAllShaders()`, `EnumerateTextures()`), this API is purely on-demand single-file lookup. Plugins must know the asset name to query it; they cannot browse the full asset library programmatically. This is both a design simplicity win and a limiting factor for generic asset browsers.

**GUID Stability** — The hardcoded `0xd42f798a...` GUID implies binary compatibility contracts. Any change to the vtable layout would require a new GUID; this is intentional, as it prevents old plugins from crashing against new editor DLLs (and vice versa).

## Potential Issues

1. **Missing Error Context** — No error callback, logging, or status reporting. Plugins cannot distinguish "file not found" from "asset corrupt" from "filesystem error." They only get `NULL`.

2. **Pointer Lifetime Ambiguity** — The file does not document memory ownership. Do returned `qtexture_t*` pointers belong to the plugin (requiring manual free) or to the editor's cache (lifetime tied to editor state)? Modern APIs would specify this explicitly (or use reference counting).

3. **No Asset Change Notifications** — If a `.shader` file is edited and reloaded while a plugin is active, the plugin's cached pointers are silently stale. No invalidation mechanism is provided.

4. **Incomplete Vtable** — `m_nSize` is present (suggesting extensibility), but no documented versioning or future-field semantics. If a plugin was compiled for a newer vtable size, it could read uninitialized memory on an older editor DLL.
