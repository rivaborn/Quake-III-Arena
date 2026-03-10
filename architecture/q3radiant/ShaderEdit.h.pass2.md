# q3radiant/ShaderEdit.h — Enhanced Analysis

## Architectural Role

`CShaderEdit` is a Win32/MFC tool window within the Q3Radiant level editor providing an interactive shader script editor with status feedback. It sits outside the runtime engine entirely—it enables map designers to author and preview shader definitions that will later be compiled into the engine's runtime shader system. Unlike the renderer's `tr_shader.c` (which parses `.shader` scripts at load time), this tool is a *design-time* companion to the shader pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- Radiant main frame (`q3radiant/MainFrm.cpp/h`) — instantiates `CShaderEdit` as a dockable/floating child window when the user opens the shader editor
- Radiant plugin/messaging system — may route shader-related commands to this window via `IMessaging`

### Outgoing (what this file depends on)
- **MFC framework only**: `CFrameWnd`, `CStatusBar`, `CToolBar`, message map macros
- **Not the engine**: No runtime renderer (`tr_shader.c`), collision, or game VM dependencies — this is a standalone authoring tool
- **External data**: Shader files from the virtual filesystem (loaded by cpp implementation, not shown)

## Design Patterns & Rationale

**MFC SDI (Single Document Interface) pattern**: The `DECLARE_DYNCREATE` macro + protected constructor + virtual `~CShaderEdit()` follow MFC's dynamic window creation protocol. This allows Radiant to create/destroy shader editor windows at runtime without explicit `new`/`delete` calls.

**Separation of concerns**: The `.h` declares only the UI shell (`CStatusBar`, `CToolBar`, message routing). The actual shader parsing, syntax highlighting, and file I/O logic would live in `.cpp`, isolated from core editor logic.

**Why this structure**: Editor tools in the early 2000s (especially MFC-based) favored this split to keep UI state separate from content logic. The frame window is purely a container; the real work happens in child view windows not visible in this header.

## Data Flow Through This File

1. **Creation**: Radiant main window calls `CFrameWnd::Create()` (via MFC dynamic creation) → `OnCreate()` initializes toolbar and status bar
2. **Usage**: User types/edits shader text → message handlers forward to hidden view window (not declared here) → validation/preview
3. **Feedback**: Status bar displays compilation warnings, syntax errors, or live preview feedback
4. **Destruction**: Frame closes → `~CShaderEdit()` cleanup (toolbars, resources)

The window acts as a **display container** for underlying shader content, mirroring the engine's two-layer architecture (client UI ↔ core logic).

## Learning Notes

**Tool vs. Runtime Chasm**: This file exemplifies Quake III's separation of *level authoring* from *engine runtime*. Radiant editors (this one, texture browser, entity inspector, etc.) have no dependency on or from the shipping engine code—they're standalone Win32 applications. The engine's `tr_shader.c` only reads the *output* (`.shader` files), not Radiant's internal state.

**Legacy MFC patterns**: `AFX_VIRTUAL` comments and `DECLARE_MESSAGE_MAP()` are hallmarks of pre-C++11 MFC. Modern engines would use cross-platform frameworks (Qt, Godot editor, Unreal editor). This code reflects early 2000s Windows-only tooling.

**No data binding**: The header shows no cvar/state sync mechanisms. User edits are likely file-driven ("save as `.shader`") rather than real-time bidirectional binding with the engine.

## Potential Issues

- **MFC-only**: Limits portability; modern forks/ports would need Qt or similar
- **Implicit view**: The actual editor content (code/text area) is not visible in this header—likely a child `CView` or custom control initialized in `.cpp`, making the architecture opaque at a glance
