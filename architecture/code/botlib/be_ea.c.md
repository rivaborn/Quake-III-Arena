# code/botlib/be_ea.c

## File Purpose
Implements the Elementary Actions (EA) layer of the Quake III bot library, providing the lowest-level interface through which bots express input — movement, aiming, attacking, jumping, crouching, and chat commands. It translates high-level bot decisions into `bot_input_t` state buffers that are later consumed by the engine.

## Core Responsibilities
- Allocate and manage per-client `bot_input_t` input buffers
- Set action flags (attack, jump, crouch, walk, use, gesture, etc.) on bot input state
- Set movement direction, speed, and view angles
- Issue text-based client commands (say, say_team, tell, use item, drop item)
- Handle jump de-bounce logic via `ACTION_JUMPEDLASTFRAME`
- Expose `EA_GetInput` to retrieve accumulated input for a frame
- Expose `EA_ResetInput` to clear per-frame state while preserving jump carry-over

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `bot_input_t` | struct (defined in `botlib.h`) | Per-client input state: action flags, weapon, direction, speed, view angles, think time |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botinputs` | `bot_input_t *` | global | Heap-allocated array of input states, one per client slot; allocated in `EA_Setup` |

## Key Functions / Methods

### EA_Setup
- Signature: `int EA_Setup(void)`
- Purpose: Allocates zeroed hunk memory for all bot input slots.
- Inputs: None; reads `botlibglobals.maxclients` for array size.
- Outputs/Return: `BLERR_NOERROR` on success.
- Side effects: Allocates `botinputs` from the engine hunk.
- Calls: `GetClearedHunkMemory`
- Notes: Must be called before any other EA function.

### EA_Shutdown
- Signature: `void EA_Shutdown(void)`
- Purpose: Frees the `botinputs` array and nulls the pointer.
- Inputs: None.
- Outputs/Return: void.
- Side effects: Frees `botinputs`; sets it to `NULL`.
- Calls: `FreeMemory`

### EA_ResetInput
- Signature: `void EA_ResetInput(int client)`
- Purpose: Clears per-frame input state while carrying the `ACTION_JUMPEDLASTFRAME` flag forward from any jump that fired.
- Inputs: `client` — client index.
- Outputs/Return: void.
- Side effects: Modifies `botinputs[client]`; clears dir, speed, actionflags; sets `ACTION_JUMPEDLASTFRAME` if jump was active.
- Calls: `VectorClear`
- Notes: Jump de-bounce: if `ACTION_JUMP` was set, next frame sees `ACTION_JUMPEDLASTFRAME` to suppress re-triggering.

### EA_GetInput
- Signature: `void EA_GetInput(int client, float thinktime, bot_input_t *input)`
- Purpose: Snapshots the current accumulated input state into the caller's buffer.
- Inputs: `client`, `thinktime` (frame delta), `input` (output buffer).
- Outputs/Return: Fills `*input` via `Com_Memcpy`.
- Side effects: Writes `bi->thinktime`; does not reset state (caller must call `EA_ResetInput`).
- Calls: `Com_Memcpy`
- Notes: The commented-out reset block was migrated to `EA_ResetInput`.

### EA_Jump / EA_DelayedJump
- Signature: `void EA_Jump(int client)` / `void EA_DelayedJump(int client)`
- Purpose: Set `ACTION_JUMP` / `ACTION_DELAYEDJUMP`, but only if the bot did not jump last frame (de-bounce).
- Inputs: `client`.
- Outputs/Return: void.
- Side effects: Modifies `bi->actionflags`.
- Notes: Guards prevent repeated jump triggers on consecutive frames.

### EA_Move
- Signature: `void EA_Move(int client, vec3_t dir, float speed)`
- Purpose: Sets the movement direction and speed, clamping speed to ±`MAX_USERMOVE` (400).
- Inputs: `client`, `dir` (unit vector), `speed`.
- Side effects: Writes `bi->dir` and `bi->speed`.
- Calls: `VectorCopy`

### EA_View
- Signature: `void EA_View(int client, vec3_t viewangles)`
- Purpose: Sets the bot's desired view angles.
- Side effects: Writes `bi->viewangles`.
- Calls: `VectorCopy`

### EA_Say / EA_SayTeam / EA_Tell / EA_Command / EA_UseItem / EA_DropItem / EA_UseInv / EA_DropInv
- All delegate immediately to `botimport.BotClientCommand` with a formatted string.
- No state written to `botinputs`.

### EA_EndRegular
- Signature: `void EA_EndRegular(int client, float thinktime)`
- Purpose: Fully commented out; was the original combined get+reset entry point, now split into `EA_GetInput` + `EA_ResetInput`.

- **Notes on trivial setters** (`EA_Gesture`, `EA_Attack`, `EA_Talk`, `EA_Use`, `EA_Respawn`, `EA_Crouch`, `EA_Walk`, `EA_Action`, `EA_MoveUp`, `EA_MoveDown`, `EA_MoveForward`, `EA_MoveBack`, `EA_MoveLeft`, `EA_MoveRight`, `EA_SelectWeapon`): Each ORs or sets a single field on `botinputs[client]` with no other logic.

## Control Flow Notes
- **Init**: `EA_Setup` is called once during botlib initialization.
- **Per-frame**: Higher-level AI layers call individual EA setters to accumulate desired actions into `botinputs[client]`. At the end of the think cycle, `EA_GetInput` snapshots the state; the caller submits it to the engine, then `EA_ResetInput` clears it for the next frame.
- **Shutdown**: `EA_Shutdown` frees all allocated input buffers.

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `VectorCopy`, `VectorClear`, `Com_Memcpy`, `qboolean`
- `l_memory.h` — `GetClearedHunkMemory`, `FreeMemory`
- `../game/botlib.h` — `bot_input_t`, `ACTION_*` flags, `botimport` (struct of engine callbacks), `BLERR_NOERROR`
- `be_interface.h` — `botlibglobals` (provides `maxclients`)
- `botimport.BotClientCommand` — engine callback; defined in the engine, not in this file
- `va()` — defined in `q_shared.c`
