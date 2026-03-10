# code/game/chars.h

## File Purpose
Defines integer constants (characteristic indices) used to index into a bot's personality/behavior data structure. Each constant maps a named behavioral trait to a slot number understood by the bot AI and botlib systems.

## Core Responsibilities
- Enumerate all bot characteristic slot indices (0–48)
- Categorize traits into logical groups: identity, combat, chat, movement, and goal-seeking
- Provide a shared vocabulary between the game module and botlib for reading/writing bot personality values

## Key Types / Data Structures
None. This file contains only `#define` preprocessor constants.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure constants header.

## Control Flow Notes
This file is passive — it participates in init/frame flow only insofar as other systems reference these indices:
- **Init**: Bot characteristic files (`.c` personality scripts) are parsed at bot spawn time; values are stored in arrays indexed by these constants.
- **Frame**: The bot AI decision functions (`be_ai_move.c`, `be_ai_goal.c`, `be_ai_chat.c`, etc.) read characteristic slots by number to scale behavior each frame.
- Not involved in rendering or shutdown directly.

## External Dependencies
- No `#include` directives; this header is self-contained.
- **Defined elsewhere / consumers:**
  - `botlib/be_ai_char.c` — reads/writes characteristic values using these indices
  - `game/ai_main.c`, `game/ai_dmq3.c`, etc. — pass these constants to botlib API calls such as `trap_Characteristic_Float` / `trap_Characteristic_String`
  - `botlib/botlib.h` — declares the `BotCharacteristic_*` API that accepts these index values

## Notes
- Index **48** (`CHARACTERISTIC_WALKER`) is out of sequential order (skips past 38–47); index 38 is `CHARACTERISTIC_WEAPONJUMPING`. This gap/ordering irregularity suggests the characteristic was added after the original numbering was laid out.
- Several characteristics are marked `//use this!!` (`CHARACTERISTIC_GRAPPLE_USER`, `CHARACTERISTIC_VENGEFULNESS`), indicating features that were planned but may have had incomplete implementation at ship time.
- Type annotations in comments (`//float [0,1]`, `//string`, `//integer`) are documentation only — enforcement is entirely up to the botlib parser.
