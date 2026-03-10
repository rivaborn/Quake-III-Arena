# code/game/syn.h

## File Purpose
Defines bitmask constants for bot chat context flags used by the AI chat system. These flags identify the situational context in which a bot chat synonym or response is valid.

## Core Responsibilities
- Define a bitmask enumeration of chat/behavior contexts for the bot AI
- Distinguish team-specific contexts (CTF red/blue, Obelisk, Harvester)
- Provide a catch-all `CONTEXT_ALL` mask for context-agnostic entries

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header of preprocessor definitions.

## Control Flow Notes
This file is consumed at compile time. It is `#include`d by the bot AI chat and behavior files (e.g., `be_ai_chat.c`, `ai_chat.c`) to interpret context bitmasks embedded in bot personality/synonym data files parsed at runtime. No runtime control flow resides here.

## External Dependencies
- No includes or external symbols. Standalone macro-only header.

---

**Notes on constants:**

| Constant | Value | Meaning |
|---|---|---|
| `CONTEXT_ALL` | `0xFFFFFFFF` | Matches any context |
| `CONTEXT_NORMAL` | `1` | Default/generic context |
| `CONTEXT_NEARBYITEM` | `2` | Bot is near an item |
| `CONTEXT_CTFREDTEAM` | `4` | CTF, red team |
| `CONTEXT_CTFBLUETEAM` | `8` | CTF, blue team |
| `CONTEXT_REPLY` | `16` | Replying to another chat message |
| `CONTEXT_OBELISKREDTEAM` | `32` | Overload gametype, red team |
| `CONTEXT_OBELISKBLUETEAM` | `64` | Overload gametype, blue team |
| `CONTEXT_HARVESTERREDTEAM` | `128` | Harvester gametype, red team |
| `CONTEXT_HARVESTERBLUETEAM` | `256` | Harvester gametype, blue team |
| `CONTEXT_NAMES` | `1024` | Context for name-specific synonyms |

Values are powers of two, designed to be OR-combined into a composite context mask for lookup and filtering.
