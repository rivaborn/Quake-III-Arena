# code/q3_ui/ui_login.c — Enhanced Analysis

## Architectural Role

This file implements a single modal dialog within the q3_ui VM module's menu stack. It serves the **GRank online rankings system** by collecting and submitting player credentials (name/password) to the client engine's rankings backend. The menu is not persistent; it pushes itself onto the global UI stack when credentials are needed, then pops off after login succeeds or the user cancels. It represents a typical leaf-level menu in Q3A's hierarchical, stack-based UI architecture.

## Key Cross-References

### Incoming (who depends on this file)
- **q3_ui vmMain dispatch** (in `ui_main.c` or `ui_atoms.c`) — routes UI menu initialization calls; likely invokes `UI_LoginMenu()` when the rankings system signals a login prompt is needed
- **GRank/rankings subsystem** (client-side, in `code/client/`) — triggers login menu display when player attempts to submit rank data without valid credentials

### Outgoing (what this file depends on)
- **q3_ui menu framework** (`ui_qmenu.c`) — `Menu_AddItem()` registers all 7 widgets with the menu container; the framework handles per-frame draw, focus management, and generic key/mouse dispatch
- **q3_ui menu stack** (`ui_atoms.c`) — `UI_PushMenu()` adds this menu to the global stack; `UI_PopMenu()` / `UI_ForceMenuOff()` remove it after user action
- **Rankings owner-draw callbacks** (`ui_rankings.c`) — `Rankings_DrawName` and `Rankings_DrawPassword` handle field rendering (latter masks password input visually)
- **Client engine rankings syscall** — `trap_CL_UI_RankUserLogin()` (defined in `ui_syscalls.c` / raw engine boundary) submits credentials to the rankings backend
- **Renderer shader cache** — `trap_R_RegisterShaderNoMip()` preloads the frame decoration bitmap during `Login_Cache()`

## Design Patterns & Rationale

**Widget Composition (Hierarchical)**
- Single `login_t` struct aggregates all UI elements (frame, labels, input fields, buttons). This mirrors the physical layout and simplifies initialization ordering. The struct hierarchy (`login_t` → `menuframework_s` → individual controls) is idiomatic to Q3A's late-1990s C-based UI architecture.

**Menu Stack Pattern**
- Q3A's UI is entirely stack-based: each menu is a modal overlay on a first-in-last-out stack. `UI_LoginMenu()` is the entry point; it initializes and pushes onto `uis.stack`. The engine's main loop automatically dispatches input to the topmost menu, avoiding per-menu custom key bindings. This is both a blessing (consistency) and a limitation (no overlapping modals).

**Event-Driven Callback Model**
- Each widget has a `generic.callback` function pointer and an `id` tag. The framework calls the callback with `(ptr, event)` when the widget is activated. This is C-style polymorphism: `((menucommon_s*)ptr)->id` is checked to route the event. Pre-C++, pre-templates, it was the standard pattern.

**Syscall Abstraction for VM Boundary**
- `trap_CL_UI_RankUserLogin()` is not declared in any header; it's a raw syscall wrapper. The q3_ui VM has no direct access to engine state — all communication flows through versioned trap ABIs. This sandbox isolation was ahead of its time (1999) and shows careful architecture even at the VM boundary.

## Data Flow Through This File

1. **Trigger**: Rankings system detects missing/expired credentials; calls into UI VM to show login dialog.
2. **Init Phase**: `UI_LoginMenu()` → `Login_MenuInit()` (zero-initializes `s_login`, caches shader, registers 7 menu items with the framework).
3. **Runtime Loop**: Menu framework handles display and input dispatch. User can focus/blur fields, type, or click buttons.
4. **User Action**:
   - **LOGIN**: `Login_MenuEvent(ptr, QM_ACTIVATED)` extracts buffers from `s_login.name_box` and `s_login.password_box`, calls `trap_CL_UI_RankUserLogin()` with plaintext credentials, then `UI_ForceMenuOff()` closes the menu and returns control to the game.
   - **CANCEL**: Pops the menu with `UI_PopMenu()`.
5. **Result**: Client engine's rankings system receives credentials (or user abort), proceeds with auth or stays in game.

## Learning Notes

**Q3A UI Philosophy**
- The UI is declarative and hierarchical, but entirely procedural to initialize (no data files for base-q3_ui). Each menu hand-codes its widget tree, positions (pixel-based), and callbacks.
- Modern engines (even id's later work, like Doom 3) moved toward declarative/scripted UIs; Q3A predates that shift.

**Idiomatic Patterns of the Era**
- `menucommon_s *ptr` cast-then-access is pre-template polymorphism. Modern C++ would use inheritance or virtual methods; this code uses void pointers and manual dispatch.
- Event filtering (`if (event != QM_ACTIVATED) return;`) is explicit and brute-force. Other events (focus, blur, key presses) are silently dropped.
- The `y` counter for layout is manual; no constraint solver or flex box. Hard-coded pixel positions make reflow inflexible but keep the code simple.

**Dead Code & Refactoring Artifacts**
- `s_login_menu`, `s_login_login`, `s_login_cancel` are declared at file scope but never used. The struct `login_t` supersedes them. This suggests an earlier version of the code used a different menu pattern, and the old globals were left behind (common in large codebases).
- Commented `trap_Cvar_Set( "rank_name", ... )` hints that credentials were once cached as CVars (a security antipattern). Direct syscall passing (current code) is better.

**Owner-Draw Callback Pattern**
- The password field uses `Rankings_DrawPassword` via `menufield_s.generic.ownerdraw`. This allows the buffer to remain unmasked in memory while the display layer renders masked asterisks. Modern UIs often do this with a separate masked-string representation; Q3A reuses the field's internal buffer and delegates rendering to a callback. Clever, but requires careful coordination.

**Cross-Module Sync**
- The credentials flow: UI VM → engine syscall → rankings backend (out of process or async). There's no error callback; if login fails, the user sees the menu disappear with no feedback. The rankings system must handle async responses and re-show the menu if needed.

## Potential Issues

1. **Dead Variables**: `s_login_menu`, `s_login_login`, `s_login_cancel` are never populated or read. They can be safely deleted to reduce confusion.

2. **Plaintext Credentials in Memory**: The password buffer is a plain C array in stack memory. No explicit zeroing after use. Modern practice would `memset(..., 0, ...)` the field before freeing. However, this is a minor concern in a single-player game; the bigger risk is over the network (mitigated by the syscall boundary).

3. **No Error Handling**: If `trap_CL_UI_RankUserLogin()` fails (network error, server down), the menu silently closes with no user feedback. The engine must handle retries and re-display internally.

4. **Commented-Out Cvar Code**: The commented `trap_Cvar_Set` calls are confusing. Either delete them or add a comment explaining why they were removed (e.g., "Credentials should not be persisted as CVars for security").

5. **Incomplete Layout y Counter**: The variable `y` is incremented but its final value is unused. No functional issue, but suggests incomplete refactoring (maybe once used for dynamic layout bounds checking).
