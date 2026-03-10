# q3radiant/SurfaceDlg.cpp — Enhanced Analysis

## Architectural Role

`SurfaceDlg.cpp` implements the **Surface Inspector** dialog in the Q3Radiant level editor—a UI widget for inspecting and modifying texture properties on brush faces and patches. Although syntactically part of `q3radiant/`, this file exists *entirely outside the runtime engine*: it bridges the editor's internal face representation to the texture definitions and shader constants (`texdef_t`, surface flags, contents bits) that are consumed by the **Renderer** and **Game VM** subsystems at runtime. Its role is to make those abstract concepts (texture coordinate matrix, contents/flags bitfields) editable in a human-readable form.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant/MainFrm.h**: Creates and manages the `g_dlgSurface` global instance
- **q3radiant editor selection/undo system**: Calls `SetTexMods()` when selection changes or `GetTexMods()`/`GrabPatchMods()` when user applies changes
- **Message handlers** (WM_HSCROLL, ON_BN_CLICKED, etc.): Respond to Windows UI events

### Outgoing (what this file depends on)
- **q3radiant/TextureLayout.h**: `ConvertTexMatWithQTexture()`, `TexMatToFakeTexCoords()`, `FakeTexCoordsToTexMat()` — coordinate system transformation functions
- **texdef_t / brushprimit_texdef_t** types: Defined in editor's internal BSP representation (not cross-checked with runtime headers)
- **Surface flags & contents enums**: `g_checkboxes[64]` array maps to 64 bits (32 flags + 32 contents bits) used by Game VM's `entityState_t` and collision system
- **Patch API**: `Patch_GetTextureName()`, `Patch_SetTextureInfo()`, `OnlyPatchesSelected()`
- **Global texture state**: `g_qeglobals.d_texturewin.texdef`, `g_qeglobals.m_bBrushPrimitMode`

## Design Patterns & Rationale

**Dual-Mode Operation**: The dialog switches between two distinct data models:
- **Brush primitive mode** (`g_qeglobals.m_bBrushPrimitMode`): Uses a 3×2 texture matrix (`brushprimit_texdef_t.coords`) for precise texture coordinate control. `GetTexMods()` / `SetTexMods()` convert between user-facing shift/scale/rotate triplets and the underlying matrix via `FakeTexCoordsToTexMat()` / `TexMatToFakeTexCoords()`. This is a *coordinate system abstraction*—the editor UI presents simplified spinners, but internally transforms to the matrix representation consumed by the renderer.
- **Legacy mode**: Direct `texdef_t` shift/scale/rotate fields stored directly on the face.

**Why this dual-mode exists**: Brush primitives are an *advanced feature* allowing texture artists to specify exact UV coordinates; the matrix form is more mathematically rigorous than shift/scale/rotate. The UI hides this complexity by always presenting shift/scale/rotate, computing the matrix as needed.

**Bitfield Aggregation**: The 64 checkboxes (`IDC_CHECK1` through `IDC_CHECK64`) map two 32-bit fields: flags (rendering/physics) and contents (volume type). These bitfields originate in the **Game VM** and **Collision system** (`code/qcommon/cm_public.h`, `code/game/g_public.h`) and are serialized into BSP map files. The editor enforces these symbolic definitions *without linking to the runtime engine*.

## Data Flow Through This File

1. **SetTexMods() → Display**: 
   - Input: `g_ptrSelectedFaces` (editor's selected face pointers) or `OnlyPatchesSelected()`
   - If brush primitive mode: load `selFace→brushprimit_texdef`, transform to fake shift/scale/rotate via `TexMatToFakeTexCoords()`
   - Output: Populate dialog text fields and checkboxes; `InvalidateRect()` to redraw

2. **GetTexMods()/GrabPatchMods() → Apply**:
   - Input: Dialog text fields and checkbox states
   - If patch mode: `GrabPatchMods()` reads patch-specific floating-point texture parameters, stores in `g_patch_texdef`
   - If brush primitive mode: `GetTexMods()` reads shift/scale/rotate, transforms to matrix via `FakeTexCoordsToTexMat()`, stores in `brushprimit_texdef`
   - Call `Select_SetTexture(pt, &local_bp)` to propagate changes to selected faces; `Patch_SetTextureInfo(&td)` for patches
   - Output: Modified texdef values written back to face/patch objects

3. **UpdateSpinners()**: Incremental spinner adjustments (up/down arrows) modify a single field and re-apply via `GetTexMods()`

## Learning Notes

**Historical context**: This UI was designed to support Q3A's shift/scale/rotate parameterization, then extended with brush primitives (UT2003-era feature). The coordinate transformation functions (`TexMatToFakeTexCoords`, etc.) show how to convert between two representations of the same concept.

**Editor/Engine boundary**: Unlike the runtime (where `texdef_t` flows directly into the renderer), the editor maintains a *separate* BSP representation with its own types. The assumption is that if you modify texture parameters in the editor, you'll recompile the BSP, and the resulting map file will contain valid `texdef_t` structs. **No validation** that the editor's `texdef_t` layout matches the runtime's; this is a historical fragility (common in tools written as afterthoughts).

**Flag/contents semantics**: The 64 checkboxes are a *generic bitfield UI*. The symbolic meaning (e.g., bit 5 = `CONTENTS_LAVA`) is not stored here; it comes from the **q3_ui** or **cgame** VM's header files (`code/cgame/tr_types.h`, `ui/menudef.h`). The editor simply treats them as opaque flags and passes them through.

**Patch vs. brush asymmetry**: Patches (curved surfaces) store floating-point texture parameters (`pt→shift[0]`, `pt→scale[0]`) with 6 decimal places, while brushes (flat faces) use integers. The `m_bPatchMode` flag selects the display format. This is a **type mismatch in the editor's data model**: `texdef_t` is union-like, reusing the same fields for both. At compile/runtime, the Game VM only uses brush `texdef_t` (patches aren't networked); the patch values are editor-only metadata.

## Potential Issues

- **No validation of texture name lookup**: `GetTexMods()` reads an arbitrary string from IDC_TEXTURE and calls `pt→SetName(sz)`. If the string doesn't match a loaded shader, the map will reference a non-existent texture at runtime.
- **Brush primitive matrix/qtexture mismatch**: `ConvertTexMatWithQTexture()` adapts a texture matrix to a specific qtexture's width/height. If the referenced texture is deleted or renamed, the matrix is silently invalid.
- **Flag/contents bit documentation missing**: The 64 checkboxes have no in-UI labels explaining what each bit means; users must cross-reference external documentation.
