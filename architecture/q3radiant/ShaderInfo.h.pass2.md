# q3radiant/ShaderInfo.h — Enhanced Analysis

## Architectural Role

`ShaderInfo.h` defines metadata wrapper for shaders within the Q3 Radiant level editor, bridging the editor's UI representation (properties, lists, display names) with the renderer's shader and texture systems. This class enables the editor to manipulate shader selections, track transparency and surface properties, and maintain real-time sync with the renderer's texture asset database (`qtexture_t`). It is an editor-only construct, not part of the runtime engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Editor UI subsystem** (implied by `q3radiant/` organization): shader browser dialogs, texture selection pickers, and surface property panels instantiate and query `CShaderInfo` for display/editing
- **Shader parsing pipeline**: likely populated by `ShaderEdit.cpp` or similar editor shader loader when maps/materials are loaded
- **Material/texture dialogs**: depend on `CShaderInfo` instances to populate shader lists and display shader names/properties

### Outgoing (what this file depends on)
- **Renderer types** (`qtexture_t`): holds a pointer to renderer texture metadata, creating implicit coupling between editor and renderer's internal texture representation
- **Windows/MFC infrastructure**: `CString`, `CStringList` are MFC collection types; `#pragma once` and MSVC version guard indicate Windows-only
- No explicit engine subsystem calls visible; this is a pure data-holder with minimal dependencies

## Design Patterns & Rationale

**Data Holder / Value Object Pattern**: `CShaderInfo` is a simple aggregate of shader metadata with no complex business logic. The public fields (`m_strName`, `m_strShaderName`, etc.) and trivial accessor (`setName`) suggest this is intentionally transparent, allowing UI code to read/write shader properties directly.

**Direct Renderer Coupling**: The `qtexture_t *m_pQTexture` pointer couples the editor's shader metadata directly to the renderer's internal texture structure. This is typical of 1990s–2000s era editors, where the tool and engine shared symbol tables rather than being loosely coupled.

**Parse-on-Construction**: The `Parse(const char *pName)` method suggests a lazy/deferred initialization pattern: shader metadata is loaded on-demand when first accessed, not eagerly during level load.

## Data Flow Through This File

1. **Ingress**: Editor calls `Parse()` with a shader definition string (likely from `.shader` files or BSP metadata)
2. **Transform**: `CShaderInfo` unpacks shader name, texture reference, surface flags, and transparency value
3. **Storage**: Instances are collected into editor containers (implied by use in lists/dialogs)
4. **Egress**: UI code reads fields (`m_strName`, `m_fTransValue`, `m_nFlags`) for display and validation; `m_pQTexture` enables quick texture preview without re-querying the renderer

## Learning Notes

**Editor-Engine Coupling of the Era**: Unlike modern game engines (which use JSON/metadata formats), Q3 Radiant directly references engine types (`qtexture_t`). This reflects a tightly-coupled tool-and-engine workflow common in the early 2000s.

**MFC Dependency**: The use of `CString` and `CStringList` (not standard C++ `std::string`/`std::vector`) indicates this code predates or explicitly targets legacy Windows toolchains. The MFC choice also explains the Windows-only `#pragma once` and MSVC version checks.

**Minimal Responsibility**: This class deliberately does not own shader compilation, texture loading, or I/O—those responsibilities live in other editor modules. `CShaderInfo` is purely a **transfer object** between the UI and renderer data.

## Potential Issues

- **Direct Pointer to Renderer Internals**: `qtexture_t *m_pQTexture` could dangling-point if the renderer unloads or moves textures in memory without notifying the editor. No evidence of lifecycle synchronization.
- **String Copying Overhead**: Storing four `CString` members per shader may be inefficient for large material libraries; modern practice would intern or reference-count.
- **No Const Correctness**: Public fields allow unchecked mutation; `setName()` exists but `m_strName` is also public, inviting inconsistency.
