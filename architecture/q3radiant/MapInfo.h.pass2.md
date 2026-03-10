# q3radiant/MapInfo.h — Enhanced Analysis

## Architectural Role

`MapInfo.h` defines an MFC dialog class (`CMapInfo`) that displays map statistics and entity information within Q3Radiant, the level editor tool. As part of the **editor's information/inspection subsystem**, this dialog bridges the map model (holding brush/entity counts) to the user interface, allowing mappers to inspect metadata about the current map without leaving the editor. This is strictly a **tool-side component** — it has no involvement in the runtime engine, bot AI, or game logic; it exists solely to improve the editor's UX.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main UI framework** (inferred from MFC dialog structure): menu commands or toolbar buttons likely instantiate `CMapInfo` to display the dialog
- **Map document/data model**: populated with counts (`m_nTotalBrushes`, `m_nTotalEntities`) extracted from the loaded BSP/map structure
- No visible references in the provided cross-reference table (which focuses on runtime engine and botlib); typical for MFC dialogs instantiated through UI events rather than function calls

### Outgoing (what this file depends on)
- **MFC framework** (`CDialog`, `CListBox`): Windows-only UI infrastructure
- **Map data model** (implicit): reads aggregate statistics during `OnInitDialog()` to populate UI fields
- **Windows registry/config** (via DDX): dialog state persistence if configured

## Design Patterns & Rationale

**MFC Dialog Exchange Pattern**: Uses `DoDataExchange(CDataExchange* pDX)` and `{{AFX_DATA}}` markers for automatic bidirectional data binding. This decouples the dialog UI controls from business logic — a standard MFC idiom from the 1990s–2000s era.

**Data Container with UI Binding**: Rather than complex logic, `CMapInfo` is purely a **data presentation vessel**. The member variables (`m_nTotalBrushes`, `m_nTotalEntities`, `m_lstEntity`) are synchronized to/from UI controls via DDX, requiring minimal hand-written glue.

**Why this structure?** MFC's code-generation tools (ClassWizard) produce this boilerplate to isolate UI concerns. Modern engines use data-driven UI (JSON/YAML configs), but Q3Radiant predates that paradigm and relies on Visual C++'s wizard-generated scaffolding.

## Data Flow Through This File

1. **Initialization**: `OnInitDialog()` fires when the dialog is created; this is where map statistics would be populated into `m_nTotalBrushes` and `m_nTotalEntities` by querying the map document.
2. **UI Binding**: `DoDataExchange()` synchronizes the C++ member variables ↔ dialog controls (list box `IDC_LIST_ENTITY`, possibly spinners or text fields for the counts).
3. **Display**: The dialog presents the entity list and aggregate statistics to the mapper.
4. **Destruction**: Dialog closes, MFC handles cleanup via `CDialog` destructor.

## Learning Notes

- **Snapshot of pre-modern UI architecture**: Q3Radiant's UI is entirely **MFC-based** (Windows-only), reflecting Visual C++ 6.0–era practices. Modern game editors (Unreal, Unity) use platform-agnostic UI frameworks (Qt, Dear ImGui, custom GPU-driven UIs).
- **Dialog-as-info-panel pattern**: Simple read-only dialogs for map statistics are a lightweight way to surface metadata without adding permanent UI panels — useful for occasional queries.
- **Editor vs. engine boundary**: This file is a clean example of the sharp separation between the **offline tool pipeline** (`q3radiant/`, `q3map/`, `bspc/`) and the **runtime engine** (`code/`). The architecture context makes this explicit: tools are in separate trees, never linked with the shipped game engine.

## Potential Issues

- **No visible initialization logic**: `OnInitDialog()` is declared but the implementation (in `.cpp`) would need to actually populate the counts. If the map document isn't passed to the dialog or queried correctly, the statistics could remain stale or uninitialized.
- **Limited entity list binding**: The `m_lstEntity` CListBox has no visible population logic in the header; the `.cpp` would need to iterate the map's entity list and insert entries. If entities are frequently modified during editing, the dialog would need refresh logic.
- **Platform-specific (Windows/MFC only)**: Hardwires the tool to Visual C++ + Windows; porting to macOS or Linux would require replacing MFC with cross-platform UI.
