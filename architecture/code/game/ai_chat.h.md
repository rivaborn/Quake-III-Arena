# code/game/ai_chat.h

## File Purpose
Public interface header declaring bot AI chat functions for Quake III Arena. It exposes event-driven chat triggers and utility functions that allow bots to send contextually appropriate chat messages during gameplay.

## Core Responsibilities
- Declare chat event hooks for game lifecycle events (enter/exit game, level start/end)
- Declare combat-contextual chat triggers (hit, death, kill, suicide)
- Declare utility functions for chat timing, position validation, and testing

## Key Types / Data Structures
None — all functions operate on `bot_state_t *`, defined elsewhere.

## Global / File-Static State
None.

## Key Functions / Methods

### BotChat_EnterGame / BotChat_ExitGame
- Signature: `int BotChat_EnterGame(bot_state_t *bs)` / `int BotChat_ExitGame(bot_state_t *bs)`
- Purpose: Trigger a chat message when a bot joins or leaves the game session.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — likely non-zero if a chat was successfully issued.
- Side effects: Not inferable from this file.
- Calls: Not inferable from this file.

### BotChat_StartLevel / BotChat_EndLevel
- Signature: `int BotChat_StartLevel(bot_state_t *bs)` / `int BotChat_EndLevel(bot_state_t *bs)`
- Purpose: Trigger a chat message at the beginning or end of a map/round.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — likely non-zero on success.
- Side effects: Not inferable from this file.
- Calls: Not inferable from this file.

### BotChat_HitTalking / BotChat_HitNoDeath / BotChat_HitNoKill
- Signature: `int BotChat_Hit*(bot_state_t *bs)`
- Purpose: Contextual hit reactions — bot was hit while talking, took a hit without dying, or hit a target without killing it.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — likely non-zero if a chat was issued.
- Side effects: Not inferable from this file.
- Calls: Not inferable from this file.

### BotChat_Death / BotChat_Kill / BotChat_EnemySuicide
- Signature: `int BotChat_Death(bot_state_t *bs)` etc.
- Purpose: Trigger kill/death/enemy-suicide taunts or reactions in chat.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — likely non-zero if a chat was issued.
- Side effects: Not inferable from this file.
- Calls: Not inferable from this file.

### BotChat_Random
- Signature: `int BotChat_Random(bot_state_t *bs)`
- Purpose: Trigger a random idle or filler chat message unprompted by a specific event.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — likely non-zero if a chat was issued.
- Side effects: Not inferable from this file.
- Calls: Not inferable from this file.

### BotChatTime
- Signature: `float BotChatTime(bot_state_t *bs)`
- Purpose: Returns the simulated time it takes the bot to "type" the currently selected chat message, used to delay movement or actions while chatting.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `float` — duration in seconds.
- Side effects: None apparent.
- Calls: Not inferable from this file.

### BotValidChatPosition
- Signature: `int BotValidChatPosition(bot_state_t *bs)`
- Purpose: Checks whether the bot is in a safe/valid position to chat (e.g., not in combat or mid-air).
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `int` — non-zero if chatting is currently appropriate.
- Side effects: None apparent.
- Calls: Not inferable from this file.

### BotChatTest
- Signature: `void BotChatTest(bot_state_t *bs)`
- Purpose: Developer/debug utility to exercise all initial chat scenarios for a bot.
- Inputs: `bs` — pointer to the bot's state.
- Outputs/Return: `void`
- Side effects: Likely prints or fires multiple chat messages; debug/test use only.
- Calls: Not inferable from this file.

## Control Flow Notes
This header is included by `ai_main.c` or similar bot AI driver code. The event functions are called reactively from game events (kills, deaths, level transitions) and periodically from the bot think loop for random chat. `BotValidChatPosition` and `BotChatTime` gate whether and for how long a bot remains in a "chatting" state during its frame update.

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation resides in `code/game/ai_chat.c`
