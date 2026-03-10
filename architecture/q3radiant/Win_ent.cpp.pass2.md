# q3radiant/Win_ent.cpp — Enhanced Analysis

## Architectural Role

This file implements the Windows-specific entity property editor dialog for the Radiant level editor. It manages a persistent modeless palette window that displays and allows editing of entity class definitions and their key/value property pairs. The file bridges Q3Radiant's UI framework with the underlying entity metadata (`eclass_t` structures) and provides a tabbed inspector interface for switching between entity, texture, console, and grouping modes.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant main window** (`MainFrm` / `g_pParentWnd`) initiates entity editor creation via `CreateEntityWindow()` and switches modes via `SetInspectorMode()`
- **Entity class browser** (`eclass.c` equivalent): `FillClassList()` populates the editor's listbox from the global `eclass` linked-list singleton
- **Global radiant state** (`g_qeglobals`): reads/writes window handles (`d_hwndEntity`, `d_hwndEdit`); accesses main window handle (`d_hwndMain`)
- **Tab control system** (`g_wndTabs`): manages multi-mode inspector tabs (Entities, Textures, Console, Groups)

### Outgoing (what this file depends on)
- **Entity metadata** (`eclass_t` structures): accesses `pec->name` and `pec->next` to enumerate available entity classes
- **Texture editor** (`g_pParentWnd->GetTexWnd()`): called when switching to W_TEXTURE mode
- **Win32 API**: `CreateWindow`, `CreateDialog`, `GetDlgItem`, `SetWindowLong`, `GetWindowLong`, `SendMessage` for GUI construction and event routing
- **Platform globals** (`g_qeglobals`, `g_pParentWnd`): shared window handle registry and parent window reference
- **Window placement persistence**: `LoadWindowPlacement()` to restore editor geometry across sessions

## Design Patterns & Rationale

1. **Message Subclassing Pattern**: `FieldWndProc` and `EntityListWndProc` wrap standard Windows controls (`edit` and `listbox`) to intercept and custom-handle Tab/Enter keystrokes. This allows context-aware navigation (Tab between key/value fields, Enter to commit property).

2. **Control Handle Pooling**: The `hwndEnt[EntLast]` array centralizes all UI element handles, enabling frame-independent access and consistent styling (all controls use `DEFAULT_GUI_FONT`). Simplifies later property updates and layout recalculation.

3. **Modeless Dialog + Reparenting**: The editor creates a temporary dialog via `CreateDialog()`, extracts its controls, and reparents them into a custom window class (`ENT_WINDOW_CLASS`). This hybrid approach retains dialog resource-binding for initial layout while gaining custom frame rendering and z-order control.

4. **Mode-Based Inspector Switching**: `SetInspectorMode()` centralizes UI state transitions. The underlying philosophy is a single persistent window that morphs its content and title based on inspector mode, rather than multiple separate windows. This saves memory and simplifies state coherence.

5. **Tab-Driven Context**: The `CTabCtrl` (MFC wrapper) provides visual mode switching. The conditional logic checks `g_pParentWnd->CurrentStyle()` to conditionally show texture/console tabs depending on the editor layout mode (QE4 single-window vs. QE3/split layouts).

## Data Flow Through This File

1. **Initialization** (`CreateEntityWindow`):
   - Create window class and main window → Create temporary palette dialog → Extract controls via `GetEntityControls()` → Destroy dialog → Subclass key/value fields and entity list → Populate listbox via `FillClassList()` → Show window hidden; set inspector mode to W_CONSOLE

2. **Entity Class Display** (`FillClassList`):
   - Iterate global `eclass` linked-list → For each entity class, send `LB_ADDSTRING` message to listbox → Store `eclass_t*` pointer as item data via `LB_SETITEMDATA`

3. **Property Editing**:
   - User enters key in `EntKeyField` → Presses Tab/Enter → `FieldWndProc` intercepts, clears `EntValueField`, moves focus
   - User enters value → Presses Enter → `AddProp()` called (implementation not shown in truncated file)
   - Key/value pair either added to properties listbox or applied to selected entity

4. **Mode Switching** (`SetInspectorMode`):
   - Caller requests mode (W_ENTITY, W_TEXTURE, W_CONSOLE, W_GROUP) or -1 for cycle
   - Advance internal `inspector_mode` state → Update window title → Enable/disable menu item (entity color selection) → Reposition tab control → Trigger redraw

## Learning Notes

- **Win32 GUI Archaeology**: This code exemplifies late-1990s/early-2000s Windows UI patterns. Modern Qt/GTK would abstract window messages; here they're handled raw. The subclassing pattern is a common Win32 idiom for extending built-in control behavior without full replacement.

- **Hybrid Dialog/Custom-Window Pattern**: The reparenting dance (`GetDlgItem` → `SetParent` → later `CreateWindow` for listboxes) reflects practical constraints of resource-based dialogs; you can't easily embed custom-drawn widgets in a dialog template, so hybrid approaches were common.

- **Radiant Editor Architecture**: q3radiant is not part of the runtime engine. It's an offline authoring tool that produces `.map` files consumed by `q3map` (BSP compiler, in `code/bspc`/`code/q3map`). The entity editor bridges the human-facing GUI to the lower-level entity class definitions that are eventually baked into the compiled BSP.

- **Idiomatic Radiant UI**: The tabbed inspector (Entities/Textures/Console/Groups) is signature Radiant UI. Modern engines often use docked panels or property inspectors; Radiant chose a shared modeless palette with mode tabs. This trades screen real estate for simplicity of implementation.

- **Missing Implementations**: The truncated file cuts off key functions like `AddProp()`, property serialization, and event dispatch from the entity listbox selection. Those would show how edited properties flow back to the map model.

## Potential Issues

1. **Unchecked Window Creation**: All `CreateWindow()` calls check the returned HWND but call `Error()` on failure, which likely uses `longjmp`. This is acceptable for a tool (not shipped with the runtime), but non-fatal errors (e.g., out of memory mid-dialog initialization) would crash the editor entirely rather than gracefully degrade.

2. **Subclass Proc Leakage**: `OldFieldWindowProc` and `OldEntityListWindowProc` are global function pointers set once. If a window is destroyed and recreated (mode switching?), restoring the old proc chain could fail. The code doesn't appear to handle this scenario.

3. **Focus Management**: `FieldWndProc` sets focus to `g_qeglobals.d_hwndCamera` on Escape. If the camera window is destroyed or not yet created, this would dereference a stale handle, causing undefined behavior.

4. **Tab Control Initialization Order**: The code inserts tab items multiple times (once unconditionally for "Groups" or "Entities", then conditionally for "Textures"/"Console"). The order depends on `CurrentStyle()` checks and is fragile to refactoring.
