# code/game/ai_chat.c — Enhanced Analysis

## Architectural Role

This file implements the **chat presentation layer** of the bot AI subsystem within the server-side game VM. It acts as a translational bridge between high-level game events (kills, deaths, level transitions) detected by the FSM state machines in `ai_dmq3.c` / `ai_dmnet.c` and the botlib chat system (`be_ai_chat.h`). While the bulk of bot AI decision-making lives in botlib, this file encodes *when* to chat, *what category* to chat in, and *what context variables* to populate—leaving actual message template expansion to the botlib layer via `BotAI_BotInitialChat()`.

## Key Cross-References

### Incoming (who depends on this file)

- **`ai_dmq3.c` / `ai_dmnet.c` (FSM state machines):** Call `BotChat_*()` functions reactively when specific game events occur (spawn, kill, death, item pickup, level end). These are the primary callers, invoked from FSM nodes in response to state transitions.
- **`ai_main.c`:** Likely drives `BotChat_Random()` on a frame-time-proportional basis during idle periods; contains main bot think loop that interleaves FSM ticks.
- **Debug console (via `BotChatTest()`):** Called from higher-level AI debug commands to exhaustively exercise all chat categories for validation.

### Outgoing (what this file depends on)

- **botlib `be_ai_chat.h` layer:**
  - `BotAI_BotInitialChat()` — queues a chat message with template name and variable substitutions; delegates actual template parsing, variable expansion, and delivery to botlib.
  - `trap_BotNumInitialChats()` — queries how many chat variants exist for a given template category (used in `BotChat_Death()` to randomize sub-category selection).
  - `trap_BotEnterChat()` — directly sends a queued chat to all clients (used in `BotChatTest()` only).

- **botlib `be_ai_char.h` / characteristic system:**
  - `trap_Characteristic_BFloat()` — fetches per-bot personality floats (e.g., `CHARACTERISTIC_CHAT_ENTEREXITGAME`, `CHARACTERISTIC_CHAT_STARTENDLEVEL`) to gate chat probability.

- **game module globals / helpers:**
  - `TeamPlayIsOn()`, `BotSameTeam()` — check team mode and affiliation; suppress chat in team-play, issue `vtaunt` voice commands instead.
  - `BotIsDead()`, `BotIsObserver()` — suppress chat when inappropriate.
  - `BotNumActivePlayers()` — suppress chat if fewer than 2 active players (single-player is boring).
  - `BotValidChatPosition()` — veto chat if bot is in hostile/unsafe state (powerups active, in lava, visible enemies nearby).
  - `BotEntityVisible()`, `BotVisibleEnemies()` — safety gates (e.g., don't taunt if enemies still visible).

- **game module state accessors:**
  - `BotSameTeam()`, `BotEntityInfo()`, `BotAI_GetClientState()` — populate `bot_state_t` and query opponent/teammate state.
  - `EasyClientName()`, `ClientName()` — fetch player names for dynamic variable substitution.

- **global cvars:**
  - `bot_nochat` — master on/off switch (all chat suppressed if set).
  - `bot_fastchat` — skip random probability roll (fast-chat mode for testing).
  - `gametype`, `g_entities`, `FloatTime()` — game state and time.

- **shared includes:**
  - `chars.h`, `inv.h`, `syn.h`, `match.h` — bot profile data and string utilities (likely define characteristic keys and inventory slot indices).

## Design Patterns & Rationale

### Event-Driven, Not Frame-Driven
Most `BotChat_*()` functions are called reactively from FSM state nodes when game events occur—not on every frame. This is memory- and network-efficient: chat is inherently infrequent and bursty. Only `BotChat_Random()` uses a frame-time probability to emit unsolicited chatter during idle periods, maintaining the illusion of personality.

### Rate-Limiting via `TIME_BETWEENCHATTING`
All chat functions enforce a global 25-second cooldown (`bs->lastchat_time` check). This prevents bot spam and reflects realistic human chat frequency. Cooldown is per-bot (stored in `bot_state_t`), not global, so multiple bots can chat simultaneously without interfering.

### Static Name Buffers for Return Values
Helper functions like `BotFirstClientInRankings()`, `BotRandomOpponentName()`, and `BotMapTitle()` return pointers to `static char[]` buffers. This pattern is efficient (no heap allocation) but thread-unsafe under SMP; acceptable here because the server runs bot logic on a single thread despite the engine's front-end/back-end SMP support.

### Layered Permission Gating
Each chat function applies a cascading filter:
1. **Global kill-switch:** `bot_nochat`
2. **Cooldown:** `lastchat_time` + `TIME_BETWEENCHATTING`
3. **Game mode:** No chat in teamplay, tournament, observer mode (but `vtaunt` voice commands in teamplay instead)
4. **Player threshold:** Suppress if ≤1 active players
5. **Bot safety:** `BotValidChatPosition()` rejects if in lava, under water, holding powerups, or standing on non-solid ground
6. **Combat safety:** `BotChat_Kill()` suppressed if visible enemies remain
7. **Personality characteristic:** `trap_Characteristic_BFloat()` provides per-bot chat probability; can be 0.0 (silent bot) to 1.0 (chatty bot)

This design respects both global server config (no spam) and per-bot personality (immersion).

### Delegation to botlib for Template Expansion
This file encodes *decision logic* (when and what to chat); botlib handles *rendering* (template lookup, variable substitution, localization). The split is clean: game knows game state (kills, deaths, items), botlib knows chat metadata and client formatting. `BotAI_BotInitialChat()` is a closure that captures the bot, template name, and up to ~10 substitution strings, then botlib expands them into actual chat text.

## Data Flow Through This File

```
Game Event (e.g., kill detected in g_combat.c)
    ↓
FSM state machine (ai_dmq3.c / ai_dmnet.c) invokes BotChat_Kill()
    ↓
[Gate 1: bot_nochat, cooldown, game mode, player count]
    ↓
[Gate 2: BotValidChatPosition(), visible enemies, same-team check]
    ↓
[Gate 3: Personality characteristic (probability roll)]
    ↓
Query game state: killer, cause of death, opponent names, map name
    ↓
Call BotAI_BotInitialChat(bs, "kill_*", var0, var1, ..., varN)
    ↓
botlib: Look up "kill_*" template, substitute variables, format message
    ↓
Server network layer: Broadcast chat to all clients
```

**Per-bot state involved:** `bot_state_t.lastchat_time`, `bot_state_t.chatto` (target audience: `CHAT_ALL`, `CHAT_TEAM`, etc.), `bot_state_t.character` (personality profile), `bot_state_t.inventory[]`, `bot_state_t.origin`, `bot_state_t.entitynum`.

## Learning Notes

### Why This Architecture?
Quake III's bot AI is split into a **self-contained library** (botlib, in `code/botlib/`) and **game-specific logic** (in `code/game/ai_*.c`). This separation allows:
- Botlib to be reused across game modes (DM, TDM, CTF, Arena, MissionPack variants) with minimal changes
- Game logic to focus on authoritative rule enforcement, not low-level navigation/pathfinding
- Easy runtime DLL swapping of botlib for iterative AI tuning (botlib is a separate `.so`/`.dll`)

**ai_chat.c's role:** It's the "glue layer" that translates from game-logic events into botlib service calls. It's lightweight, readable, and domain-specific (chat logic only).

### Idiomatic Patterns
- **Characteristic system:** Rather than hard-coded bot personality, Q3 uses a `characteristics.c` database with per-bot float/int/string tuples loaded from `.c` files. This allows LTK map-makers to author bots with specific personalities (camper, rusher, sniper). `BotChat_*()` queries these characteristics to scale chat frequency, taunting, etc.
- **String templates:** Chat messages are not hard-coded; botlib loads them from script files (`.c` chat files, parsed by `be_ai_chat.c`). This enables easy localization and mod customization.
- **Syscall delegation:** Game VMs never directly call botlib functions; instead, they issue `trap_BotLib*` syscalls (opcode range 200–599). The server marshals these calls to botlib via the `botlib_import_t` vtable. This sandboxing prevents cheating (a malicious cgame VM cannot directly modify botlib state).

### Modern Comparison
- **ECS engines** (Bevy, Unity) separate "logic" (systems) from "data" (components). Q3's `ai_chat.c` acts like a **system**: it queries bot state, applies rules, and issues commands. botlib is the **data/service layer**.
- **FSM pattern:** `ai_dmq3.c` implements a Hierarchical Finite State Machine (HFSM) with nested states (e.g., `AINode_DM_Movement` → `AINode_Combat`). Each node checks conditions and may invoke `BotChat_*()` before transitioning. Modern engines use behavior trees or planners, but FSM is elegant here for controlled, predictable bot behavior.
- **Cooldown/rate-limiting:** Standard in all game AI to prevent spam and resource exhaustion. The `lastchat_time` pattern is a simple, effective time-based gate.

## Potential Issues

1. **Static buffer return values are non-reentrant**
   - `BotFirstClientInRankings()`, `BotLastClientInRankings()`, `BotRandomOpponentName()`, `BotMapTitle()` all return pointers to function-local `static char[]` buffers.
   - **Risk:** If a function is called twice in the same expression (e.g., `BotAI_BotInitialChat(..., BotRandomOpponentName(bs), ..., BotMapTitle(), ...)`), the second call overwrites the first buffer.
   - **Actual risk level: Low** — these functions are only called once per `BotChat_*()` invocation, and results are consumed immediately by `BotAI_BotInitialChat()`, which copies them into an internal queue. No known case of double-call in same statement.
   - **Potential fix:** Use caller-provided buffers or dynamic allocation (more defensive but adds overhead).

2. **`BotValidChatPosition()` performs a BSP trace every invocation**
   - Called by most `BotChat_*()` functions to check if the bot is standing on solid ground.
   - **Risk:** If chat is triggered frequently, trace overhead could add up.
   - **Actual risk level: Negligible** — chat is rate-limited to once per 25 seconds per bot; a single trace is cheap.

3. **Missing null-checks on `BotRandomOpponentName()` return**
   - If all opponents are dead or spectators, the function initializes the `opponents[]` array to `[0]` and returns `EasyClientName(0, ...)`.
   - **Risk:** If client 0 doesn't exist or is invalid, chat message could be malformed.
   - **Actual risk level: Low** — the function is only called in game modes with active players (gated by `BotNumActivePlayers()`); client 0 is almost always valid.

4. **Hardcoded chat template names (e.g., `"game_enter"`, `"death_rail"`) are not validated**
   - If botlib's script files don't define a template, `BotAI_BotInitialChat()` silently fails to queue a message.
   - **Risk:** Typos in template names cause chat to silently drop.
   - **Actual risk level: Low** — templates are defined in well-tested botlib script files and rarely change.

5. **MissionPack `#ifdef` splits chat logic**
   - Death categories and weapon names differ between base Q3 and MissionPack (e.g., Nailgun, Chaingun, Kamikaze).
   - **Risk:** If compiling for wrong mod, chat references invalid weapons or MOD codes.
   - **Actual risk level: Low** — build system enforces consistent `MISSIONPACK` flag across all modules.
