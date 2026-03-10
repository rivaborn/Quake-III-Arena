# code/game/be_ai_char.h

## File Purpose
Public API header for the bot character system, exposing functions to load, query, and free bot personality/skill profiles. It defines the interface through which game code retrieves typed characteristic values (float, integer, string) from a named character file.

## Core Responsibilities
- Declare the bot character load/free lifecycle functions
- Expose typed accessors for individual bot characteristics by index
- Provide bounded variants of numeric accessors to clamp values within caller-specified ranges
- Declare a global shutdown function to release all cached character data

## Key Types / Data Structures
None. Characters are referenced by opaque `int` handles; no structs are declared in this header.

## Global / File-Static State
None declared in this file.

## Key Functions / Methods

### BotLoadCharacter
- Signature: `int BotLoadCharacter(char *charfile, float skill)`
- Purpose: Loads a bot character definition from a file, parameterized by a skill level.
- Inputs: `charfile` — path to the character definition file; `skill` — floating-point skill scalar.
- Outputs/Return: Integer handle identifying the loaded character; likely 0 or -1 on failure.
- Side effects: Allocates character data internally; result must be freed via `BotFreeCharacter`.
- Calls: Not inferable from this file.
- Notes: The `skill` parameter likely blends or selects characteristic values for difficulty scaling.

### BotFreeCharacter
- Signature: `void BotFreeCharacter(int character)`
- Purpose: Releases resources associated with a previously loaded character handle.
- Inputs: `character` — handle returned by `BotLoadCharacter`.
- Outputs/Return: void.
- Side effects: Frees internal character allocation.
- Calls: Not inferable from this file.

### Characteristic_Float / Characteristic_BFloat
- Signature: `float Characteristic_Float(int character, int index)` / `float Characteristic_BFloat(int character, int index, float min, float max)`
- Purpose: Retrieves a float characteristic by index; the `B` variant clamps the result to `[min, max]`.
- Inputs: `character` handle, `index` into the characteristic table; `B` variant adds clamp bounds.
- Outputs/Return: The characteristic value as a float.
- Side effects: None expected.
- Calls: Not inferable from this file.

### Characteristic_Integer / Characteristic_BInteger
- Signature: `int Characteristic_Integer(int character, int index)` / `int Characteristic_BInteger(int character, int index, int min, int max)`
- Purpose: Retrieves an integer characteristic by index; the `B` variant clamps the result.
- Inputs/Outputs: Analogous to the float variants above.
- Side effects: None expected.

### Characteristic_String
- Signature: `void Characteristic_String(int character, int index, char *buf, int size)`
- Purpose: Copies a string characteristic into a caller-provided buffer.
- Inputs: `character` handle, `index`, destination `buf`, buffer `size`.
- Outputs/Return: void; result written into `buf`.
- Side effects: Writes to caller-supplied buffer; `size` guards against overflow.

### BotShutdownCharacters
- Signature: `void BotShutdownCharacters(void)`
- Purpose: Frees all cached/loaded bot character data at shutdown.
- Side effects: Global deallocation of all character resources.

## Control Flow Notes
This header is consumed by game-side bot AI code (e.g., `ai_main.c`) during bot initialization. `BotLoadCharacter` is called at bot spawn; accessors are called during AI decision-making frames; `BotFreeCharacter` is called on bot removal; `BotShutdownCharacters` is called at server/game shutdown. Implementations live in `code/botlib/be_ai_char.c`.

## External Dependencies
- No includes in this header.
- All function bodies defined in `code/botlib/be_ai_char.c` (defined elsewhere).
- Consumed via the botlib interface layer (`be_interface.c`) or directly by game bot code.
