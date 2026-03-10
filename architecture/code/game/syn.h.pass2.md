# code/game/syn.h — Enhanced Analysis

## Architectural Role
This header serves as the **context vocabulary** for the bot chat system, bridging runtime personality/synonym data files and the AI chat pipeline across both botlib and the game VM. It defines a bitmask protocol that allows precompiled bot dialogue data (synonyms, responses, named character voices) to specify situational applicability without hardcoding game logic. The constants directly reflect Quake III's gametypes (CTF, Obelisk, Harvester) and team structure, making context matching data-driven rather than logic-driven.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/ai_chat.c`** — Game VM AI personality/chat module consumes these context flags when matching bot dialogue templates to current game state
- **`code/botlib/be_ai_chat.c`** — Botlib's chat synthesis layer uses these constants to filter and apply contextualized bot responses during runtime
- Bot personality/synonym data files (parsed at load time) embed these context values to restrict when specific dialogue is valid

### Outgoing (what this file depends on)
- None; this is a pure header with no external dependencies

## Design Patterns & Rationale

**Bitmask enumeration for composable context:**
- All constants are powers of 2 (1, 2, 4, 8, 16, ..., 1024), enabling bitwise OR composition
- A bot's current context is built by combining applicable flags: e.g., `CONTEXT_CTFREDTEAM | CONTEXT_NEARBYITEM` when a red-team bot is near an item in CTF
- Allows efficient lookup/filtering in chat synonym tables without branching on individual flags

**Gametype-aware taxonomy:**
- Base contexts: `CONTEXT_NORMAL` (always valid), `CONTEXT_NEARBYITEM`, `CONTEXT_REPLY`
- Team/gametype-specific: separate red/blue flags for CTF (`CONTEXT_CTF{RED,BLUE}TEAM`), Obelisk (`CONTEXT_OBELISK{RED,BLUE}TEAM`), Harvester (`CONTEXT_HARVESTER{RED,BLUE}TEAM`)
- Reflects the engine's multi-gametype design; data files can be conditionally active based on the loaded game mode

**Data-driven personality system:**
- Rather than embedding dialogue logic in C, Quake III ships precompiled synonym/response tables (binary format) that reference these context bits
- Allows mods and custom bot personalities to author new dialogue without recompiling the engine
- `CONTEXT_NAMES` (1024) separates name-keyed dialogue (character-specific callouts) from generic responses

## Data Flow Through This File

```
[Bot personality/synonym data files (.aas, binary)]
                          ↓
       [Runtime parser reads context flags]
                          ↓
  [Game/botlib chat modules compare against current]
  [bot state: team, item proximity, game type, etc.]
                          ↓
  [Bitwise AND: current_context & synonym_context_mask]
                          ↓
     [Match → synthesize dialogue; no match → skip]
```

Game state drives context assembly: `SV_GameFrame` → bot AI tick → chat evaluation → context mask lookup in synonym table.

## Learning Notes

**What developers learn from this file:**
1. **Context-driven dialogue:** Q3A's bot AI separates **what** to say (data) from **when** to say it (context flags), predating modern dialogue trees
2. **Gametype polymorphism:** The duplication of team flags across three gametypes shows the engine's approach to mode-specific behavior: define constants for each mode, let data files decide applicability
3. **Bitmask idioms:** Powers of 2 for composability are idiomatic to early 2000s game engines; modern engines often use enums or string keys, but bitmasks remain efficient for bit-level filtering

**Idiomatic to Q3A / era:**
- **Data-driven bot personality:** Unlike many games where AI dialogue is hardcoded, Q3A ships bot personalities as separate binary blobs. This enabled easy mod authoring and personality swaps without recompilation—a best practice for an open-source engine.
- **Minimal constants exposure:** The header is deliberately minimal; the actual synonym/personality data format and loading is opaque to the engine, reducing coupling.

## Potential Issues
No issues are inferable from the code or architecture context. The bitmask design is well-suited to its use case and the constant set is stable across the three major gametypes. The single `CONTEXT_NAMES` catch-all is large (1024 = next power of 2 after 512), providing a clean gap for future expansion.
