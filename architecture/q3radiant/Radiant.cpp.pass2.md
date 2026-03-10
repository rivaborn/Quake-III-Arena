# q3radiant/Radiant.cpp — Enhanced Analysis

## Architectural Role

This file is the **MFC application bootstrap and frame loop** for Q3Radiant, the official Quake III Arena level editor. It initializes the Windows application, manages the OpenGL context lifecycle, and drives the editor's per-frame message processing. Radiant sits **outside the runtime engine** — it produces `.map` source files that are compiled offline by `q3map` (BSP) and `bspc` (AAS) tools into assets consumed by the server and client at runtime.

## Key Cross-References

### Incoming
- **Windows OS**: implicit `WinMain()` entry point (provided by MFC)
- **MFC framework**: message dispatch (`BEGIN_MESSAGE_MAP`) for file/help commands
- **User input**: keyboard/mouse routed through MFC to child frame handlers

### Outgoing
- **OpenGL dynamic loader** (`QGL_Init`, `QGL_Shutdown`): establishes GL function pointers at runtime, with fallback from `opengl32.dll` → `opengl.dll` (SGI variant)
- **Preferences dialog** (`g_PrefsDlg`): reads/writes OpenGL backend selection (`m_bSGIOpenGL`)
- **Main frame** (`CMainFrame`): creates window, loads menu/accelerator tables, receives `RoutineProcessing()` calls
- **Registry/filesystem**: stores editor state in either Windows registry or local `REGISTRY.INI`
- **Help system**: launches `Q3RManual.chm` via `ShellExecute`

### Architectural position
The editor is **not part of the runtime engine execution path**. Maps flow: `.map` (Radiant) → `q3map` (BSP compiler) → `.bsp` (runtime). The editor uses OpenGL only for the 3D viewport; it does not invoke the renderer DLL that the runtime engine loads.

## Design Patterns & Rationale

1. **MFC MDI skeleton**: Standard Microsoft Foundation Classes pattern for Windows multi-document interface (era-typical mid-2000s choice; deprecated by 2010s)

2. **Registry vs .INI fallback**: Checks for local `REGISTRY.INI` first, falls back to Windows registry. This allows distribution as a standalone folder without installer registration — defensive design for portability.

3. **Dynamic OpenGL binding with retry**: Attempts `opengl32.dll + glu32.dll` first (Microsoft standard), retries with `opengl.dll + glu.dll` (SGI variant used on some deployments). This dual-loading pattern was necessary because OpenGL availability varied by Windows version/GPU driver era.

4. **Idle-loop dispatch**: `OnIdle()` acts like a game-engine frame loop, calling `g_pParentWnd->RoutineProcessing()` repeatedly — idiomatic for interactive GUI apps of the era that needed high-frequency updates (viewport animation, background operations).

5. **Per-instance registry namespace**: Rather than a single global registry key, Radiant allocates unique `Software\Q3Radiant\IniPrefs<N>` keys per instance (lines 130–160). This allows multiple Radiant installations on one machine without collisions, storing opaque binary buffers (void pointers) that can't fit in `.INI` files.

## Data Flow Through This File

```
Windows Startup
  ↓
CRadiantApp::InitInstance()
  ├─ Probe for REGISTRY.INI in exe directory
  ├─ If found: read/write prefs to .INI + allocated registry key
  │           (registry holds void* pointers; .INI holds text values)
  ├─ Else: use standard registry path
  ├─ QGL_Init(opengl32.dll, glu32.dll) [+ retry with opengl.dll, glu.dll]
  ├─ Parse command line for "builddefs" flag → g_bBuildList
  ├─ Create CMainFrame, load IDR_MENU1, IDR_MINIACCEL
  ├─ Set help file path → m_pszHelpFilePath
  └─ ShowWindow()
  
Per-frame (implicit in MFC)
  ↓
CRadiantApp::OnIdle() [loop while OS allows]
  └─ g_pParentWnd->RoutineProcessing()
  
Shutdown
  ↓
CRadiantApp::ExitInstance()
  └─ QGL_Shutdown()
```

## Learning Notes

1. **MFC is Windows-specific and era-bound**: This codebase reflects early-2000s Windows C++ idioms (MFC, registry, WinAPI). By 2010+ the industry shifted to C#/.NET or cross-platform Qt/wxWidgets for editors.

2. **Dynamic GL loading was essential**: In the OpenGL 1.x era, function pointers had to be discovered at runtime, especially across Windows versions and GPU drivers. The editor shares this pattern with the runtime renderer (`code/renderer/qgl.h`). Modern OpenGL loading is typically via GLEW, GLAD, or bundled with the platform API.

3. **Idiomatic Quake III era design**:
   - Offline compilation pipeline: editor → compiler tools → runtime assets (no runtime code generation)
   - Separate tool/runtime code paths (Radiant never runs game logic; just geometry/entity)
   - Registry/INI split for installer-free distribution (common for source-released games)
   - Idle-loop pattern borrowed from realtime game engines (but here used for UI updates, not physics)

4. **Tool bootstrapping**: The `g_bBuildList` flag (set by "builddefs" command-line arg) suggests Radiant can batch-generate BSP data without user interaction — useful for automated build pipelines.

## Potential Issues

1. **Unsafe string operations** (lines 125, 139, 149): `strcpy()`, `sprintf()` without bounds checking. By modern standards, should use `strncpy()` or safer wrappers.

2. **Unbounded registry key search** (lines 130–160): The loop incrementing `i` has no upper bound — if all keys from `IniPrefs0` to `IniPrefs2147483647` are occupied, the loop never terminates (though practically impossible).

3. **Silent help failure**: If `Q3RManual.chm` doesn't exist at the constructed path, `ShellExecute()` fails silently with no user feedback (line 237).

4. **Hardcoded paths and registry keys**: `"Software\\Q3Radiant\\IniPrefs"`, `"Q3RManual.chm"` are magic strings. Moving or renaming these breaks user installations.

5. **Deprecated framework**: MFC is no longer actively maintained; linking against older versions risks security/compatibility issues on modern Windows.
