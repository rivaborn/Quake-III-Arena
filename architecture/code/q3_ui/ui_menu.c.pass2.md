# code/q3_ui/ui_menu.c — Enhanced Analysis

## Architectural Role

This file is the entry point and navigation hub of the q3_ui VM module, serving as the sole implementation of the main menu that users encounter at engine startup and after returning from any sub-menu. It bridges three critical engine-level concerns: CD key validation (security boundary), error overlay dispatch (fault handling), and the top-level menu stack management. Unlike typical UI screens which are ephemeral menu items, `UI_MainMenu` resets the entire menu stack (`uis.menusp = 0`), acting as a "home" state that all other menus return to.

## Key Cross-References

### Incoming (who depends on this file)
- **Client loop** (`code/client/cl_ui.c`): calls `VM_Call(uivm, UI_INIT)` which invokes `vmMain` in `ui_main.c`, which then calls `UI_MainMenu()` during initial UI setup or after menu stack exhaustion
- **Engine/game code**: `trap_Cvar_Set("com_errorMessage", ...)` writes engine error messages that this file reads and displays
- **Other UI screens** (`ui_*.c` files in q3_ui): all return here via `UI_MainMenu()` call after user selects back/cancel
- **ui_main.c (`UI_PushMenu` macro)**: menu framework dispatcher that routes keyboard input to `Main_MenuEvent`

### Outgoing (what this file depends on)
- **Renderer subsystem** (`code/renderer/tr_main.c`): via `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_RenderScene`, `trap_R_RegisterModel` — full 3D scene pipeline
- **Menu framework** (`code/q3_ui/ui_qmenu.c`): `Menu_Draw`, `Menu_AddItem` for generic item lifecycle; `color_red`, `menu_text_color`, `menu_null_sound` constants
- **UI atom drawing** (`code/q3_ui/ui_atoms.c`): `UI_DrawProportionalString_AutoWrapped`, `UI_DrawProportionalString`, `UI_DrawString`, `UI_AdjustFrom640` for 2D text rendering
- **Filesystem** via traps: `trap_FS_GetFileList` for mod detection (Team Arena check)
- **CD key validation** (`code/qcommon/cmd.c`): `trap_VerifyCDKey` syscall validates against engine's keyfile
- **Cvar system** (`code/qcommon/cvar.c`): reads `com_errorMessage`, `ui_cdkeychecked`; writes `sv_killserver`, `fs_game`
- **Console command execution**: `trap_Cmd_ExecuteText` to trigger `vid_restart` when switching mods

## Design Patterns & Rationale

**Dual-mode draw function** (`Main_MenuDraw` handles both main menu and error overlay): Avoids code duplication by factoring 3D banner rendering into one function reused by two menu states. The `if (strlen(s_errorMessage.errorMessage))` branch gates which content (error text vs. menu items) is rendered—a pragmatic pattern for engine-level error interception.

**State zeroing on entry** (`memset(&s_main, 0, ...)`): Each `UI_MainMenu()` call wipes prior state, ensuring menus are idempotent. This is critical for correct behavior when returning from sub-menus or error conditions—no stale widget state persists.

**Conditional menu item assembly**: Team Arena is added only if the mod directory is detected at runtime. This avoids hard-coding mod dependencies and allows the same binary to run on base Q3A or with Team Arena installed, reflecting Q3A's modular design philosophy.

**Early CD key check**: Validation happens before any menu interaction, blocking entry to the full menu if the key is invalid. This is a security gate—once `ui_cdkeychecked` is set, subsequent calls skip re-validation, optimizing for the common case where the key is already confirmed.

**Menu stack reset instead of push**: `uis.menusp = 0` before `UI_PushMenu` clears the entire prior stack. This is intentional—it prevents accumulation of stale menus in the stack and ensures the main menu is always the sole item in the stack, making it the true "home" state.

## Data Flow Through This File

1. **Entry**: `UI_MainMenu()` called by client during startup or when menu stack is exhausted
2. **Validation phase**: CD key check (if not demo and not already verified) → redirect to CD key menu or continue
3. **State initialization**: Zero `s_main` and `s_errorMessage`; load banner model via `MainMenu_Cache()`
4. **Cvar check**: Read `com_errorMessage` and decide: error overlay mode or normal menu mode
5. **Widget assembly**: If normal mode, build 8 menu items (or 9 with Team Arena) with callbacks pointing to `Main_MenuEvent`
6. **Push & display**: Set key catcher to UI, reset menu stack depth, push `s_main.menu` onto stack
7. **Per-frame rendering**: Each frame, `Main_MenuDraw` is called, which:
   - Sets up a viewport (640×120 virtual) excluding the world model
   - Renders a rotating 3D banner (via sine-modulated yaw angle)
   - Renders either error text or the menu items
   - Draws copyright/demo watermark strings
8. **Event dispatch**: When user activates a menu item, `Main_MenuEvent` looks up the ID and calls the appropriate sub-menu function (e.g., `UI_SPLevelMenu()`)
9. **Exit flow**: The exit button triggers a confirmation dialog (`UI_ConfirmMenu`) which, on confirmation, calls `MainMenu_ExitAction` → `UI_CreditMenu()` → engine shutdown

## Learning Notes

**3D rendering inside UI VM**: This file is instructive for understanding that the UI VM is not purely 2D—it has full access to the renderer's 3D pipeline via `trap_R_*` syscalls. This enables rich interactive experiences (like animated model previews elsewhere in the UI) from the VM layer.

**Error overlay as a non-blocking fault handler**: The engine can write a human-readable error message to the `com_errorMessage` cvar, and the UI will automatically display it as an overlay on the next `UI_MainMenu()` call, without crashing or requiring engine restart. This is a soft fault recovery pattern: errors are logged, the user is informed, and execution continues.

**Mod detection and fs_game switching**: The Team Arena check demonstrates how Q3A handles mod discovery at runtime—by scanning the filesystem for known mod directories. Switching `fs_game` and calling `vid_restart` shows how a pure-filesystem-based mod system works without a registry or installer metadata.

**Menu framework reuse**: The generic `menutext_s` / `menuframework_s` types and `Menu_AddItem` / `Menu_Draw` functions are reusable abstractions used across all q3_ui menu screens. This is a precursor to modern UI component architectures but implemented procedurally in 1999.

**Early-bind rendering state**: Comments note that FOV wobble was removed at designer request—the code was optimized after initial implementation, showing iterative tuning of visual feedback. The fixed 640×120 viewport and 60/19.6875 FOV are hand-tuned magic numbers for the banner's visual framing.

## Potential Issues

- **CD key check skipped in demo mode**: A security gate is disabled for demo builds (`!uis.demoversion` check). This is intentional for distribution/testing but worth flagging as a sensitive code path.
- **Hard-coded mod directory name**: The Team Arena check is hard-coded to look for `"missionpack"`. If the directory name changes, the check silently fails and the menu item never appears, with no diagnostic output.
- **4096-byte error message buffer**: The fixed-size `s_errorMessage.errorMessage[4096]` could overflow if the engine writes a larger message. The `trap_Cvar_VariableStringBuffer` call should bound this, but it's worth auditing the engine-side cvar write.
