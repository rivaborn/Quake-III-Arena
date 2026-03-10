# q3radiant/ScriptDlg.cpp — Enhanced Analysis

## Architectural Role
This file implements a simple modal dialog for the Q3Radiant level editor that allows users to discover and execute custom editor scripts. Located in the **level editor tool subsystem** (outside the runtime engine), it bridges the editor's GUI framework (MFC) to the editor's script execution layer. The dialog reads a static script registry at initialization time and provides interactive script selection without feedback to the main editor workflow.

## Key Cross-References

### Incoming (who depends on this file)
- Editor main frame (`q3radiant/MainFrm.cpp` or similar): Likely instantiates and shows `CScriptDlg` modally in response to a menu action
- MFC framework: Calls `OnInitDialog()`, message handlers (`OnRun()`, `OnDblclkListScripts()`), and `DoDataExchange()`

### Outgoing (what this file depends on)
- `RunScriptByName()`: External editor function (location unknown from provided context, likely in another q3radiant file) that executes the named script
- Windows API: `GetPrivateProfileSectionNames()` from `kernel32.dll` for INI file parsing
- `g_strAppPath`: Global path constant from editor initialization (likely in `q3radiant/Radiant.cpp` or `QE3.cpp`)
- MFC message map system: `CDialog` base, `DDX_Control`, `DDX_LBString`

## Design Patterns & Rationale

### Dialog-Based Configuration
This is a simple **modal selector dialog** pattern—the user cannot interact with the main editor while it's open. No settings are persisted; the dialog is purely a read-once, execute-once interface. This fits the lightweight nature of editor tooling circa 2005 (pre-async UI refactoring eras).

### INI-Based Registry
Scripts are discovered by parsing a `scripts.ini` file in the editor's app directory at dialog init time. The section names become the script names. This avoids hard-coding script lists and allows user/mod customization via text file editing—a common pattern in older game tools.

### Message Routing Symmetry
Both the **Run button** and **double-click on list** trigger the same code path (`UpdateData()`, `EndDialog(IDOK)`, `RunScriptByName()`). This avoids duplication and ensures consistent behavior regardless of invocation method—idiomatic MFC modal dialog style.

## Data Flow Through This File

1. **Init**: `OnInitDialog()` is called by MFC framework
   - Allocate temporary 16 KB buffer
   - Read section names from `scripts.ini` via Windows API
   - Populate list control `IDC_LIST_SCRIPTS` with script names
   - Deallocate buffer
   - Return `TRUE`

2. **User Interaction**: Dialog is modal and blocking
   - User selects a script from the list (or highlights one and clicks Run)
   - User clicks Run button OR double-clicks the list item

3. **Execution & Close**:
   - `UpdateData(TRUE)` copies the selected script name from the list control into member variable `m_strScript`
   - `EndDialog(IDOK)` closes the dialog and returns control to the caller
   - `RunScriptByName(m_strScript.GetBuffer(0), true)` executes the script asynchronously (or synchronously—the `true` flag is unclear without seeing the function definition)

## Learning Notes

### Idiomatic MFC Dialog Workflow
This file exemplifies **1990s–2000s GUI conventions** for modal dialogs:
- DDX/DDV automatic data marshaling between controls and member variables
- Message map macros for event routing
- `DoDataExchange()` as the centralized binding point
- `CString` UNICODE/ANSI wrapper to avoid manual buffer management

Modern engines (Unreal, Unity) use **async UI stacks** and **data binding frameworks**; this dialog design is synchronous and stateful—reflecting the constraints of pre-2010 tool development.

### Script Discovery Pattern
The `GetPrivateProfileSectionNames()` enumeration is a **Windows-only, INI-file-specific pattern**. Modern tools use:
- JSON/YAML config files with structured parsing
- Directory scanning (e.g., `scripts/` folder with one file per script)
- Plugin or module systems with reflection

The INI pattern here is simple but inflexible: adding a script requires manual INI edits.

### Memory Leak Potential
Line 87–98 allocates a 16 KB buffer on the stack via `new char[16384]` and deallocates it at line 97. Modern C++ would use `std::string` or `std::vector<char>` to avoid manual cleanup. This code is safe (does delete the allocation) but is defensive against a more modern style.

## Potential Issues

1. **Hard-coded Script Path**: The script file is assumed to be `{g_strAppPath}\scripts.ini`. If `g_strAppPath` is not initialized or the file doesn't exist, `GetPrivateProfileSectionNames()` silently returns 0, and the user sees an empty script list with no error message. No validation or user feedback.

2. **Fixed Buffer Size**: A 16 KB buffer for section names is assumed to be sufficient. If `scripts.ini` contains very long script names or a very large number of scripts, the buffer overflows silently (the return value `n` is ignored). This is a **classic buffer-overflow vulnerability**, though mitigated by the fact that the file is local and controlled by the user/admin.

3. **No Error Handling**: `RunScriptByName()` is called with no error handling. If the script file doesn't exist or has syntax errors, the user receives no feedback from this dialog.

4. **String Encoding Assumptions**: `GetBuffer(0)` returns a C string pointer; if `m_strScript` contains non-ASCII characters, `RunScriptByName()` must be ANSI-aware or the engine must support widechar. No explicit encoding contract is visible.

5. **Editor Isolation**: This dialog is completely isolated from the runtime engine architecture (no calls into `qcommon`, `q_shared`, etc.). This is correct for a tool, but suggests that editor scripts are a **separate feature layer** with no shared infrastructure—potential for divergence if the same scripting language is used elsewhere.
