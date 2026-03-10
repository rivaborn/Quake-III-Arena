# q3radiant/EditWnd.h — Enhanced Analysis

## Architectural Role

`CEditWnd` is a lightweight MFC control wrapper in the Q3Radiant level editor GUI. It provides a customizable text-input control used in property dialogs and entity/shader editors throughout the radiant UI. As a tool component (not runtime engine), it has no direct interaction with the game engine—it only participates in map authoring workflows where users edit textual properties before compilation to BSP/AAS.

## Key Cross-References

### Incoming (who depends on this file)
- Window/dialog classes in `q3radiant/` that embed or instantiate `CEditWnd` for text field UI (likely property panels, entity editors, shader editors)
- MFC framework's message dispatch and dynamic object creation system via `DECLARE_DYNCREATE`

### Outgoing (what this file depends on)
- Windows/MFC framework (`CEdit` base class from `afxwin.h`, implied)
- No engine subsystem dependencies—this is purely a UI utility with no game-side side effects

## Design Patterns & Rationale

**MFC Wrapper Pattern**: Extends `CEdit` to provide a reusable control with custom initialization logic. The `DECLARE_DYNCREATE` macro enables MFC's dialog template system to instantiate the class dynamically at runtime rather than embedding it statically.

**Message Map Architecture**: The placeholder `//{{AFX_MSG(CEditWnd)}}` region follows MFC's code-generation convention where Visual C++ ClassWizard would inject message handlers (e.g., `WM_CHAR`, `WM_KEYDOWN`) automatically. The absence of handlers here suggests this control is intentionally minimal—only `PreCreateWindow` is overridden to customize window creation parameters before the OS window is created.

**Rationale for Structure**: Early 2000s Windows development (when Q3 shipped) relied on MFC for rapid UI prototyping. This wrapper allows derived dialogs or the editor core to standardize text-input behavior (e.g., custom fonts, readonly mode, input validation) across multiple dialogs without code duplication.

## Data Flow Through This File

```
[Dialog Resource Template] 
  → MFC creates CEditWnd instance (DECLARE_DYNCREATE)
  → PreCreateWindow called with CREATESTRUCT
    (custom window style/class/font can be applied here)
  → Windows window handle created
  → [User typing] → Message Map dispatch
  → [Dialog gets text via GetWindowText]
  → [Entity/property data stored to map data structure]
```

## Learning Notes

**Era-Specific Design**: This code exemplifies late 1990s/early 2000s Windows game tool architecture. Modern level editors (Unreal Editor 5, Unity Editor) use C# or scripting with responsive property panels; Q3Radiant used heavyweight C++ MFC with Visual C++ 6.0-era patterns.

**Minimal Customization**: Unlike modern controls, `CEditWnd` doesn't expose observable public state or validation. It relies entirely on `PreCreateWindow` for customization—users would override this method if they needed to set fonts, styles, or max text length before the window exists. This is idiomatic to MFC's pre-creation pattern (contrast with modern declarative UI frameworks).

**No Vector/BSP Coupling**: This editor component is intentionally decoupled from the BSP/AAS compilation logic (`code/bspc/`, `code/botlib/`). Textual property editing happens in the editor tool; the actual map geometry and navigation mesh are computed offline by `q3map` and `bspc` tools.

## Potential Issues

- **Platform Lock-in**: MFC is Windows-only; Q3Radiant was not ported to Linux/macOS without substantial effort (Unix/Mac support required complete UI rewrites).
- **Vague Customization Hook**: The empty `PreCreateWindow` override provides no guidance to maintainers on *what* properties are typically modified. A subclass would need to peek at MFC docs or reverse-engineer usage patterns in dialogs that use this control.
- **Message Map Fragility**: MFC's code-generation model is brittle—ClassWizard-generated message maps can break if manually edited incorrectly, though this particular class shows defensive empty placeholders to reduce risk.
