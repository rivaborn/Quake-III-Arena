# code/macosx/macosx_display.h — Enhanced Analysis

## Architectural Role

This file is part of the macOS platform layer's graphics implementation module (GLimp). It provides the complete display and gamma management interface required by the renderer during fullscreen mode transitions, initialization, and shutdown. The functions bridge the platform-independent renderer (`code/renderer/tr_init.c`, `tr_backend.c`) to the Core Graphics display API, enabling hardware-level display capture, gamma ramp control, and pixel-perfect fullscreen switching—critical for Quake III's competitive latency requirements.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_init.c` / `GLimp_Init()`**: Calls `Sys_StoreGammaTables()` at startup to capture original hardware gamma state; calls `Sys_FadeScreens()` before display mode switch.
- **`code/renderer/tr_init.c` / `GLimp_Shutdown()`**: Calls `Sys_UnfadeScreens()` to restore gamma; calls `Sys_ReleaseAllDisplays()` to release display capture on exit.
- **`code/renderer/tr_backend.c` / mode-switch code**: Uses `Sys_GetMatchingDisplayMode()` to enumerate and select a Core Graphics display mode matching the requested resolution/refresh/depth.
- **`code/macosx/macosx_glimp.m`**: The actual GLimp implementation calls these functions as part of the macOS OpenGL context lifecycle.

### Outgoing (what this file depends on)
- **`code/macosx/macosx_local.h`**: Defines `glwgamma_t` (gamma table struct with `display` and per-channel ramp arrays), `glwstate_t` global, and `glw_state` singleton holding original gamma tables.
- **`code/renderer/tr_local.h`**: Provides `qboolean` type.
- **Core Graphics framework** (via includes): `CGDirectDisplayID`, `CGSetDisplayTransferByTable`, `CGDisplayCapture`, `CGReleaseAllDisplays`, `CGDisplayAvailableModes()`.
- **Objective-C runtime** (via `@class NSDictionary` forward declaration): Used for display mode dictionary return type.

## Design Patterns & Rationale

### Hardware Gamma Capture & Restore
The file implements a **save/restore pattern** for hardware gamma ramps: on init, `Sys_StoreGammaTables()` reads the current state into `glw_state.originalDisplayGammaTables[]` so that `Sys_UnfadeScreen()` can restore each display exactly. This is necessary because fullscreen mode switches may reset or corrupt hardware gamma—the game must restore the user's original calibration on exit.

### Display Fade/Unfade Symmetry
`Sys_FadeScreens()` → mode switch → `Sys_UnfadeScreens()` is a classic bracket pattern: fade hides the visual glitch of mode transition, unfade restores the restored gamma. This was standard practice for 2000s fullscreen games to avoid flicker or "gamma flash."

### Per-Display and Bulk Operations
The file exposes both per-display (`Sys_FadeScreen`, `Sys_UnfadeScreen`) and all-display (`Sys_FadeScreens`, `Sys_UnfadeScreens`) variants. This allows the renderer to either manage a single display (single-monitor case) or all displays at once (multi-monitor), a common pattern in macOS where apps may span displays.

### Direct Hardware Ramping (Not Post-Process)
Unlike modern engines using shader-based gamma correction, Q3A directly manipulates the hardware gamma ramp via `CGSetDisplayTransferByTable`. This provides **guaranteed latency-neutral** gamma application (no render-to-framebuffer delay) and was essential for competitive FPS play.

## Data Flow Through This File

```
Startup:
  Sys_StoreGammaTables()
    → reads CGDisplayTransferByTable for all displays
    → stores in glw_state.originalDisplayGammaTables[]

Mode Switch:
  Sys_FadeScreens()
    → interpolates current gamma toward black (fraction 0.0→1.0)
    → calls Sys_SetScreenFade internally for each display
    → applies via CGSetDisplayTransferByTable

  [mode change happens here]

  Sys_UnfadeScreens()
    → calls Sys_UnfadeScreen() for each display
    → restores stored gamma via CGSetDisplayTransferByTable

Shutdown:
  Sys_ReleaseAllDisplays()
    → calls CGReleaseAllDisplays()
    → returns control to window manager
```

## Learning Notes

- **Era-Specific Approach**: This pattern is typical of 2005–2010 3D engines. Modern engines (Unreal, Unity, Godot) use shader-based post-process gamma correction instead, avoiding direct hardware manipulation (safer, more portable, respects OS accessibility settings).
- **macOS Specificity**: The implementation in `macosx_display.m` would use Objective-C and Core Graphics; the `unix/` and `win32/` equivalents would use XRandR/xgamma or WGL/DirectDraw respectively. The public interface here is platform-agnostic, but the implementation is deeply macOS-specific.
- **Competitive Gaming Priority**: The fade/unfade and direct gamma access show Q3A's focus on latency-critical competitive play—every microsecond matters.
- **Global State Management**: Reliance on `glw_state` global means display management is tightly coupled to the GLimp lifecycle; no multi-context safety.

## Potential Issues

1. **No Error Handling**: All functions return `void`; if `CGSetDisplayTransferByTable` fails (e.g., insufficient privileges on newer macOS), there's no way to signal failure. The render path silently fails.
2. **Multi-Display Edge Cases**: `Sys_GetMatchingDisplayMode()` returns a single `NSDictionary` for the best match, but multi-monitor fullscreen spanning might not be properly handled.
3. **macOS Version Drift**: Direct gamma ramp access via Core Graphics has become increasingly restricted on newer macOS versions (10.15+); this code likely has bitrot on modern hardware/OS combinations.
4. **No Gamma Table Caching**: `Sys_GetGammaTable` reads from hardware each time; repeated reads during rapid mode changes could be slow or unreliable.
