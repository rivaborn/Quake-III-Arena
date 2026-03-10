# code/game/ai_team.h — Enhanced Analysis

## Architectural Role

This header exposes the bot team AI module as a public interface within the Game VM subsystem. It serves as the abstraction layer between per-bot FSM logic (`ai_dmnet.c`) and team-level coordination, enabling bots to query/set teammate roles and broadcast team intentions via voice chat. All declarations map to implementations in `code/game/ai_team.c`, which runs inside the QVM and calls back into the engine via `trap_*` syscalls.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/g_bot.c`** — Likely calls `BotTeamAI(bs)` each server frame from the bot think loop (during `GAME_RUN_FRAME`)
- **`code/game/ai_dmnet.c`** — Bot deathmatch FSM calls `BotGetTeamMateTaskPreference` and `BotSetTeamMateTaskPreference` during team role negotiation
- **`code/game/ai_team.c`** — Internal team AI logic; these are forward declarations for its public API
- **`code/game/ai_chat.c`** — Likely integrates with voice chat functions for team announcements

### Outgoing (what this file's implementation depends on)
- **`code/botlib`** — `ai_team.c` calls `trap_BotLib*` syscall range (200–599) for pathfinding, goal evaluation, movement synthesis
- **Engine server layer** — `trap_SendServerCommand`, `trap_SetConfigstring` for voice chat broadcast to clients
- **Game globals** (`g_local.h`) — Bot state, team/entity references, configstring indices
- **`code/game/be_ai_*.h`** — Botlib public API declarations (goal, movement, weapon selection)

## Design Patterns & Rationale

**Syscall-based abstraction:** All team AI logic runs in the VM and communicates with the engine exclusively via `trap_*` syscalls. This isolation enables VMs to be reloaded without engine shutdown and supports both native DLL and QVM execution.

**Preference registry for role coordination:** Rather than a complex FSM-to-FSM negotiation protocol, bots store preferences (`int` tokens: e.g., `PREFERENCE_DEFLAG`, `PREFERENCE_OFLAG`) in bot state. This is a simple yet effective pattern for distributed team coordination—avoids synchronous negotiation overhead.

**Voice chat as string dispatch:** Voice chat functions accept a key string (not integer enum), enabling internationalization and server-side message mapping without VM recompilation.

**Frame-driven team updates:** `BotTeamAI` follows the per-frame tick pattern of all game entities, called during `SV_GameFrame` → `VM_Call(gvm, GAME_RUN_FRAME)`. Enables reactive team strategy updates without polling overhead.

## Data Flow Through This File

```
Server frame loop
  └─> BotTeamAI(bot_state_t *bs)
       ├─> Reads current team objectives/entity states via trap_* syscalls
       ├─> Evaluates team needs (defense, offense, flag pickup, etc.)
       ├─> Calls BotSetTeamMateTaskPreference() to coordinate roles
       └─> Calls BotVoiceChat() to broadcast team comms
            └─> Encoded as server command → multicast to all clients
```

Preference data enters via setter, is read via getter during team FSM decisions. Voice chat data is *consumed* by the engine (network dispatch), not stored.

## Learning Notes

**Idiomatic late-1990s architecture:** This thin syscall boundary is typical of the era (Doom III, Half-Life 2 also used VM-to-engine syscalls). Modern engines would use:
- **Callbacks/delegates** for event-driven updates
- **ECS or entity systems** for decoupled team state
- **Message queues** for async team coordination
- **Versioned capability negotiation** instead of simple int preferences

**Voice chat as localization vector:** Storing voice keys (e.g., `"VoiceCmd_OnDefense"`) instead of hardcoded strings allows the server/client to translate to localized audio files—a pattern seen in `ui/menudef.h` and cgame HUD callouts.

**No explicit team state ownership:** The header does not declare team-level globals (those live in `g_local.h` or embedded in bot state). This enforces a per-bot centric model; team decisions aggregate from individual bot logic rather than a central authority.

## Potential Issues

- **No bounds checking on teammate index:** `BotGetTeamMateTaskPreference(bs, teammate)` and `BotSetTeamMateTaskPreference(bs, teammate, ...)` assume `teammate` is valid; no validation visible at header level (must occur in `ai_team.c`).
- **Voicechat key validation:** `BotVoiceChat(bs, toclient, voicechat)` accepts a string key; parsing and index lookup happen in implementation—no type safety if key is invalid.
- **No error return codes:** All functions return `void`; callers cannot detect failures (e.g., invalid teammate, bad voice key). Implementation must fail silently or via side effects (missed messages).
