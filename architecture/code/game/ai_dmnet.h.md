# code/game/ai_dmnet.h

## File Purpose
Public interface header for the Quake III Arena deathmatch bot AI state machine. It declares the state-enter functions (`AIEnter_*`) and state-node functions (`AINode_*`) that implement the bot's high-level behavioral FSM, along with diagnostic utilities.

## Core Responsibilities
- Declare the FSM state-entry transition functions (`AIEnter_*`) called when a bot switches states
- Declare the FSM state-node execution functions (`AINode_*`) called each frame to run the current state's logic
- Export node-switch diagnostic helpers for debugging bot behavior
- Define the `MAX_NODESWITCHES` guard constant to cap FSM transition history

## Key Types / Data Structures
None defined here; depends on `bot_state_t` defined elsewhere (likely `ai_main.h` / `g_local.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### AIEnter_* (family)
- **Signature:** `void AIEnter_<StateName>(bot_state_t *bs, char *s)`
- **Purpose:** Transition the bot into the named FSM state. Records the switch for diagnostics and sets `bs->ainode` to the corresponding `AINode_*` function pointer.
- **Inputs:** `bs` — bot state; `s` — string label identifying the caller/transition source (used in debug dumps).
- **Outputs/Return:** void
- **Side effects:** Modifies `bs->ainode`; logs to node-switch history.
- **Calls:** Defined in `ai_dmnet.c`; typically calls `BotRecordNodeSwitch`.
- **Notes:** One enter function per FSM state; `char *s` is a debug/trace string, not a game string.

### AINode_* (family)
- **Signature:** `int AINode_<StateName>(bot_state_t *bs)`
- **Purpose:** Execute one frame of logic for the named FSM state. Evaluates the bot's situation and either continues in the state or calls an `AIEnter_*` to transition out.
- **Inputs:** `bs` — current bot state snapshot.
- **Outputs/Return:** `int` — non-zero to continue; 0 typically signals the state completed/failed.
- **Side effects:** May modify `bs` fields; issues bot commands via the botlib API.
- **Calls:** Defined in `ai_dmnet.c`; may call `AIEnter_*`, movement/combat helpers, botlib goal/route functions.
- **Notes:** States cover the full lifecycle: `Intermission`, `Observer`, `Respawn`, `Stand`, `Seek_ActivateEntity`, `Seek_NBG` (Nearby Goal), `Seek_LTG` (Long-Term Goal), `Seek_Camp`, `Battle_Fight`, `Battle_Chase`, `Battle_Retreat`, `Battle_NBG`. Note `AINode_Seek_Camp` has no corresponding `AINode_` declaration — only an `AIEnter_` is declared.

### BotResetNodeSwitches
- **Signature:** `void BotResetNodeSwitches(void)`
- **Purpose:** Clears the circular/linear node-switch history buffer, typically at the start of a frame or respawn.
- **Side effects:** Resets global switch-tracking state in `ai_dmnet.c`.

### BotDumpNodeSwitches
- **Signature:** `void BotDumpNodeSwitches(bot_state_t *bs)`
- **Purpose:** Prints the recorded FSM transition history for a bot to the log/console, used for AI debugging.
- **Side effects:** I/O (logging).

## Control Flow Notes
`ai_dmnet.h` is the FSM backbone of the deathmatch bot. Each game frame, `ai_main.c` calls `bs->ainode(bs)` (the current node function pointer). That node function either runs its behavior or calls an `AIEnter_*` to switch states, which sets a new `bs->ainode`. `MAX_NODESWITCHES = 50` prevents infinite transition loops within a single frame.

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` (game-side bot state structure)
- `ai_dmnet.c` — provides all implementations declared here
- Consumers: `ai_main.c`, `ai_dmq3.c`, team AI files
