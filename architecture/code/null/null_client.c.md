# code/null/null_client.c

## File Purpose
Provides a null (stub) implementation of the client subsystem for use in dedicated server or headless builds where no actual client functionality is required. All functions have empty bodies or return safe default values.

## Core Responsibilities
- Satisfy the linker's demand for client API symbols in non-client builds
- Provide a no-op `CL_Init` that registers the `cl_shownet` cvar (minimum viable init)
- Return safe defaults (`qfalse`/`qtrue`) from boolean-returning stubs
- Allow the server-side codebase to compile and link without the full client module

## Key Types / Data Structures
None (no types defined; all types come from `client.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cl_shownet` | `cvar_t *` | global | Registered in `CL_Init`; declared `extern` in `client.h`. Only cvar initialized; all others are left NULL. |

## Key Functions / Methods

### CL_Init
- Signature: `void CL_Init(void)`
- Purpose: Minimal client initialization; registers `cl_shownet` cvar.
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `Cvar_Get`, writing into the cvar system.
- Calls: `Cvar_Get`
- Notes: Only this function has any real work; all other inits are no-ops.

### CL_CDKeyValidate
- Signature: `qboolean CL_CDKeyValidate(const char *key, const char *checksum)`
- Purpose: Stub CD-key validation; always reports the key as valid.
- Inputs: `key`, `checksum` — ignored
- Outputs/Return: `qtrue`
- Side effects: None
- Calls: None
- Notes: Allows dedicated/headless builds to pass any CD-key check unconditionally.

### CL_GameCommand / UI_GameCommand
- Signature: `qboolean CL_GameCommand(void)` / `qboolean UI_GameCommand(void)`
- Purpose: Indicate no client-side command was consumed.
- Inputs: None
- Outputs/Return: `qfalse`
- Side effects: None
- Calls: None
- Notes: Comment `bk001204 - non-void` marks the return value as a correctness fix.

### Notes (trivial stubs)
- `CL_Shutdown`, `CL_MouseEvent`, `Key_WriteBindings`, `CL_Frame`, `CL_PacketEvent`, `CL_CharEvent`, `CL_Disconnect`, `CL_MapLoading`, `CL_KeyEvent`, `CL_ForwardCommandToServer`, `CL_ConsolePrint`, `CL_JoystickEvent`, `CL_InitKeyCommands`, `CL_CDDialog`, `CL_FlushMemory`, `CL_StartHunkUsers`, `CL_ShutdownAll` — all empty no-ops with no side effects.

## Control Flow Notes
This file is selected at build time instead of the real `code/client/` implementation when building a dedicated server (`null` platform target). It has no frame/update integration; `CL_Frame` is a no-op so the main loop calling it simply does nothing on the client side.

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- `Cvar_Get` — defined in `code/qcommon/cvar.c`
- `cvar_t`, `netadr_t`, `msg_t`, `fileHandle_t`, `qboolean` — defined elsewhere in qcommon/game headers
