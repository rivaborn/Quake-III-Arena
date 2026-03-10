# code/q3_ui/ui_signup.c — Enhanced Analysis

## Architectural Role

This file implements the GRank online account registration UI—a single modal menu within the q3_ui VM's larger menu system. It is a thin presentation layer that collects signup form data (name, password, email) and delegates account creation to the engine's rankings subsystem via a GRank-specific syscall. The file demonstrates how the UI VM layer isolates menu logic behind the `trap_*` syscall boundary and participates in the modal menu stack (`UI_PushMenu`/`UI_PopMenu`) that underpins Quake III's entire menu navigation architecture.

## Key Cross-References

### Incoming (who depends on this file)
- **Callers of `UI_SignupMenu`**: Likely `ui_rankings.c` or a login-flow menu (not explicitly visible in xref, but inferred from GRank workflow)
- **Menu stack**: Called via `UI_PushMenu(&s_signup.menu)`, which is managed by `ui_atoms.c` and the central menu framework

### Outgoing (what this file depends on)
- **Menu framework**: `Menu_AddItem`, `menuframework_s`, `menutext_s`, `menufield_s`, `menubitmap_s` types (defined in `ui_qmenu.c` / `ui_local.h`)
- **UI navigation**: `UI_PushMenu`, `UI_PopMenu`, `UI_ForceMenuOff` (from `ui_atoms.c`)
- **Ownerdraw callbacks**: `Rankings_DrawName`, `Rankings_DrawPassword`, `Rankings_DrawText` (from `ui_rankings.c`)
- **Engine traps**:
  - `trap_CL_UI_RankUserCreate` — GRank-specific syscall; likely routes to `SV_RankCreateAccount` or similar in `code/server/sv_rankings.c`
  - `trap_Cvar_VariableValue` — read `client_status` cvar to check eligibility
  - `trap_R_RegisterShaderNoMip` — preload decorative frame bitmap
- **Global constants**: `colorRed`, `colorMdGrey` (from `ui_local.h` / `q_shared.c`)

## Design Patterns & Rationale

**Modal Menu Stack Pattern**: Like all q3_ui menus, this file uses a push-pop modal stack. The menu remains active until explicitly popped or `UI_ForceMenuOff` is called. This is idiomatic to era-2000s game UI architectures and contrasts with modern immediate-mode UI systems.

**Widget Declarative Initialization**: `Signup_MenuInit` mirrors a UI builder approach—each widget's properties (position, flags, color, callback) are set explicitly before registration. This is verbose compared to data-driven approaches (like `code/ui` scripts) but runtime-efficient and type-safe.

**Conditional Availability**: The `client_status` cvar check in `Signup_MenuInit` disables all input fields if the player is not eligible (i.e., not newly registered). This enforcement is UI-side; the server double-checks authorization on the `trap_CL_UI_RankUserCreate` syscall.

**Callback Dispatch**: `Signup_MenuEvent` is a single event handler delegating on widget ID. This is simpler than per-widget callbacks and concentrates logic, though it couples multiple concerns.

**Dead Code Evolution**: Commented-out cvar-setting and sprintf-based command paths suggest the GRank integration evolved from a command-line API (`"cmd rank_create ..."`) to a direct syscall. This is a common refactoring pattern in long-lived codebases.

## Data Flow Through This File

```
1. UI_SignupMenu()
   ↓
2. Signup_MenuInit()
   ├─ Signup_Cache() → trap_R_RegisterShaderNoMip (preload frame)
   ├─ Create/configure all widgets (name, password, email, buttons)
   ├─ Read client_status cvar → conditionally deactivate fields
   └─ Menu_AddItem × 11 → register all widgets with framework
   ↓
3. UI_PushMenu(&s_signup.menu)
   → Menu now active on stack; events dispatched to it each frame
   ↓
4. User interacts (key/mouse) → framework routes to Signup_MenuEvent
   ├─ ID_SIGNUP: 
   │  ├─ strcmp(password_box, again_box) → validate match
   │  └─ trap_CL_UI_RankUserCreate(...) → syscall to rankings backend
   │  └─ UI_ForceMenuOff() → close all menus
   └─ ID_CANCEL:
      └─ UI_PopMenu() → return to previous menu
```

**Input validation occurs entirely in `Signup_MenuEvent`**: only password-match is checked. The email and name have no length or format validation at UI layer; the server is expected to validate on the `trap_CL_UI_RankUserCreate` syscall.

## Learning Notes

**Q3A UI Architecture Era (2000)**: This file exemplifies the modal-menu-stack + widget-callback model used throughout the id codebase. Modern engines (Unity, Unreal) moved to scene graphs or retained-mode UI; Q3A's approach is procedural and straightforward but inflexible—reordering widgets or adding conditional logic requires code changes, not configuration.

**Syscall Boundary**: `trap_CL_UI_RankUserCreate` is a GRank-specific extension added after base Q3A's release (likely for the online rankings feature added in a patch or MissionPack). It demonstrates how the engine/VM boundary accommodates new subsystems without recompiling the base engine—new syscall opcodes are simply added to the trap dispatch table.

**Minimal Client-Side Validation**: The UI only validates password match; email format, name uniqueness, and password strength are all server-side concerns. This reflects the security model of early 2000s games where the server was trusted and the client was not.

**Ownerdraw Indirection**: The `Rankings_Draw*` callbacks (set via `generic.ownerdraw`) allow special rendering (e.g., password masking) without requiring the text field widget to know about it. This is a form of the Strategy pattern.

**Unused Declarations** (`s_signup_menu`, `s_signup_signup`, `s_signup_cancel`): These were likely declarations intended for a refactored version that moved to a data-driven menu system or a different initialization path. They're dead code and could be removed.

## Potential Issues

1. **Silent Password Mismatch** (flagged by `GRANK_FIXME` comment): When passwords don't match, the function silently `break`s without setting an error message or user feedback. A player might not realize why their signup failed.

2. **No Input Validation**: Name and email fields accept any input; the server must validate. If the server has strict rules, the UI should mirror them for better UX.

3. **Hardcoded Layout**: Pixel positions and field widths are hardcoded; no abstraction for different resolutions or UI scaling (though Q3A rendered in a fixed 640×480 virtual space, so this was acceptable for the era).

4. **Truncated String Risk**: The commented-out `sprintf` path suggests awareness of buffer overflow risk, but the current code path does not show explicit bounds checking on string copies into `s_signup.*_box.field.buffer`.

5. **Missing Logout/Already-Logged-In Workflow**: The eligibility check (`client_status != QGR_STATUS_NEW`) prevents signup if the player is already ranked, but there's no UI feedback explaining why signup is disabled—just greyed-out buttons.

6. **No Rate Limiting**: Multiple rapid signups are not prevented at the UI layer; the server must throttle/deduplicate.
