# code/game/ai_team.h

## File Purpose
Public interface header for Quake III Arena's bot team AI module. Declares the entry points and utility functions used by other game modules to drive team-based bot behavior and voice communication.

## Core Responsibilities
- Expose the main team AI tick function (`BotTeamAI`) for per-frame bot updates
- Provide teammate task preference get/set API for coordinating team roles
- Declare voice chat dispatch functions for bot-to-client voice communication

## Key Types / Data Structures
None. (All types, including `bot_state_t`, are defined elsewhere.)

## Global / File-Static State
None.

## Key Functions / Methods

### BotTeamAI
- Signature: `void BotTeamAI(bot_state_t *bs)`
- Purpose: Main per-frame team AI update for a single bot. Evaluates team objectives (flag capture, defense, etc.) and dispatches bot actions.
- Inputs: `bs` — pointer to the bot's state structure
- Outputs/Return: void
- Side effects: Modifies bot state, may issue movement/action commands via botlib
- Calls: Not inferable from this file
- Notes: Likely called each server frame from the game module's bot think loop

### BotGetTeamMateTaskPreference
- Signature: `int BotGetTeamMateTaskPreference(bot_state_t *bs, int teammate)`
- Purpose: Queries the stored task preference (e.g., attack, defend, escort) assigned to a specific teammate index.
- Inputs: `bs` — bot state; `teammate` — teammate client number or index
- Outputs/Return: integer preference value (semantics defined in `ai_team.c`)
- Side effects: None (read-only query)
- Calls: Not inferable from this file
- Notes: Used to avoid assigning duplicate roles to teammates

### BotSetTeamMateTaskPreference
- Signature: `void BotSetTeamMateTaskPreference(bot_state_t *bs, int teammate, int preference)`
- Purpose: Stores a task preference for a given teammate, enabling role coordination across the bot team.
- Inputs: `bs` — bot state; `teammate` — teammate index; `preference` — task role constant
- Outputs/Return: void
- Side effects: Writes into `bs` (or shared team state embedded in it)
- Calls: Not inferable from this file

### BotVoiceChat
- Signature: `void BotVoiceChat(bot_state_t *bs, int toclient, char *voicechat)`
- Purpose: Sends a voice chat message from the bot to a specific client (or broadcast).
- Inputs: `bs` — bot state; `toclient` — destination client number; `voicechat` — voice chat key string
- Outputs/Return: void
- Side effects: Issues a game syscall or network message; may print to HUD
- Calls: Not inferable from this file

### BotVoiceChatOnly
- Signature: `void BotVoiceChatOnly(bot_state_t *bs, int toclient, char *voicechat)`
- Purpose: Variant of `BotVoiceChat` that sends voice chat without accompanying text; restricts output to audio-only channel.
- Inputs: Same as `BotVoiceChat`
- Outputs/Return: void
- Side effects: Same as `BotVoiceChat` but suppresses text echo
- Calls: Not inferable from this file

## Control Flow Notes
This is a header-only declaration file. `BotTeamAI` is the frame-driven entry point, called from the game's bot think loop (likely in `g_bot.c` or `ai_main.c`). The preference functions are utility accessors called during team coordination logic within `ai_team.c`. Voice chat functions are called reactively in response to team events.

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation bodies reside in `code/game/ai_team.c`
