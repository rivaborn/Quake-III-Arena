# q3radiant/TextureBar.h — Enhanced Analysis

## Architectural Role

`TextureBar.h` defines the UI dialog bar for interactive texture surface property manipulation in the Q3Radiant level editor (a tool-chain component, not runtime engine code). It bridges the editor's brush selection mechanism with texture coordinate transformations (shift, scale, rotation) that ultimately compile into BSP surface properties and affect runtime rendering. This is part of the editor's **content authoring pipeline**, where user interactions are preserved as `surfaceTerrainInfo_t` or equivalent shader/surface metadata in compiled `.bsp` files.

## Key Cross-References

### Incoming (who depends on this)
- **q3radiant/MainFrm.cpp/h**: Creates and manages the `CTextureBar` instance as a dockable dialog bar in the editor's main frame window
- **Editor geometry selection**: Whenever a brush face is selected in the viewport, `GetSurfaceAttributes()` is called to populate the spinners with the current face's texture properties
- **MFC message routing**: All spin button delta notifications and apply buttons route through the standard MFC message map dispatch

### Outgoing (what this file depends on)
- **MFC framework**: `CDialogBar`, `CSpinButtonCtrl`, `CDataExchange` (Windows-only dependency)
- **q3radiant/Map.cpp** or equivalent: `GetSurfaceAttributes()` likely queries the currently selected brush face; `SetSurfaceAttributes()` writes transformed values back to the face's surface definition
- **q3radiant/resource.h**: Dialog resource IDs (`IDD_TEXTUREBAR`, spin control IDs)
- **q3radiant/Undo.cpp** (implicit): Surface property changes should trigger undo/redo snapshot capture (not visible in header)

## Design Patterns & Rationale

1. **MFC DialogBar Pattern**: Inherits from `CDialogBar` rather than `CDialog` to enable docking within the main frame, reflecting the editor's modular palette-based UI design (common in 1990s–2000s Windows applications).

2. **Data Exchange (DDX)**: The `DoDataExchange()` override synchronizes UI controls ↔ member variables; spin button values automatically marshal to `m_nHShift`, `m_nVShift`, etc.

3. **Spin Button Event Pattern**: `OnDeltaposSpinXxx()` handlers respond to incremental user adjustments (arrow buttons), allowing smooth parameter tweaking without typing.

4. **Attribute Get/Set Abstraction**: `GetSurfaceAttributes()` / `SetSurfaceAttributes()` decouple the UI widget state from the underlying brush/surface model, enabling the dialog to work with different selection types (faces, brushes, patches) via a common interface.

**Rationale**: This design reflects early 2000s Windows editor conventions. The indirection through Get/Set methods allows the editor to maintain a selection manager elsewhere in the codebase without this dialog needing to know the full entity/brush model.

## Data Flow Through This File

```
User Input (spin button click)
  ↓
OnDeltaposSpinHshift() / OnDeltaposSpinVshift() / etc.
  ↓
Member variables updated (m_nHShift, m_nVScale, m_nRotate, ...)
  ↓
OnBtnApplytexturestuff() / OnSelectionPrint()
  ↓
SetSurfaceAttributes() — applies to selected brush face(s)
  ↓
[Runtime: values compile into BSP file as surface texcoord offsets]

Reverse flow (user selects different face):
Selected face in viewport
  ↓
GetSurfaceAttributes() — populates spinners
  ↓
DoDataExchange() marshals to UI controls
  ↓
User sees current texture properties
```

## Learning Notes

- **Editor vs. Engine**: This file is **editor toolchain only**—the compiled texture transformations flow into `.bsp` files but this UI never runs at game time. It illustrates the clear separation between offline content creation (q3radiant/) and runtime execution (code/).
  
- **MFC Era Pattern**: The heavy reliance on MFC message maps (`DECLARE_MESSAGE_MAP`, `afx_msg`) and DDX reflects late 1990s Windows development idioms. Modern editors use event systems with stronger type safety.

- **Texture Coordinate Semantics**: The HShift/VShift/HScale/VScale/Rotate values correspond directly to `texCoord_t` transformations in the shader/surface system. At runtime, the renderer applies these offsets to UV coordinates during rasterization (see `tr_shade.c`, `RB_CalcTexCoords()`), but the *definition* is baked into the BSP at compile time.

- **Integration Point**: The editor's surface property system must track which faces are selected and notify this dialog via `GetSurfaceAttributes()`. This implies a global or frame-wide selection manager (likely in `MainFrm` or a document class).

## Potential Issues

- **No visible undo/redo**: The header shows property setters but no integration with an undo stack. A production editor would auto-snapshot state before each change (likely handled in an owning class, but worth verifying in the `.cpp` implementation).

- **No validation**: Spin button ranges are unspecified in the header; overflow/underflow behavior and clamping rules are hidden in the implementation.

- **Editor-only tooling**: This code is not compiled into the game engine itself—it's part of the standalone q3radiant executable. Any bugs here affect map authoring, not runtime gameplay.
