# code/client/cl_console.c

## File Purpose
Implements the in-game developer console for Quake III Arena, handling text buffering, scrollback, notify overlays, animated slide-in/out drawing, and chat message input modes.

## Core Responsibilities
- Maintain a circular text buffer (`con.text`) for scrollback history
- Handle line wrapping, word wrapping, and color-coded character storage
- Animate console slide open/close via `displayFrac`/`finalFrac` interpolation
- Render the solid console panel, scrollback arrows, version string, and input prompt
- Render transparent notify lines (recent messages) over the game view
- Manage chat input modes (global, team, crosshair target, last attacker)
- Register console-related commands (`toggleconsole`, `clear`, `condump`, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `console_t` | struct | Full console state: text buffer, scroll position, display fraction, notify timestamps, color |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `con` | `console_t` | global | Singleton console state |
| `con_conspeed` | `cvar_t *` | global | CVar controlling console slide speed (`scr_conspeed`) |
| `con_notifytime` | `cvar_t *` | global | CVar for how long notify lines remain visible |
| `console_color` | `vec4_t` | global | Default console text color (white) |
| `g_console_field_width` | `int` | global | Character width of the console input field (default 78) |

## Key Functions / Methods

### Con_Init
- **Signature:** `void Con_Init(void)`
- **Purpose:** Bootstrap the console subsystem.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Registers CVars `con_notifytime` / `scr_conspeed`; clears input field and history; registers 7 console commands.
- **Calls:** `Cvar_Get`, `Field_Clear`, `Cmd_AddCommand`
- **Notes:** Called once during client startup; does not initialize `con.initialized` — that is deferred to first print.

### CL_ConsolePrint
- **Signature:** `void CL_ConsolePrint(char *txt)`
- **Purpose:** Primary entry point for all text output to the console buffer. Handles word-wrap, color codes, `[skipnotify]` prefix, and notify timestamp updates.
- **Inputs:** `txt` — raw string, may contain color escape sequences or `[skipnotify]` prefix.
- **Outputs/Return:** None
- **Side effects:** Writes encoded `short` values (color<<8 | char) into `con.text`; advances `con.x` / `con.current`; calls `Con_Linefeed`; updates `con.times[]`.
- **Calls:** `Q_strncmp`, `Con_CheckResize`, `Q_IsColorString`, `Con_Linefeed`
- **Notes:** Lazy-initializes `con` on first call. `[skipnotify]` suppresses the transparent overlay timestamp for that line.

### Con_CheckResize
- **Signature:** `void Con_CheckResize(void)`
- **Purpose:** Recomputes `linewidth`/`totallines` when the video resolution changes; reformats existing buffer content into the new layout.
- **Inputs:** None (reads `SCREEN_WIDTH`, `SMALLCHAR_WIDTH`)
- **Outputs/Return:** None
- **Side effects:** Allocates stack buffer `tbuf[CON_TEXTSIZE]` (MAC_STATIC), rewrites `con.text`.
- **Calls:** `Com_Memcpy`, `Con_ClearNotify`
- **Notes:** Called at start of every `Con_DrawConsole` frame to catch vid-mode changes.

### Con_DrawConsole
- **Signature:** `void Con_DrawConsole(void)`
- **Purpose:** Frame-level draw dispatcher: routes to full-screen solid console, partial solid console, or notify-only overlay.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Triggers rendering calls.
- **Calls:** `Con_CheckResize`, `Con_DrawSolidConsole`, `Con_DrawNotify`

### Con_DrawSolidConsole
- **Signature:** `void Con_DrawSolidConsole(float frac)`
- **Purpose:** Renders the full drop-down console panel: background shader, red separator line, version string, scrollback text, backscroll arrows, and input line.
- **Inputs:** `frac` — fraction of screen height to occupy (0.0–1.0).
- **Outputs/Return:** None
- **Side effects:** Issues multiple renderer calls (`re.SetColor`, `SCR_DrawPic`, `SCR_DrawSmallChar`, etc.).
- **Calls:** `SCR_AdjustFrom640`, `SCR_DrawPic`, `SCR_FillRect`, `re.SetColor`, `SCR_DrawSmallChar`, `Con_DrawInput`, `Con_Linefeed` (indirectly)

### Con_RunConsole
- **Signature:** `void Con_RunConsole(void)`
- **Purpose:** Per-frame animation update: slides `displayFrac` toward `finalFrac` based on `con_conspeed` and `cls.realFrametime`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Mutates `con.displayFrac`, `con.finalFrac`.
- **Calls:** None (reads CVars directly)

### Con_DrawNotify
- **Signature:** `void Con_DrawNotify(void)`
- **Purpose:** Draws the last `NUM_CON_TIMES` (4) recent lines as a transparent HUD overlay, and the active chat input line.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Renderer color/draw calls; reads `cl.snap.ps.pm_type`, `cls.keyCatchers`.
- **Calls:** `re.SetColor`, `SCR_DrawSmallChar`, `SCR_DrawBigString`, `Field_BigDraw`

### Con_Dump_f
- **Signature:** `void Con_Dump_f(void)`
- **Purpose:** Command handler: writes visible console text to a file, stripping trailing spaces.
- **Inputs:** Console command argument 1 = filename.
- **Outputs/Return:** None
- **Side effects:** File I/O (`FS_FOpenFileWrite`, `FS_Write`, `FS_FCloseFile`).
- **Calls:** `Cmd_Argc`, `Cmd_Argv`, `FS_FOpenFileWrite`, `FS_Write`, `FS_FCloseFile`, `Con_Bottom`

### Con_MessageMode_f / _f2 / _f3 / _f4
- **Notes:** Four command handlers setting up different chat targets (global, team, crosshair player, last attacker) by configuring `chat_playerNum`, `chat_team`, and toggling `KEYCATCH_MESSAGE`.

### Con_PageUp / Con_PageDown / Con_Top / Con_Bottom / Con_Close
- **Notes:** Simple navigation helpers mutating `con.display`; `Con_Close` also clears `KEYCATCH_CONSOLE` and resets fractions.

## Control Flow Notes
- **Init:** `Con_Init` called once from `CL_Init`.
- **Per-frame update:** `Con_RunConsole` called each client frame to animate the console.
- **Per-frame draw:** `Con_DrawConsole` called from the screen update path after `Con_RunConsole`.
- **Print path:** All engine text output routes through `CL_ConsolePrint` (called from `Com_Printf` via a function pointer set in `common.c`).

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `keys.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `cl` (`clientActive_t`), `cgvm`, `re` (renderer exports), `g_consoleField`, `chatField`, `chat_playerNum`, `chat_team`, `historyEditLines`, `g_color_table`, `cl_noprint`, `cl_conXOffset`, `com_cl_running`; renderer entry points `SCR_DrawSmallChar`, `SCR_DrawPic`, `SCR_FillRect`, `Field_Draw`, `Field_BigDraw`, `Field_Clear`, `VM_Call`
