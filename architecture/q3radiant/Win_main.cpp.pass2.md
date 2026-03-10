# q3radiant/Win_main.cpp — Enhanced Analysis

## Architectural Role

This file bridges the Q3Radiant level editor (a standalone Win32 application) to the offline compilation toolchain. It manages the launch lifecycle of BSP/VIS/LIGHT operations—whether as external processes or via internal DLL execution—and orchestrates post-compilation workflows (pointfile inspection, optional game launch). Critically, it sits in the **editor-to-tools boundary**, not the runtime engine; it has no direct interaction with `qcommon/`, renderer, client, or server.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main UI layer** (`qe3.h`, `entityw.h`, `PrefsDlg.h`): editor menus and preferences dialogs call `RunBsp()` in response to user menu actions
- **Editor global state** (`g_qeglobals.d_project_entity`, `g_qeglobals.d_hwndMain`, `currentmap`): reads project settings and map state
- **Inspector/console UI** (`SetInspectorMode(W_CONSOLE)`): directs BSP output to the embedded console window

### Outgoing (what this file depends on)
- **q3map/bspc toolchain**: invoked either via `CreateProcess()` (external `.bat` script wrapper) or as a DLL via internal threading (`RunTools()`, `ThreadTools()`)
- **Windows process management**: `CreateProcess()`, `GetExitCodeProcess()`, `Sleep()`, `WaitForSingleObject()`
- **IPC mechanisms**: 
  - Custom window message `wm_AddCommand` (registered via `RegisterWindowMessage()`) for q3map process server communication
  - `GlobalAddAtom()` + `PostMessage()` to send commands to a remote "Q3Map Process Client" window if running
- **File I/O**: temporary directory (`GetTempPath()`), map save/load coordination with editor (`Map_SaveFile()`)
- **Time tracking**: `CTime::GetCurrentTime()` for elapsed-time reporting after build
- **Pointfile integration**: `Pointfile_Delete()`, `Pointfile_Check()` (leak visualization)
- **Game launch** (optional): `WinExec()` to run Quake 3 with the compiled map, with path resolution and file copy

## Design Patterns & Rationale

1. **Dual Execution Modes** (`g_PrefsDlg.m_bInternalBSP` flag):
   - **External process**: Traditional `CreateProcess()` → `.bat` wrapper → tool stdio capture to temp file
   - **Internal DLL**: threaded `RunTools(p, g_hWnd, ...)` for more responsive UI (avoids blocking); shares memory with editor process
   - **Rationale**: external is safer/isolated; internal is faster but requires DLL availability

2. **Process Server Pattern** (remote q3map):
   - If `q3map_process_server` executable is running and window found, split BSP commands into atoms and post to it via `wm_AddCommand`
   - **Rationale**: enables distributed compilation; editor doesn't block; queries can queue on a dedicated machine

3. **String Template Expansion** (`QE_ExpandBspString`):
   - Substitutes `$` (source path), `!` (rsh prefix), `@` (quote marker) in command strings from project entity
   - **Rationale**: defers tool paths and remote-execution commands to project configuration; supports both local and network execution

4. **Temp File & Batch Wrapper**:
   - `.bat` file written to system temp, executed via `CreateProcess()`, output redirected to `junk.txt`
   - **Rationale**: pre-Win2k workaround for command-line length limits and stdio capture; obsolete by modern standards but retained for compatibility

## Data Flow Through This File

1. **Trigger**: User selects menu "BSP" → editor calls `RunBsp(command_string)` with action key (e.g., `"bsp"`).
2. **Pre-flight**:
   - Check if BSP already running or tool thread active → bail if so
   - Query project entity for `rshcmd` (remote shell), path settings, tool paths
   - Save current map to disk (or `.reg` region file if `region_active`)
3. **Path Resolution**:
   - Extract or build working path from map filename (DOS → Unix conversion for remote)
   - Check for running "Q3Map Process Client" window (IPC shortcut)
4. **Command Expansion**:
   - Substitute variables into template from project entity (source path, remote basepath, output redirection)
5. **Execution**:
   - If process server found: split by `&&`, strip remote shell prefix, post atoms to server window → return immediately
   - Else if internal DLL enabled: spawn thread running `RunTools()` with DLL command string
   - Else: write `.bat`, `CreateProcess()`, monitor with `GetExitCodeProcess()` polling
6. **Post-Build** (`DLLBuildDone()`):
   - Print elapsed time
   - `Pointfile_Check()` → highlight leaks in editor if found
   - If `m_bRunQuake`, launch game with map or copy BSP to game directory and launch with `+map` flag

## Learning Notes

- **Editor-to-Toolchain Bridge Pattern**: Separates level editor (UI/viewport) from compilation (heavy CPU/I/O). Many engines use similar IPC or plugin patterns.
- **Pre-Modern Windows Workarounds**: `.bat` wrappers and `junk.txt` redirection reflect early-2000s Windows quirks (command-line limits, no native process stdio capture). Modern code would use `HANDLE` pipes directly.
- **Thread-Safe Globals** (`g_hToolThread`, `g_hWnd`): minimal synchronization; assumes single-threaded editor event loop calling this code. Races possible if UI tries to launch BSP while thread still running—mitigated by `bsp_process` check.
- **Atom-Based IPC** (`GlobalAddAtom` / `wm_AddCommand`): lightweight way to queue short strings (up to 255 chars) across process boundaries without shared memory; q3map process server listens on named window and dequeues atoms.
- **Idiomatic Quake III**:
  - Project entity (`g_qeglobals.d_project_entity`) as a key-value store for tool configuration (no .ini or XML)
  - Tight coupling to Win32 console (`Sys_Printf` into embedded W_CONSOLE window)
  - Manual time tracking (`CTime`) rather than profiler integration

## Potential Issues

1. **Race Condition on `g_hToolThread`**: If user launches BSP while internal thread still running, code checks `g_hToolThread` but doesn't synchronize with thread completion. `GetExitCodeProcess()` is called but result not used atomically.
2. **Path Conversion Assumption**: `QE_ExpandBspString` assumes "maps/" or "maps\\" subdirectory exists in map filename; fails silently if not found, falls back to `ExtractFileName()`.
3. **Hardcoded Remote Paths**: `baseq2\maps\` path for game launch is hardcoded; assumes Q3A filesystem layout but code references "baseq2" (Quake 2 artifact?).
4. **Process Server Window Search**: `EnumChildWindows()` on entire desktop is O(n windows) and can be slow; no timeout on `FindWindow()` if server is hung.
5. **No Error Recovery**: If `CreateProcess()` fails, calls `Error()` (fatal); no graceful fallback or user notification via dialog.
