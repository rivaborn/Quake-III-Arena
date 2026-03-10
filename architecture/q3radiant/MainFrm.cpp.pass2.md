# q3radiant/MainFrm.cpp — Enhanced Analysis

## Architectural Role

`MainFrm.cpp` implements the main window frame of the Q3Radiant level editor—a Windows MFC application separate from the runtime engine. It serves as the command routing hub and state coordinator for all editor subsystems: it owns the message dispatch loop, manages multiple docked/floating child windows (3D camera, XY orthogonal, Z-elevation, texture browser), and orchestrates editor-wide state through global variables and a centralized keyboard command table. This file bridges the Win32 message pump to the editor's domain logic via an unconventional command-table dispatch mechanism atypical of standard MFC.

## Key Cross-References

### Incoming (who depends on this file)
- **MFC framework** via `IMPLEMENT_DYNAMIC` and message map (`ON_WM_*`, `ON_COMMAND`): Windows sends all frame messages here
- **Child view windows** (`ZWnd`, `CamWnd`, `TexWnd`, `EditWnd`, `entityw.h`) likely call back to `g_pParentWnd` to coordinate redraws, state updates, and clipboard operations
- **Dialog classes** (`PrefsDlg`, `RotateDlg`, `EntityListDlg`, `ScriptDlg`, `NewProjDlg`, `CommandsDlg`, `ScaleDialog`, `FindTextureDlg`, `SurfaceDlg`, `PatchDensityDlg`, `DialogThick`, `PatchDialog`, `NameDlg`, `dlgcamera.h`) instantiated/launched here
- **Global state readers** throughout the codebase access `g_strAppPath`, `g_pParentWnd`, `g_Preferences`/`g_PrefsDlg`, `g_nUpdateBits`, `g_bScreenUpdates`, `g_strProject`

### Outgoing (what this file depends on)
- **qe3 system** (`qe3.h`): the core editor model/domain logic layer; implied by the `#include` and use of global state
- **All view window classes** via direct instantiation and message forwarding (ZWnd, CamWnd, TexWnd, EditWnd)
- **Preferences system** (`CPrefsDlg g_Preferences`): global prefs instance
- **botlib and aas subsystems** indirectly through menu items (e.g., entity color dialogs, leak spots)
- **Platform layer** (implicit via MFC/Windows): all GDI, window creation, message dispatch

## Design Patterns & Rationale

**Command Dispatch Table (unusual for MFC):**  
The `SCommandInfo g_Commands[]` array (~150 entries) maps human-readable command names (e.g., `"ToggleOutlineDraw"`) to Windows `VK_*` keycodes, modifier bits, and `ID_*` resource constants. This is a **data-driven command table** pattern that decouples keyboard input from menu items—atypical of MFC's built-in `ON_COMMAND` mechanism. The rationale: Radiant's command set is large and complex; a table allows runtime rebinding (evidenced by `CommandsDlg` in the includes) and facilitates serialization of custom keybindings. The modifier bit scheme encodes shift/alt/ctrl/press-only in low bits (bits 0–4).

**Global State Management:**  
`g_pParentWnd`, `g_Preferences`, `g_nUpdateBits`, `g_bScreenUpdates`, `g_strProject` are all globals rather than instance variables. This reflects the single-document model and the C-codebase legacy of the original Radiant: a single global editor state shared across all subsystems rather than encapsulated OOP design. Comments in the code acknowledge this ("both of the above should be made members of CMainFrame").

**Deferred Screen Updates:**  
`g_bScreenUpdates` is a boolean gate: when false, child windows suppress repaints (implied by "used in a few places to disable updates for speed reasons"). This is a manual **dirty-rect/deferred refresh** pattern predating modern retained-mode UIs—useful for batch operations (e.g., rotating selected brushes) where repainting on every small change would be slow.

**Modifier Bit Encoding in Commands:**  
Each command stores a `nModifiers` field with bits:
- Bit 0: Shift
- Bit 1: Alt
- Bit 2: Control
- Bit 4: Press-only (vs. release)

This compact encoding allows one command name to bind to multiple key+modifier combinations (e.g., `"Patch TAB"` appears twice with modifiers `0x00` and `0x01` for Tab alone and Shift+Tab).

## Data Flow Through This File

1. **Input Entry:** Windows keyboard/mouse messages → `OnKeyDown`, `WM_PARENTNOTIFY`, child window mouse handlers
2. **Command Dispatch:** Message map routes to `ON_COMMAND` handlers (e.g., `OnFileOpen`, `OnSelectionClone`) or lookup in `g_Commands` table by key+modifier
3. **View Coordination:** Command handlers invoke methods on child window pointers (implicit via global state or direct member pointers in CMainFrame)
4. **State Broadcast:** `g_nUpdateBits` (a bitmask) flags which subsystems need redraws; child windows poll this or subscribe to update notifications
5. **Preferences Synchronization:** `g_Preferences` (global CPrefsDlg instance) persists editor settings; loaded/saved on startup/shutdown
6. **Output:** Child windows redraw based on state changes; multiple `Invalidate()` calls trigger WM_PAINT and screen updates

## Learning Notes

- **MFC-era UI architecture**: This is a 1990s-2000s Windows-native MFC application, not a cross-platform or modern web/game UI toolkit. Students studying legacy game tools or Windows native programming will find a textbook example of MFC message maps, dialog boxes, and MDI/SDI frame management.
- **Command tables vs. menu-driven UI**: Rather than relying on menu items to trigger commands, Radiant uses a sparse **command name → keybinding** table. This is closer to how game engines (id/Quake) manage commands (see `cmd.c` in qcommon) than traditional GUI apps. It reflects the tool's origin as a C program adapted to Windows.
- **Global state pervasiveness**: The extensive use of globals (`g_pParentWnd`, `g_Preferences`) is a red flag by modern OOP standards but was pragmatic for rapid porting of C codebases. The comments in the file acknowledge this.
- **Deferred updates and frame-rate independence**: The `g_bScreenUpdates` flag is an early form of **framerate-independent rendering control**, allowing the editor to pause visual updates during heavy operations (BSP compilation, lightmap generation).
- **Shader editor and BSP tool integration**: The visible includes and command handlers reference BSP compilation (leak spots, entity lists), suggesting tight coupling between the editor UI and offline compilation tools (q3map, bspc) via file I/O or subprocess calls.

## Potential Issues

- **Global variable spaghetti**: The reliance on `g_pParentWnd`, `g_Preferences`, `g_nUpdateBits`, `g_bScreenUpdates` makes it difficult to refactor, test, or reuse subsystems independently. A multi-instance editor would require significant refactoring.
- **Incomplete command-to-handler mapping** (inferred): The command table lists ~150 commands, but only some map directly to `ON_COMMAND` message map entries. The dispatch path for unmapped commands is unclear (likely handled dynamically at runtime or via secondary lookup).
- **No visible input validation**: Menu items and keyboard shortcuts are dispatched without apparent bounds checking on the command table index or modifier validation. Malformed keybinding configs could cause out-of-bounds access.
- **Thread safety**: Global state (`g_nUpdateBits`, `g_bScreenUpdates`) is accessed from multiple message handlers without synchronization. If BSP compilation or other operations run in background threads, races are possible.
