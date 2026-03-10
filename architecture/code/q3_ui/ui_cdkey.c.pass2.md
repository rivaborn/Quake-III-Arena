# code/q3_ui/ui_cdkey.c — Enhanced Analysis

## Architectural Role

This file implements the CD key validation dialog within the legacy Q3A UI VM (`code/q3_ui`). It bridges the client-side UI framework to engine-level CD key storage and verification syscalls (`trap_SetCDKey`, `trap_GetCDKey`, `trap_VerifyCDKey`), providing a modal full-screen menu for manual key entry. The module exemplifies the Q3A DRM era: before Steam/EGS, CD keys were lightweight anti-copying tokens managed by the engine itself rather than external account systems.

## Key Cross-References

### Incoming (who depends on this file)
- `code/q3_ui/ui_main.c` (and other menu activation paths) invoke `UI_CDKeyMenu()` / `UI_CDKeyMenu_f()` to push this modal onto the menu stack
- `UI_CDKeyMenu_Cache()` is called during UI VM initialization to pre-load shader assets

### Outgoing (what this file depends on)
- **Engine syscalls:**
  - `trap_SetCDKey`, `trap_GetCDKey`, `trap_VerifyCDKey` — CD key persistence and cryptographic validation (server-side; engine calls into game DLL)
  - `trap_Cvar_Set` — sets `ui_cdkeychecked` flag for controller state
  - `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode` — renderer and input syscalls
- **UI framework (from `code/q3_ui/`):**
  - `UI_PushMenu`, `UI_PopMenu` — menu stack management
  - `Menu_AddItem` — widget registration
  - `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_DrawProportionalString` — 2D drawing
  - `uis.menusp` (global `uiStatic_t`) — menu stack depth check for Back button visibility
  - Color globals (`color_yellow`, `color_white`, `color_orange`, `color_red`, `listbar_color`)

## Design Patterns & Rationale

1. **Menu Framework Abstraction:**  
   Uses declarative widget composition (`menuframework_s` + array of `menucommon_s` subclasses) rather than imperative draw loops. Enables consistent key/mouse routing and rendering across all menus without per-menu input dispatch code.

2. **Owner-Draw Callback Pattern:**  
   `UI_CDKeyMenu_DrawKey()` overrides the default field rendering to display live validation feedback (yellow = incomplete, red = invalid, white = valid). This is more UX-responsive than post-submission errors.

3. **Client-Side Pre-Validation Pipeline:**  
   `UI_CDKeyMenu_PreValidateKey()` performs lightweight format validation (length + charset) **before** any server round-trip. Reduces latency and provides immediate UX feedback; actual cryptographic verification is delegated to `trap_VerifyCDKey()` (engine-side) to prevent client-side key spoofing.

4. **Conditional Menu Stack Integration:**  
   Back button is omitted when `uis.menusp == 0` (no parent menu). This prevents soft-lock if the CD Key menu is the root menu in some scenarios (e.g., first-time launch or certain game modes).

## Data Flow Through This File

1. **Initialization Phase** (`UI_CDKeyMenu_Init`):
   - Retrieve persisted key via `trap_GetCDKey()`
   - Verify it via `trap_VerifyCDKey()` (async?); if invalid, zero the buffer
   - Layout all widgets with fixed screen coordinates (640×480 virtual space)
   - Conditionally add Back button based on menu stack state

2. **Runtime Phase** (per-frame):
   - `Menu_Draw()` invokes `UI_CDKeyMenu_DrawKey()` callback each frame
   - Displays buffer contents + cursor + real-time validation status
   - Input events (`Menu_DefaultKey`) route to `UI_CDKeyMenu_Event()` on button activation only

3. **Exit Phase** (on Accept or Back):
   - Accept: call `trap_SetCDKey(buffer)` if non-empty, then pop menu
   - Back: pop menu without saving
   - Menu stack unwind resumes previous menu

## Learning Notes

- **Era-Specific DRM:** CD keys as 16-char tokens with restricted charset (`2 3 7 a b c d g h j l p r s t w`) represent Q3's original anti-copying scheme. Modern engines use account-based authentication (Steam, Epic).
- **Synchronous UI, Async Validation:** The engine's `trap_VerifyCDKey()` is likely synchronous (quick local check), but the architecture allows for async server-side validation if needed. The engine could defer the check until connect time.
- **Full-Screen Modal Dialog Pattern:** Unlike modern UIs (which layer modals), Q3's menu stack approach replaces the entire previous menu context. This simplifies state management but risks UX confusion if the player forgets where they came from (mitigated by conditional Back button).
- **Virtual Resolution Hardcoding:** All menu coordinates are hard-coded to 640×480, with scaling left to renderer. Modern engines would use layout managers or relative positioning.

## Potential Issues

1. **Buffer Overflow Risk (Low):**  
   `UI_CDKeyMenu_DrawKey()` draws the entire buffer without explicit bounds checking, but `widthInChars == 16` and `maxchars == 16` constraints in initialization should prevent overflow. However, if the buffer is not null-terminated, `strlen()` in `UI_CDKeyMenu_PreValidateKey()` could read past bounds.

2. **Hard-Coded Validation Charset:**  
   The allowed characters are baked into `UI_CDKeyMenu_PreValidateKey()`. If expansion packs (e.g., Team Arena) use different CD key formats, this function would reject them. No mechanism for runtime charset configuration.

3. **No Rate Limiting on `trap_SetCDKey()`:**  
   Rapid key submissions are not throttled client-side. The engine should validate server-side to prevent brute-force attacks, but the UI does not protect against malicious input patterns.
