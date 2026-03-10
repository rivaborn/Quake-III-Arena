# code/q3_ui/ui_sparena.c

## File Purpose
Handles the launch sequence for a single-player arena in Quake III Arena's UI layer. It configures the necessary CVars and issues the server command to start a specific SP map.

## Core Responsibilities
- Ensures `sv_maxclients` is at least 8 before starting an SP arena
- Resolves the numeric SP level index from arena metadata, with special-case handling for "training" and "final" arenas
- Writes the resolved level selection into the `ui_spSelection` CVar for downstream use
- Executes the `spmap` command to load the chosen map

## Key Types / Data Structures
None defined in this file; relies entirely on types from `ui_local.h`.

## Global / File-Static State
None.

## Key Functions / Methods

### UI_SPArena_Start
- **Signature:** `void UI_SPArena_Start( const char *arenaInfo )`
- **Purpose:** Initiates a single-player arena session given an arena info string. Clamps `sv_maxclients`, resolves the arena's logical level number (with special overrides), updates `ui_spSelection`, and enqueues the `spmap` console command.
- **Inputs:** `arenaInfo` — a key/value info string (Q3 infostring format) containing at minimum `"num"`, `"map"`, and optionally `"special"` keys.
- **Outputs/Return:** `void`
- **Side effects:**
  - Sets CVar `sv_maxclients` to 8 if currently below 8
  - Sets CVar `ui_spSelection` to the resolved level index
  - Appends `spmap <mapname>\n` to the command buffer via `EXEC_APPEND`
- **Calls:**
  - `trap_Cvar_VariableValue("sv_maxclients")`
  - `trap_Cvar_SetValue("sv_maxclients", 8)`
  - `Info_ValueForKey` (for `"num"`, `"special"`, `"map"`)
  - `Q_stricmp` (checks `"training"` and `"final"`)
  - `UI_GetNumSPTiers` (to compute final-tier level index)
  - `trap_Cvar_SetValue("ui_spSelection", level)`
  - `trap_Cmd_ExecuteText(EXEC_APPEND, ...)`
  - `va` (format helper)
- **Notes:**
  - `"training"` maps to level `−4` (a sentinel below the normal tier range)
  - `"final"` maps to `UI_GetNumSPTiers() * ARENAS_PER_TIER`, placing it just past the last regular tier
  - `EXEC_APPEND` is used, so the command is not executed synchronously

## Control Flow Notes
This file is a pure utility invoked by the SP level-selection UI (e.g., `ui_splevel.c` or `ui_spskill.c`) when the player confirms arena entry. It is not called during frame updates, rendering, or shutdown — it is a one-shot trigger on player action.

## External Dependencies
- **Includes:** `ui_local.h` (which pulls in `q_shared.h`, `bg_public.h`, trap syscall declarations)
- **Defined elsewhere:**
  - `trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_Cmd_ExecuteText` — UI syscall stubs (`ui_syscalls.c`)
  - `Info_ValueForKey`, `Q_stricmp`, `atoi`, `va` — shared utilities (`q_shared.c`)
  - `UI_GetNumSPTiers`, `ARENAS_PER_TIER` — SP game info module (`ui_gameinfo.c` / `bg_public.h`)
