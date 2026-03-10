# code/game/ai_cmd.h

## File Purpose
Header file for the bot AI command/message processing subsystem in Quake III Arena. It declares the public interface for bot team-command parsing and team goal reporting used by the game module's AI layer.

## Core Responsibilities
- Exposes the `BotMatchMessage` function for parsing and dispatching incoming chat/voice commands to a bot
- Exposes `BotPrintTeamGoal` for outputting the bot's current team objective
- Declares the `notleader` array used to track which clients have been flagged as non-leaders across the bot subsystem

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `notleader` | `int[MAX_CLIENTS]` | global (extern) | Per-client flag array indicating whether a given client is not considered a team leader; consulted during team-command processing |

## Key Functions / Methods

### BotMatchMessage
- **Signature:** `int BotMatchMessage(bot_state_t *bs, char *message);`
- **Purpose:** Attempts to match an incoming chat/voice string against known bot command patterns and executes the appropriate response or state change.
- **Inputs:** `bs` — pointer to the bot's state; `message` — null-terminated chat string to parse.
- **Outputs/Return:** `int` — likely non-zero if the message was matched and handled, zero otherwise.
- **Side effects:** May mutate `bs` fields (goals, flags, behavior state); may set entries in `notleader`.
- **Calls:** Defined in `ai_cmd.c`; not inferable from this header alone.
- **Notes:** The archive comment references `ai_chat.c`, suggesting the command/chat subsystems share close lineage or were historically split from the same file.

### BotPrintTeamGoal
- **Signature:** `void BotPrintTeamGoal(bot_state_t *bs);`
- **Purpose:** Prints or announces the bot's current team goal, likely via the game's print/console mechanism for debugging or in-game team feedback.
- **Inputs:** `bs` — pointer to the bot's state.
- **Outputs/Return:** void.
- **Side effects:** I/O — emits text output (console or in-game chat).
- **Calls:** Defined in `ai_cmd.c`; not inferable from this header.
- **Notes:** None inferable beyond signature.

## Control Flow Notes
This header is included by other game-side AI files (e.g., `ai_main.c`, `ai_team.c`) that drive the per-frame bot think loop. `BotMatchMessage` would be called when a new server command or chat event is dispatched to a bot entity; `BotPrintTeamGoal` is a diagnostic/output helper invoked on demand.

## External Dependencies
- **`bot_state_t`** — defined in `ai_main.h` or `g_local.h`; the central bot runtime state structure.
- **`MAX_CLIENTS`** — defined in `q_shared.h`; engine-wide client count limit.
- Implementation lives in `code/game/ai_cmd.c`.
