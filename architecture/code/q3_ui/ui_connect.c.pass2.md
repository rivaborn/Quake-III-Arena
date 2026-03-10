# code/q3_ui/ui_connect.c — Enhanced Analysis

## Architectural Role
This file implements a single UI screen within the Q3A UI VM (`code/q3_ui`), specifically the connection/loading screen shown during the client's transition from disconnected → server-connected states. It bridges the client engine's connection state machine (tracked in `uiClientState_t`) with the UI rendering layer, displaying real-time progress feedback and handling the ESC key to abort connections. This screen is also overlaid on the cgame loading screen to prevent visual flicker during fast LAN/local connects.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/client/cl_ui.c`**: The client engine invokes this file's functions via the UI VM's `vmMain` dispatch. `UI_DrawConnectScreen` is called each frame during `CA_CONNECTING` / `CA_CHALLENGING` / `CA_CONNECTED` states; `UI_KeyConnect` handles keyboard input routed by the UI input dispatcher.
- **`code/cgame/cg_main.c`** (or equivalent): May call `UI_DrawConnectScreen(..., qtrue)` as an overlay during the cgame's own loading phase to prevent screen-blinking artifacts.

### Outgoing (what this file depends on)
- **Client engine state**: Reads `uiClientState_t` (populated by `code/client/cl_parse.c`) via `trap_GetClientState`, and reads configstrings/cvars via `trap_GetConfigString` and `trap_Cvar_VariableValue`.
- **UI rendering subsystem** (`code/q3_ui/ui_atoms.c` / `ui_main.c`): Calls `UI_SetColor`, `UI_DrawHandlePic`, `UI_DrawProportionalString`, `UI_DrawProportionalString_AutoWrapped`, `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`. These are the Q3A UI VM's 2D rendering API, wrapping renderer syscalls.
- **Global UI state** (`uis` from `code/q3_ui/ui_main.c`): Reads `uis.realtime` and `uis.frametime` to compute elapsed transfer time.
- **Utility functions**: `Menu_Cache` (shader precache), `Info_ValueForKey` (parse key-value pairs from configstrings), `va` (formatted strings), `Com_sprintf` (safe sprintf).

## Design Patterns & Rationale

1. **Screen Controller Pattern**: `UI_DrawConnectScreen` + `UI_KeyConnect` form a cohesive unit responsible for the entire connection screen's lifecycle. This mirrors the Q3A UI's menu system architecture where each screen is a draw function + key handler pair.

2. **State-Machine Display**: The function reads the client's connection state enum (`connstate_t`) and selects the appropriate display text and sub-screens. Early returns for `CA_LOADING`/`CA_PRIMED` optimize for states where the game is taking over rendering.

3. **Graceful Degradation**: Download ETA calculation guards against division-by-zero and falling-back to "estimating" when elapsed time is zero or transfer rate is unmeasured. This avoids UI crashes on transient network conditions.

4. **Utility Helpers**: `UI_ReadableSize` and `UI_PrintTime` format raw numeric values into human-readable strings. This is idiomatic for Q3A's data-agnostic rendering layer—the file doesn't assume anything about the precision of its inputs.

## Data Flow Through This File

```
Client State Machine (code/client)
         ↓
trap_GetClientState() → uiClientState_t { connState, servername, messageString, ... }
         ↓
UI_DrawConnectScreen()
  ├─ Reads: cstate.connState, CS_SERVERINFO configstring (map name), cstate.updateInfoString (MOTD)
  ├─ On CA_CONNECTED: reads cl_downloadName cvar → delegates to UI_DisplayDownloadInfo
  │    ├─ Reads: cl_downloadSize, cl_downloadCount, cl_downloadTime cvars
  │    ├─ Computes: transfer rate (KB/sec), ETA, bytes transferred
  │    ├─ Formats: UI_ReadableSize → "123.45 MB", UI_PrintTime → "2 min 30 sec"
  │    └─ Renders: 2D text overlay with file progress, rate, ETA
  ├─ Other states (CA_CONNECTING, CA_CHALLENGING): renders simple state text
  └─ Outputs: 2D rendered text via UI_DrawProportionalString syscalls

User presses ESC
         ↓
UI_KeyConnect(K_ESCAPE)
         ↓
trap_Cmd_ExecuteText(EXEC_APPEND, "disconnect\n")
         ↓
Client command FIFO → Server state machine abort
```

## Learning Notes

- **UI VM Architecture**: Q3A UI modules are fully sandboxed; they cannot directly access engine state or make syscalls beyond the published `trap_*` interface. This file exemplifies the pattern: all data comes from trap calls, all output goes to trap calls.
- **Virtual Coordinate System**: The UI layer uses a fixed 640×480 virtual coordinate space, independent of the actual framebuffer. This enables consistent UI layout across screen resolutions.
- **Connection State Visibility**: The file demonstrates how the engine's connection state machine (owned by `code/client`) becomes visible to the UI VM. Modern engines often use a centralized "app state" struct; Q3A distributes state (client state, player state, entity state) across multiple query functions.
- **Dead Code Pattern**: `lastLoadingText` is allocated but never read for comparison—only cleared on state regression. This suggests incomplete refactoring or abandoned text-caching optimization.
- **Integer Overflow Awareness** (line ~156): The ETA computation divides by 1024 before multiplying to avoid 32-bit integer overflow around 4 MB transfers. This is a micro-optimization for 2000s-era machines with slower networks; modern engines would use 64-bit integers.

## Potential Issues

1. **Unused Static Variable**: `lastLoadingText[MAX_INFO_VALUE]` is cleared but never read. If the intent was to cache and compare text to avoid redundant redraws, that logic is missing. If not needed, it's dead code.
2. **Password Field Completely Disabled**: The entire password input block is guarded by `#if 0`, suggesting incomplete/removed feature. The global `passwordNeeded` is always `qtrue` but unused.
3. **Missing Null-Checks on Format Buffers**: Functions like `UI_ReadableSize` and `UI_PrintTime` assume `buf` is non-null; no defensive checks. This is acceptable if the caller contract guarantees non-null, but is worth noting.
4. **Potential Precision Loss in ETA**: The ETA formula uses integer arithmetic (KB-scaled division); floating-point estimates might be more accurate for slower transfers or very large files.
