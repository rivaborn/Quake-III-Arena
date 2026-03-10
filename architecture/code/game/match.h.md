# code/game/match.h

## File Purpose
This header defines all symbolic constants used by the bot AI's natural-language chat matching and team-command messaging system. It provides message type identifiers, match-template context flags, command sub-type bitmasks, and variable-slot indices that map parsed chat tokens to structured bot commands.

## Core Responsibilities
- Define the escape character (`EC`) used to delimit in-game chat tokens
- Declare bitmask flags for match-template parsing contexts (e.g., CTF, teammate address, time)
- Enumerate all bot-to-bot and bot-to-player message type codes (`MSG_*`)
- Provide command sub-type bitmask flags (`ST_*`) for qualifying message semantics
- Define named indices for word-replacement variable slots in message templates

## Key Types / Data Structures
None. This file contains only preprocessor `#define` constants.

## Global / File-Static State
None.

## Key Functions / Methods
None. Header is purely declarative.

## Control Flow Notes
This file is consumed at compile time by the bot AI modules (`ai_chat.c`, `ai_cmd.c`, `ai_team.c`, `ai_dmnet.c`) and the botlib chat system. It does not participate directly in any runtime flow, but its constants gate every branch in the bot chat-matching logic:

- **Init:** `MSG_ENTERGAME` is checked when a bot first joins a game.
- **Frame/Update:** Each game frame may trigger chat parsing; `MTCONTEXT_*` flags select which match templates are active. Parsed results are dispatched by `MSG_*` code, and sub-type qualifiers (`ST_*`) refine the command intent.
- **Shutdown:** No role.

## External Dependencies
- No includes in this file itself.
- Consumed by: `code/game/ai_chat.c`, `code/game/ai_cmd.c`, `code/game/ai_team.c`, and related bot source files (defined elsewhere).
- `EC` (`"\x19"`) must match the escape character literal used in chat string definitions in `g_cmd.c` (comment-enforced contract, not compiler-enforced).

---

**Notes:**
- `ST_1FCTFGOTFLAG` (`65535` / `0xFFFF`) appears to be a sentinel or "all flags set" value rather than a single-bit flag — its use among power-of-two `ST_*` values suggests a special aggregate case for one-flag CTF mode.
- Several `#define` names collide in value (e.g., `THE_ENEMY` and `THE_TEAM` are both `7`; `FLAG` and `PLACE` are both `1`; `ADDRESSEE` and `MESSAGE` are both `2`) — these are intentional aliasing of variable-slot indices for different message contexts, not bugs.
- `MSG_WHOISTEAMLAEDER` contains a typo ("LAEDER" instead of "LEADER") preserved from the original id Software source.
