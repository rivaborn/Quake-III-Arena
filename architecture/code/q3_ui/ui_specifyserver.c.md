# code/q3_ui/ui_specifyserver.c

## File Purpose
Implements the "Specify Server" UI menu, allowing players to manually enter a server IP address and port number to connect to directly. It is a simple two-field input form within the Q3 legacy UI module.

## Core Responsibilities
- Define and initialize all menu items (banner, decorative frames, address/port fields, go/back buttons)
- Handle user activation events for "Go" (connect) and "Back" (pop menu) buttons
- Preload/cache all required bitmap art assets via the renderer
- Build and dispatch the `connect <address>:<port>` command string to the engine
- Push the assembled menu onto the active UI menu stack

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `specifyserver_t` | struct | Aggregates all menu widgets for the Specify Server screen (framework, banner, frames, input fields, buttons) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_specifyserver` | `specifyserver_t` | static (file) | Singleton instance holding all menu state for this screen |
| `specifyserver_artlist` | `char*[]` | static (file) | NULL-terminated list of shader paths to preload via `SpecifyServer_Cache` |

## Key Functions / Methods

### SpecifyServer_Event
- **Signature:** `static void SpecifyServer_Event( void* ptr, int event )`
- **Purpose:** Callback for interactive menu items; handles connect and back actions.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is acted upon)
- **Outputs/Return:** void
- **Side effects:** On `ID_SPECIFYSERVERGO`: appends a `connect <host>[:<port>]\n` command to the engine command buffer. On `ID_SPECIFYSERVERBACK`: pops the current menu.
- **Calls:** `Com_sprintf`, `strcpy`, `strlen`, `trap_Cmd_ExecuteText`, `va`, `UI_PopMenu`
- **Notes:** No-ops silently if the domain field is empty. Port is appended only when non-empty. Uses `EXEC_APPEND`, consistent with the trap header's "don't use EXEC_NOW" note.

### SpecifyServer_MenuInit
- **Signature:** `void SpecifyServer_MenuInit( void )`
- **Purpose:** Zeroes the global state, calls the cache function, then initializes every widget's position, flags, and default values before registering them with the menu framework.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes into `s_specifyserver`; pre-fills the port field with `"27960"` (default Q3 port); calls `SpecifyServer_Cache`.
- **Calls:** `memset`, `SpecifyServer_Cache`, `Menu_AddItem`, `Com_sprintf`
- **Notes:** Port field is `QMF_NUMBERSONLY`, max 5 chars. Domain field allows up to 80 chars. Layout is hard-coded to 640×480 virtual resolution.

### SpecifyServer_Cache
- **Signature:** `void SpecifyServer_Cache( void )`
- **Purpose:** Iterates `specifyserver_artlist` and registers each shader with the renderer so assets are resident before the menu is drawn.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for each art entry; results (handles) are discarded — registration is the side effect.
- **Calls:** `trap_R_RegisterShaderNoMip`
- **Notes:** Handles are not stored; the bitmap widgets reference assets by name string, and the menu framework resolves them on draw.

### UI_SpecifyServerMenu
- **Signature:** `void UI_SpecifyServerMenu( void )`
- **Purpose:** Public entry point; initializes the menu and pushes it onto the UI stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `SpecifyServer_MenuInit` and `UI_PushMenu`.
- **Calls:** `SpecifyServer_MenuInit`, `UI_PushMenu`

## Control Flow Notes
This file has no frame/update loop. It is purely event-driven: `UI_SpecifyServerMenu` is called once to show the menu; `SpecifyServer_Event` fires when the user activates a button. Teardown happens via `UI_PopMenu` in the back handler or implicitly when the engine processes the `connect` command.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types/macros)
- **Defined elsewhere:** `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `Com_sprintf`, `va`, `color_white` — all provided by the broader Q3 UI/engine runtime
