# code/game/ai_vcmd.h

## File Purpose
Public interface header for bot voice chat command handling in Quake III Arena's game-side AI system. Declares functions used to process and respond to voice chat events as part of bot behavioral logic.

## Core Responsibilities
- Expose the bot voice chat command dispatcher (`BotVoiceChatCommand`)
- Expose the "defend" voice chat response handler (`BotVoiceChat_Defend`)
- Serve as the include boundary between `ai_vcmd.c` and other game AI modules

## Key Types / Data Structures
None defined in this file; relies on `bot_state_t` defined elsewhere (likely `ai_main.h` / `g_local.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### BotVoiceChatCommand
- **Signature:** `int BotVoiceChatCommand(bot_state_t *bs, int mode, char *voicechat)`
- **Purpose:** Central dispatcher — evaluates an incoming voice chat string and routes the bot to the appropriate behavioral response.
- **Inputs:** `bs` — bot state; `mode` — team/global chat scope; `voicechat` — voice chat token/identifier string.
- **Outputs/Return:** `int` — likely non-zero if the command was recognized and handled, 0 otherwise.
- **Side effects:** Not inferable from this file; implementation in `ai_vcmd.c`.
- **Calls:** Not inferable from this file.
- **Notes:** `mode` probably maps to constants distinguishing team-only vs. all-players voice scope.

### BotVoiceChat_Defend
- **Signature:** `void BotVoiceChat_Defend(bot_state_t *bs, int client, int mode)`
- **Purpose:** Handles the "defend" voice chat order, directing the bot to adopt a defensive goal toward a specific client or location.
- **Inputs:** `bs` — bot state; `client` — client index issuing or being referenced by the command; `mode` — chat scope.
- **Outputs/Return:** `void`.
- **Side effects:** Likely modifies `bs` goal/chat state; not further inferable from this file.
- **Calls:** Not inferable from this file.
- **Notes:** One of potentially several `BotVoiceChat_*` handlers (e.g., `_Attack`, `_Follow`) declared only in `ai_vcmd.c`.

## Control Flow Notes
This header is included by other game AI files (e.g., `ai_team.c`, `ai_dmnet.c`) that need to dispatch or react to voice commands. It plays no direct role in frame/update loops — it provides declarations so callers can invoke `BotVoiceChatCommand` when a voice event is received server-side.

## External Dependencies
- `bot_state_t` — defined elsewhere, likely `ai_main.h`
- Implementation body: `ai_vcmd.c` (noted in `$Archive` comment)
- No standard library includes in this header
