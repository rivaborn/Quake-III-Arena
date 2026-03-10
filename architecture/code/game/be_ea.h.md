# code/game/be_ea.h

## File Purpose
Declares the "Elementary Actions" (EA) API for the Quake III bot library. It provides the bot system's lowest-level abstraction over client input, translating high-level bot decisions into discrete client commands and movement/view inputs that are eventually forwarded to the server.

## Core Responsibilities
- Declare client-command EA functions (chat, arbitrary commands, discrete button actions)
- Declare movement EA functions (crouch, walk, strafe, jump, directional move)
- Declare view/weapon EA functions (aim direction, weapon selection)
- Declare input aggregation and dispatch functions (end-of-frame flush, input readback, reset)
- Declare module lifecycle entry points (setup/shutdown)

## Key Types / Data Structures
None defined in this header. References `vec3_t` and `bot_input_t` defined elsewhere.

## Global / File-Static State
None declared in this header.

## Key Functions / Methods

### EA_Say / EA_SayTeam
- Signature: `void EA_Say(int client, char *str)` / `void EA_SayTeam(int client, char *str)`
- Purpose: Queue a global or team-only chat message as a bot client command.
- Inputs: `client` — bot client index; `str` — message text.
- Outputs/Return: void
- Side effects: Adds a `say`/`say_team` command to the bot's pending input buffer.
- Calls: Defined in `botlib/be_ea.c`.
- Notes: Thin wrapper over `EA_Command`.

### EA_Command
- Signature: `void EA_Command(int client, char *command)`
- Purpose: Queue an arbitrary client console command string for the bot.
- Inputs: `client` — bot client index; `command` — raw command string.
- Outputs/Return: void
- Side effects: Buffers command for delivery on next `EA_EndRegular`.
- Calls: Defined in `botlib/be_ea.c`.

### EA_Action
- Signature: `void EA_Action(int client, int action)`
- Purpose: Set a bitmask action flag (e.g., attack, crouch, jump) in the bot's current usercmd.
- Inputs: `client`; `action` — bitmask constant (e.g., `ACTION_ATTACK`).
- Outputs/Return: void
- Side effects: Modifies the bot's pending `usercmd_t` button/movement bits.

### EA_Move
- Signature: `void EA_Move(int client, vec3_t dir, float speed)`
- Purpose: Set the bot's movement direction and speed for the current frame.
- Inputs: `client`; `dir` — normalized world-space direction; `speed` — scalar magnitude.
- Outputs/Return: void
- Side effects: Writes `forwardmove`/`sidemove`/`upmove` fields of the pending usercmd.

### EA_View
- Signature: `void EA_View(int client, vec3_t viewangles)`
- Purpose: Set the bot's view angles (pitch/yaw/roll) for the current frame.
- Inputs: `client`; `viewangles` — target angles in degrees.
- Outputs/Return: void
- Side effects: Writes `angles` fields of the pending usercmd.

### EA_EndRegular
- Signature: `void EA_EndRegular(int client, float thinktime)`
- Purpose: Finalize and submit the accumulated regular input (movement, view, buttons) for the current think cycle.
- Inputs: `client`; `thinktime` — duration of this think frame in seconds.
- Outputs/Return: void
- Side effects: Flushes the pending usercmd to the bot client input pipeline.

### EA_GetInput
- Signature: `void EA_GetInput(int client, float thinktime, bot_input_t *input)`
- Purpose: Read back the current accumulated bot input into a `bot_input_t` structure.
- Inputs: `client`; `thinktime`; `input` — output struct pointer.
- Outputs/Return: Populates `*input`.
- Side effects: None beyond filling the struct.

### EA_ResetInput
- Signature: `void EA_ResetInput(int client)`
- Purpose: Clear all pending input state for the bot client, ready for a new frame.
- Inputs: `client`.
- Side effects: Zeroes the bot's pending usercmd and command buffer.

### EA_Setup / EA_Shutdown
- Signature: `int EA_Setup(void)` / `void EA_Shutdown(void)`
- Purpose: Initialize and tear down the EA module's internal state (client input arrays).
- Outputs/Return: `EA_Setup` returns non-zero on success.
- Side effects: Allocates/frees per-client input state arrays.

## Control Flow Notes
The EA layer sits at the bottom of the bot AI stack. Each bot think frame, higher-level AI layers (movement, goal, weapon) call individual `EA_*` functions to accumulate intent. At the end of the frame, `EA_EndRegular` / `EA_GetInput` aggregate the result into a `bot_input_t`, which the server-side bot code (`sv_bot.c`) feeds into the standard client input path. `EA_Setup`/`EA_Shutdown` are called during botlib initialization and shutdown.

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_input_t` — defined in `botlib.h` / `be_aas_def.h`
- Implementation: `botlib/be_ea.c`
- Consumed by: `game/ai_move.c`, `game/ai_dmq3.c`, and other game-side AI modules via the botlib interface
