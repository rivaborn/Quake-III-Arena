# code/cgame/cg_syscalls.c

## File Purpose
Implements the cgame module's system call interface for the DLL build path. Each `trap_*` function wraps a variadic `syscall` function pointer that dispatches into the engine using integer opcode identifiers defined in `cg_public.h`.

## Core Responsibilities
- Receive and store the engine-provided syscall dispatcher via `dllEntry`
- Expose typed `trap_*` wrappers for every engine service the cgame module needs
- Convert `float` arguments to `int`-width bit-reinterpretations via `PASSFLOAT` before passing through the integer-only syscall ABI
- Cover all engine subsystems: console, cvar, filesystem, collision, sound, renderer, input, cinematic, and snapshot/game-state retrieval

## Key Types / Data Structures
None defined in this file; all types (`vmCvar_t`, `trace_t`, `refEntity_t`, etc.) are from included headers.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `syscall` | `static int (QDECL *)( int arg, ... )` | static (file) | Function pointer to the engine dispatcher; initialized to `-1` cast as a pointer; set by `dllEntry` at module load |

## Key Functions / Methods

### dllEntry
- **Signature:** `void dllEntry( int (QDECL *syscallptr)( int arg, ... ) )`
- **Purpose:** Engine entry point called immediately after the DLL is loaded; stores the engine's syscall dispatcher
- **Inputs:** `syscallptr` — the engine-provided variadic dispatch function
- **Outputs/Return:** void
- **Side effects:** Writes the file-static `syscall` pointer
- **Calls:** None
- **Notes:** Must be called before any `trap_*` function; until then `syscall` holds an invalid sentinel (`-1`)

### PASSFLOAT
- **Signature:** `int PASSFLOAT( float x )`
- **Purpose:** Type-puns a `float` to its raw `int` bit-pattern so floats can be passed through the `int`-typed syscall variadic ABI without conversion loss
- **Inputs:** `x` — float value
- **Outputs/Return:** `int` bit-representation of the float
- **Side effects:** None
- **Calls:** None
- **Notes:** Used extensively by renderer and input traps (`trap_R_DrawStretchPic`, `trap_R_AddLightToScene`, `trap_SetUserCmdValue`, `trap_R_LerpTag`, etc.)

### trap_* (all wrappers)
- **Pattern:** Each function calls `syscall( CG_<OPCODE>, args... )`, forwarding typed parameters through the integer ABI
- **Notable groups:**
  - **Cvar:** `trap_Cvar_Register/Update/Set/VariableStringBuffer`
  - **Filesystem:** `trap_FS_FOpenFile/Read/Write/FCloseFile/Seek`
  - **Collision:** `trap_CM_LoadMap`, box/capsule traces (plain and transformed), `trap_CM_MarkFragments`
  - **Sound:** start/stop/looping sounds, spatialization, background track, register
  - **Renderer:** world load, model/skin/shader registration, scene building (`ClearScene`, `AddRefEntity`, `AddPoly`, `RenderScene`), 2D drawing, font, shader remap, PVS test
  - **Game state / snapshots:** `trap_GetGameState`, `trap_GetSnapshot`, `trap_GetServerCommand`, `trap_GetCurrentCmdNumber`, `trap_GetUserCmd`
  - **Input / keys:** `trap_Key_IsDown/GetCatcher/SetCatcher/GetKey`, `trap_SetUserCmdValue`
  - **Cinematic:** `trap_CIN_PlayCinematic/StopCinematic/RunCinematic/DrawCinematic/SetExtents`
  - **Misc:** `trap_SnapVector`, `trap_RealTime`, `trap_MemoryRemaining`, `trap_PC_*` (script/parser), `trap_GetEntityToken`, `trap_R_inPVS`

## Control Flow Notes
- Called at module initialization: the engine calls `dllEntry` to inject the syscall pointer.
- From that point forward, every frame the cgame calls `trap_*` functions freely during `CG_DrawActiveFrame` and other cgame logic.
- This file has no frame loop of its own; it is a pure dispatch layer invoked by other cgame source files.
- Excluded from QVM builds (`#ifdef Q3_VM #error`); the QVM build uses `cg_syscalls.asm` instead, which maps the same `trap_*` symbols directly to Q3VM syscall instructions.

## External Dependencies
- **Includes:** `cg_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:** All `CG_*` opcode constants (e.g., `CG_PRINT`, `CG_R_RENDERSCENE`) — defined in `cg_public.h`; all struct types (`trace_t`, `refEntity_t`, `snapshot_t`, `glconfig_t`, etc.) — defined in shared/renderer headers; `QDECL` calling-convention macro — from `q_shared.h`
