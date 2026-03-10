# code/win32/win_gamma.c

## File Purpose
Manages hardware gamma ramp correction on Win32, using either the 3Dfx-specific WGL extension or the standard Win32 `SetDeviceGammaRamp` API. It saves the original gamma on init, applies game-specified gamma tables per frame, and restores the original on shutdown.

## Core Responsibilities
- Detect whether the hardware/driver supports gamma ramp modification (`WG_CheckHardwareGamma`)
- Save the pre-game hardware gamma ramp for later restoration
- Validate saved gamma ramp sanity (monotonically increasing, crash-recovery linear fallback)
- Apply per-channel RGB gamma ramp tables to the display device (`GLimp_SetGamma`)
- Apply Windows 2000-specific gamma clamping restrictions
- Enforce monotonically increasing gamma values before submission
- Restore original hardware gamma on game exit (`WG_RestoreGamma`)

## Key Types / Data Structures
None declared in this file; relies on `glconfig_t` (from `tr_local.h`) and `glwstate_t` (from `glw_win.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_oldHardwareGamma` | `unsigned short[3][256]` | static (file) | Stores the original hardware gamma ramp (R/G/B × 256 entries) captured at init, used to restore display on exit |

## Key Functions / Methods

### WG_CheckHardwareGamma
- **Signature:** `void WG_CheckHardwareGamma( void )`
- **Purpose:** Probes hardware gamma support; saves current gamma ramp into `s_oldHardwareGamma`; sets `glConfig.deviceSupportsGamma`
- **Inputs:** None (reads `glConfig.driverType`, `r_ignorehwgamma` cvar, `qwglSetDeviceGammaRamp3DFX`)
- **Outputs/Return:** void; mutates `glConfig.deviceSupportsGamma` and `s_oldHardwareGamma`
- **Side effects:** Calls `GetDC`/`ReleaseDC` on the desktop window; prints `PRINT_WARNING` on broken or suspicious gamma tables; overwrites `s_oldHardwareGamma` with a linear ramp if a prior crash is detected (entry 181 high-byte == 255)
- **Calls:** `GetDC`, `ReleaseDC`, `GetDesktopWindow`, `qwglGetDeviceGammaRamp3DFX`, `GetDeviceGammaRamp`, `ri.Printf`
- **Notes:** `GLDRV_STANDALONE` drivers unconditionally skip gamma support (non-3Dfx path only). Sanity check verifies `table[c][255] > table[c][0]` for all channels.

### GLimp_SetGamma
- **Signature:** `void GLimp_SetGamma( unsigned char red[256], unsigned char green[256], unsigned char blue[256] )`
- **Purpose:** Converts 8-bit per-channel gamma arrays to 16-bit ramps and submits them to the display device
- **Inputs:** Three 256-entry byte arrays for red, green, blue gamma correction curves
- **Outputs/Return:** void
- **Side effects:** Calls `GetVersionEx` to detect Windows 2000 and apply per-entry clamping; calls `SetDeviceGammaRamp` or `qwglSetDeviceGammaRamp3DFX` on `glw_state.hDC`; prints via `Com_DPrintf`/`Com_Printf`
- **Calls:** `GetVersionEx`, `SetDeviceGammaRamp`, `qwglSetDeviceGammaRamp3DFX`, `Com_DPrintf`, `Com_Printf`
- **Notes:** Early-outs if `!glConfig.deviceSupportsGamma`, `r_ignorehwgamma` is set, or `glw_state.hDC` is null. Win2K clamp: entries `[0..127]` capped at `(128+i)<<8`; entry 127 additionally capped at `254<<8`. Monotonicity enforcement pass runs unconditionally after W2K clamping.

### WG_RestoreGamma
- **Signature:** `void WG_RestoreGamma( void )`
- **Purpose:** Restores the saved `s_oldHardwareGamma` ramp to the display device on shutdown
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `SetDeviceGammaRamp` or `qwglSetDeviceGammaRamp3DFX`; for the non-3Dfx path, acquires/releases a fresh DC on the desktop window (does **not** use `glw_state.hDC`, safe to call after context teardown)
- **Calls:** `qwglSetDeviceGammaRamp3DFX`, `GetDC`, `ReleaseDC`, `GetDesktopWindow`, `SetDeviceGammaRamp`
- **Notes:** No-op if `glConfig.deviceSupportsGamma` is false.

## Control Flow Notes
- **Init:** `WG_CheckHardwareGamma` is called during renderer initialization (from `GLimp_Init` / `win_glimp.c`) before the first frame.
- **Per-frame:** `GLimp_SetGamma` is called by `R_SetColorMappings` whenever gamma-related cvars change; it is not called every frame unconditionally.
- **Shutdown:** `WG_RestoreGamma` is called from `GLimp_Shutdown` to undo display gamma before the window is destroyed.

## External Dependencies
- `<assert.h>` — standard C (unused in active code paths)
- `../renderer/tr_local.h` — `glConfig` (`glconfig_t`), `ri` (`refimport_t`), `r_ignorehwgamma` cvar
- `../qcommon/qcommon.h` — `Com_DPrintf`, `Com_Printf`
- `glw_win.h` — `glw_state` (`glwstate_t`), exposes `glw_state.hDC`
- `win_local.h` — Win32 headers (`windows.h`), `OSVERSIONINFO`, `GetVersionEx`
- `qwglSetDeviceGammaRamp3DFX`, `qwglGetDeviceGammaRamp3DFX` — defined elsewhere (WGL extension pointers, loaded in `win_glimp.c`)
- `glConfig.deviceSupportsGamma`, `glConfig.driverType` — defined in renderer globals (`tr_init.c`)
