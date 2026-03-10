# code/win32/win_gamma.c — Enhanced Analysis

## Architectural Role

This file implements the Win32 platform abstraction layer for hardware gamma correction, sitting between the renderer's color pipeline (`tr_local.h`) and the Win32 display subsystem. It's part of the renderer's `GLimp_*` platform interface—the renderer calls `GLimp_SetGamma` whenever gamma cvars change (e.g., `r_gamma`, `r_intensity`), and this file translates those requests into Win32 API calls. The file also anchors the renderer's lifecycle: it captures the pre-game gamma ramp during `GLimp_Init` (via `WG_CheckHardwareGamma`) and restores it on `GLimp_Shutdown` to undo any display state changes.

## Key Cross-References

### Incoming (who depends on this file)
- **`win_glimp.c`** — calls `WG_CheckHardwareGamma()` during renderer init, calls `GLimp_SetGamma()` from `R_SetColorMappings` (renderer front-end), calls `WG_RestoreGamma()` during shutdown
- **Renderer front-end** (`tr_main.c::R_SetColorMappings`) — indirectly via `win_glimp.c` bridge
- **Globals depended upon:**
  - `glConfig.deviceSupportsGamma` (read/write) — set here, read by renderer before calling `GLimp_SetGamma`
  - `glConfig.driverType` (read) — used to detect `GLDRV_STANDALONE` (non-3Dfx drivers)
  - `r_ignorehwgamma` cvar (read) — allows user to disable gamma entirely
  - `glw_state.hDC` (read) — valid after context creation, used for all `SetDeviceGammaRamp` calls

### Outgoing (what this file depends on)
- **`tr_local.h`** — `glConfig` (`glconfig_t`) for capability flags and `ri` (`refimport_t`) for logging
- **`qcommon.h`** — `Com_DPrintf`, `Com_Printf` for debug/error output
- **`glw_win.h`** — `glw_state` (`glwstate_t`), specifically `glw_state.hDC` (device context handle)
- **`win_local.h`** — brings in `<windows.h>`, `OSVERSIONINFO`, platform macros
- **WGL extension pointers** — `qwglSetDeviceGammaRamp3DFX`, `qwglGetDeviceGammaRamp3DFX` (loaded elsewhere in `win_glimp.c`, checked at runtime)
- **Win32 API:** `GetDC`, `ReleaseDC`, `GetDesktopWindow`, `GetVersionEx`, `SetDeviceGammaRamp`

## Design Patterns & Rationale

1. **Lazy Validation + Crash Recovery**
   - Gamma is validated only once during init, not per-frame, reducing overhead
   - Detects prior crashes via a heuristic (entry `[181]` high-byte == 255), falling back to a linear ramp to restore hardware to sane state
   - Reflects real-world game crash scenarios where unrestored gamma tables corrupt the display

2. **Dual API Fallback Chain**
   - Prefers 3Dfx WGL extension (`qwglSetDeviceGammaRamp3DFX`) as fast path if available
   - Falls back to standard Win32 `SetDeviceGammaRamp` for all other drivers
   - Non-3Dfx standalone drivers unconditionally skip gamma (hardware limitation of the era)

3. **Platform-Quirk Conditionals**
   - Win2K gamma ramp clamping (`table[j][i] ≤ (128+i)<<8` for entries 0–127) applied only when OS version matches
   - Avoids assuming uniform behavior across Windows versions (important for 2000/XP era portability)

4. **Monotonicity Enforcement**
   - Final pass ensures `table[j][i] ≥ table[j][i-1]` for all channels after all clamping
   - Prevents gamma "inversions" that could confuse the display hardware
   - Runs unconditionally — simpler than trying to guarantee monotonicity earlier

5. **16-Bit Expansion**
   - Converts 8-bit per-channel input (`unsigned char[256]`) to 16-bit ramps with duplication: `((red[i] << 8) | red[i])`
   - Achieves precision without requiring caller to pre-compute 16-bit values

6. **Safe Resource Management**
   - `GetDC`/`ReleaseDC` pairs are always balanced, even on early-exit paths
   - For restoration path, acquires a fresh DC on desktop window (safe if context has been torn down)

## Data Flow Through This File

```
WG_CheckHardwareGamma (Init):
  GetDC(desktop) → GetDeviceGammaRamp(hDC, s_oldHardwareGamma) → ReleaseDC
  Validate: table[c][255] > table[c][0] for all channels
  Detect crash: if table[0][181].high == 0xFF, replace with linear ramp
  Set glConfig.deviceSupportsGamma ← {true | false}

GLimp_SetGamma (Per-cvar-change):
  Input: red[256], green[256], blue[256] (8-bit per-channel curves)
  Expand: table[c][i] = (input[c][i] << 8) | input[c][i]
  Clamp (Win2K only): table[j][i] ≤ min((128+i)<<8, 254<<8 for i==127)
  Enforce monotonicity: table[j][i] = max(table[j][i], table[j][i-1])
  Apply: SetDeviceGammaRamp(hDC, table) or qwglSetDeviceGammaRamp3DFX(hDC, table)
  Output: Hardware gamma updated

WG_RestoreGamma (Shutdown):
  GetDC(desktop) → SetDeviceGammaRamp(hDC, s_oldHardwareGamma) → ReleaseDC
  (or use 3Dfx path if available)
  Restore display to pre-game state
```

## Learning Notes

1. **Legacy Multi-API Support**
   — The 3Dfx WGL extension path reflects early 2000s hardware diversity; modern engines would drop this, but Q3A supported then-current discrete GPUs with proprietary extensions.

2. **Crash Robustness**
   — The crash-detection heuristic (entry 181) is idiomatic for shipped games: developers discovered this pattern reliably indicated a prior unclean shutdown and built in recovery.

3. **Platform Quirk Handling**
   — Win2K's gamma clamping requirement is not obvious from API docs; this likely came from painful bug reports. Modern engines might test at startup and cache results.

4. **Gamma as Global State**
   — Unlike modern high-level graphics APIs (Vulkan, DX12), OpenGL 1.x exposes hardware gamma as a global display property, not per-context. This file treats it correctly as shared engine state, not renderer-private.

5. **Idiomatic Early-Outs**
   — `GLimp_SetGamma` has multiple guards (`!glConfig.deviceSupportsGamma`, `r_ignorehwgamma->integer`, `!glw_state.hDC`). This makes it safe to call unconditionally from the renderer without additional checks upstream.

## Potential Issues

1. **No Multi-Monitor Support**
   — All code assumes a single desktop window (`GetDesktopWindow()`). Multi-display setups would need per-monitor DC enumeration; not a concern for Q3A era, but limits modern compatibility.

2. **Platform Version Check Fragility**
   — `vinfo.dwMajorVersion == 5 && vinfo.dwPlatformId == VER_PLATFORM_WIN32_NT` is a hardcoded check for Win2K specifically. Windows XP also has major version 5 (NT line), so this clamping may apply to XP as well (either intentionally or by accident). Modern code would cache this at init.

3. **Assumption: Valid State**
   — The file assumes `glw_state.hDC` is valid whenever `glConfig.deviceSupportsGamma == true`. A null check exists in `GLimp_SetGamma`, but there's no defensive validation of the DC handle's actual validity after creation.

4. **Commented Debug Code**
   — `mapGammaMax()` is left in the source (lines 105–122), likely test/exploration code. Minor code hygiene issue, but harmless.

5. **No Fallback if SetDeviceGammaRamp Fails**
   — If `SetDeviceGammaRamp` returns `false` in `GLimp_SetGamma`, the code prints a warning but doesn't retry or degrade gracefully. On some Win32 systems (e.g., remote desktop, certain GPU drivers), gamma may not be supported even if the initial check passed.
