# code/game/chars.h — Enhanced Analysis

## Architectural Role

This file defines the characteristic index vocabulary for the bot personality API, which bridges the game VM (client-facing) and botlib engine library (server-side AI stack). Rather than a simple data structure header, it's an **API contract** for cross-VM-boundary personality queries: the game VM uses these indices to call `trap_Characteristic_Float(botNum, CHARACTERISTIC_*)` and `trap_Characteristic_String(botNum, CHARACTERISTIC_*)` syscalls that reach back into botlib's `struct bot_personality_t` arrays. The VM-serialized indices must never change, making this an intentionally stable ABI surface.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/ai_dmq3.c`** — base DM AI reads `CHARACTERISTIC_ATTACK_SKILL`, `CHARACTERISTIC_AGGRESSION`, `CHARACTERISTIC_CAMPER`, `CHARACTERISTIC_ALERTNESS`, `CHARACTERISTIC_FIRETHROTTLE` to modulate combat FSM behavior each frame
- **`code/game/ai_dmnet.c`** — team DM/CTF AI reads same combat traits plus `CHARACTERISTIC_VENGEFULNESS` (if implemented) to select targets and camping spots
- **`code/game/ai_chat.c`** — chat system reads all `CHARACTERISTIC_CHAT_*` indices (21–35) to weight message-generation probabilities at emotional/social decision points
- **`code/game/be_ai_move.h` / botlib movement API** — reachability validation reads `CHARACTERISTIC_WEAPONJUMPING`, `CHARACTERISTIC_GRAPPLE_USER` to determine if a bot can execute special movement tricks
- **`code/botlib/be_ai_char.c`** — loads personality files (`.c` bot data) and stores values in indexed arrays; directly implements the characteristic slot storage

### Outgoing (what this file depends on)
- None. This is a pure constants header with no `#include` directives and no external dependencies.

## Design Patterns & Rationale

**Opaque Indexed API Boundary:**  
Characteristics are not passed as structured records; the game VM does not have direct visibility into `bot_personality_t` layout. Instead, botlib exports indexed accessor syscalls (`BotCharacteristic_Float`, `BotCharacteristic_String`) that the game VM invokes by index. This decouples bot personality schema evolution from VM binary stability.

**Logical Grouping Over Flat Enumeration:**  
Despite being a flat index range (0–48), the header groups characteristics by behavioral domain:
- **Identity** (name, gender): rarely queried; loaded once at spawn
- **Combat** (skill, accuracy, aim): queried per-frame during weapon selection and aiming
- **Chat** (file, CPM, tendencies): batched at specific event triggers (kills, deaths, spawn)
- **Movement** (crouch, jump, grapple): consulted during reachability validation and travel-type execution
- **Goal** (aggression, camping, item weights): polled during goal selection FSM

This grouping mirrors the internal `be_ai_*.c` module structure, suggesting characteristics were designed to be cache-friendly when fetched in domain-specific clusters.

**Stable Index Numbering for QVM Serialization:**  
The gap at index 48 (`CHARACTERISTIC_WALKER` placed after 38, skipping 39–47) indicates a characteristic was added post-hoc without renumbering. Unlike in-memory structs, VM bytecode compiled years ago must not break if enum values shift. These constants are **never** removed or reordered; only appended.

**Type Annotations as Documentation:**  
Comments like `//float [0, 1]`, `//string`, `//integer [1, 4000]` exist solely for human reference. Botlib's parser is the sole enforcer of type and range; violations are not caught by the C compiler.

## Data Flow Through This File

**Load-Time (bot spawn):**
1. Server calls `BotAI_LoadCharacteristics(botNum)` 
2. Botlib parses the bot's `.c` personality file (e.g., `bots/visor_bot.c`)
3. For each parsed line (e.g., `"attack_skill" "0.75"`), botlib maps the key to a characteristic index (2, in this case) and stores `personality[2] = 0.75f`
4. Characteristics are now resident in botlib's singleton `aasworld.bots[botNum].character`

**Runtime (per-frame AI decision):**
1. Game AI code calls `trap_Characteristic_Float(botNum, CHARACTERISTIC_ATTACK_SKILL)` 
2. Syscall handler (`SV_GameSystemCalls`) dispatches to `BotCharacteristic_Float(botNum, 2)`
3. Botlib returns `personality[2]`, used immediately (e.g., to scale strafe radius in `be_ai_move.c`)
4. No caching; fresh reads every frame ensure dynamic difficulty adjustments are live

**Unidirectional Read-Only:**  
The game VM reads characteristics but never writes them post-spawn. This asymmetry reflects the architecture: botlib owns personality data; the game VM is a consumer.

## Learning Notes

**Characteristic Indices as a Stable API Surface:**  
Modern engines use versioned APIs or protocol buffers for cross-module communication. Q3A's approach—a flat integer enumeration—is era-appropriate (late 1990s) and highlights a design constraint: the game VM is precompiled bytecode shipped in `.pk3` archives. Reordering or removing a characteristic would silently corrupt all existing bot files and break third-party mods.

**Incomplete Features (Markers for Post-Shipping Development):**  
The `//use this!!` comments on `CHARACTERISTIC_GRAPPLE_USER` (39) and `CHARACTERISTIC_VENGEFULNESS` (43) suggest feature flags for planned mechanics:
- **Grapple Hook**: The hooked weapon exists in the arena, but bot prioritization of it may not be fully wired.
- **Revenge Mechanics**: Bots should prefer to hunt down whoever killed them most recently—a classic game psychology feature rarely implemented at that era.

This pattern (incomplete feature hints left in source) is common in shipped games and illustrates how personality subsystems were extended iteratively during development.

**Contrast with Modern Personality Systems:**  
Contemporary engines (Unreal, Unity) expose bot/NPC behavior trees or behavior parameters as structured records (classes with `@Serializable` annotations, YAML schemas). Q3A's flat-indexed approach is less discoverable but more robust across version boundaries—a tradeoff favoring longevity of third-party content.

## Potential Issues

**Index 48 Ordering Anomaly:**  
`CHARACTERISTIC_WALKER` (index 48) logically belongs near `CHARACTERISTIC_CROUCH` / `CHARACTERISTIC_JUMPER` (36–37), not after `CHARACTERISTIC_FIRETHROTTLE` (47). If a bot file mistakenly references an index based on source comment ordering rather than actual indices (e.g., assuming walker ≈ 38), it will read `CHARACTERISTIC_WEAPONJUMPING` instead. Unlikely in practice (botlib parses by name, not index), but a subtle foot-gun for hardcoded clients.

**No Bounds Validation at VM Boundary:**  
If game VM code calls `trap_Characteristic_Float(botNum, 999)`, the syscall handler should validate the index against the characteristic range [0–48]. If this check is missing, memory corruption in botlib's personality arrays could occur. (Not inferable from this file alone; requires inspection of `SV_GameSystemCalls`.)
