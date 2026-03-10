# code/macosx/macosx_display.h

## File Purpose
Public interface header for macOS display management in Quake III Arena. It declares functions for querying display modes, managing hardware gamma ramp tables, and fading/unfading displays during mode switches.

## Core Responsibilities
- Declare the display mode query function (`Sys_GetMatchingDisplayMode`)
- Declare gamma table storage and retrieval functions
- Declare per-display and all-display fade/unfade operations
- Declare display release cleanup

## Key Types / Data Structures
None defined here; depends on types from included headers.

| Name | Kind | Purpose |
|------|------|---------|
| `glwgamma_t` | struct (defined in `macosx_local.h`) | Holds a CGDirectDisplayID and per-channel gamma ramp arrays (red, green, blue) |
| `NSDictionary` | Objective-C class (forward declared) | Used as return type for `Sys_GetMatchingDisplayMode` to represent a Core Graphics display mode dictionary |

## Global / File-Static State
None declared in this file.

## Key Functions / Methods

### Sys_GetMatchingDisplayMode
- **Signature:** `NSDictionary *Sys_GetMatchingDisplayMode(qboolean allowStretchedModes)`
- **Purpose:** Queries Core Graphics for a display mode matching the current video settings (resolution, refresh rate, bit depth).
- **Inputs:** `allowStretchedModes` — whether non-native aspect ratios are acceptable.
- **Outputs/Return:** Pointer to an `NSDictionary` describing the matched display mode, or `nil` on failure.
- **Side effects:** None inferable from declaration.
- **Calls:** Not inferable from this file.
- **Notes:** Uses Objective-C/CoreGraphics display mode enumeration internally (defined in `macosx_display.m`).

### Sys_StoreGammaTables
- **Signature:** `void Sys_StoreGammaTables()`
- **Purpose:** Saves the current hardware gamma ramps for all active displays into `glw_state.originalDisplayGammaTables` for later restoration.
- **Inputs:** None.
- **Outputs/Return:** void.
- **Side effects:** Writes to `glw_state` global.
- **Calls:** Not inferable from this file.

### Sys_GetGammaTable
- **Signature:** `void Sys_GetGammaTable(glwgamma_t *table)`
- **Purpose:** Reads the current hardware gamma ramp from a specific display into the provided `glwgamma_t` structure.
- **Inputs:** `table` — pointer to a `glwgamma_t` with `.display` set to the target display ID.
- **Outputs/Return:** Fills `table->red/green/blue` arrays in-place.
- **Side effects:** None beyond writing to `*table`.
- **Calls:** Not inferable from this file.

### Sys_SetScreenFade
- **Signature:** `void Sys_SetScreenFade(glwgamma_t *table, float fraction)`
- **Purpose:** Applies a partial fade to a display by interpolating its gamma ramp toward black by `fraction` (0.0 = full brightness, 1.0 = full black).
- **Inputs:** `table` — source gamma table; `fraction` — blend factor.
- **Outputs/Return:** void.
- **Side effects:** Calls `CGSetDisplayTransferByTable` or equivalent; modifies hardware gamma.
- **Calls:** Not inferable from this file.

### Sys_FadeScreens / Sys_FadeScreen
- **Signature:** `void Sys_FadeScreens()` / `void Sys_FadeScreen(CGDirectDisplayID display)`
- **Purpose:** Fade all displays or a single display to black (used before fullscreen mode switches).
- **Side effects:** Modifies hardware gamma state.

### Sys_UnfadeScreens / Sys_UnfadeScreen
- **Signature:** `void Sys_UnfadeScreens()` / `void Sys_UnfadeScreen(CGDirectDisplayID display, glwgamma_t *table)`
- **Purpose:** Restore all displays or a single display to stored gamma after a mode switch.
- **Side effects:** Restores hardware gamma from `table`.

### Sys_ReleaseAllDisplays
- **Signature:** `void Sys_ReleaseAllDisplays()`
- **Purpose:** Releases capture of all CGDirectDisplayIDs, returning control to the window manager on shutdown.
- **Side effects:** Calls `CGReleaseAllDisplays()` or equivalent.

## Control Flow Notes
These functions are called during the macOS GLimp lifecycle: `Sys_StoreGammaTables` is called at init; fade/unfade pairs bracket fullscreen mode transitions in `GLimp_Init`/`GLimp_Shutdown`; `Sys_ReleaseAllDisplays` is called at shutdown. Not involved in the per-frame render path.

## External Dependencies
- `tr_local.h` — renderer types (`qboolean`, `glconfig_t`, etc.)
- `macosx_local.h` — `glwgamma_t`, `glwstate_t`, `CGDirectDisplayID`, `glw_state` global
- `ApplicationServices/ApplicationServices.h` (via `macosx_local.h`) — `CGDirectDisplayID`, Core Graphics display API
- Implementations defined in `macosx_display.m` (not visible here)
