# ui/menudef.h

## File Purpose
A pure preprocessor-definition header that declares all symbolic constants used by the Quake III Arena menu/UI scripting system. It defines item types, visual styles, feeder IDs, owner-draw widget IDs, conditional display flags, and voice-chat command strings shared between the cgame, UI, and menu scripting layers.

## Core Responsibilities
- Define numeric constants for all UI item widget types (text, button, slider, listbox, etc.)
- Define alignment and text-style rendering flags for menu items
- Define window border and background-fill style constants
- Enumerate list-box feeder IDs that drive dynamic data sources (maps, servers, players, etc.)
- Provide bitmask flags for conditional HUD/UI element visibility (`CG_SHOW_*`, `UI_SHOW_*`)
- Enumerate owner-draw widget IDs for both the cgame HUD layer (`CG_*`, 1–69) and the UI layer (`UI_*`, 200–256)
- Declare string constants for all voice-chat commands used in team play

## Key Types / Data Structures
None. This file contains only `#define` preprocessor constants; no structs, enums, typedefs, or functions are declared.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure constants header.

## Control Flow Notes
This header is passive — it has no runtime behavior. It is `#include`d by:
- `code/cgame/` sources to interpret HUD owner-draw IDs and `CG_SHOW_*` display flags
- `code/ui/` and `code/q3_ui/` sources to interpret `UI_SHOW_*` flags and `UI_OWNERDRAW_*` IDs
- The menu script parser (`.menu` / `.txt` files compiled via macro expansion) to translate designer-facing symbol names into integers at build time

The constants act as a shared contract between the UI scripting system and both the cgame and UI modules, which register owner-draw callbacks keyed on these IDs during initialization.

## External Dependencies
- No includes.
- All constants are consumed by external translation units; none are defined or implemented here.

## Notes
- `CG_OWNERDRAW_BASE` starts at 1; `UI_OWNERDRAW_BASE` starts at 200 — the gap (70–199) is reserved or unused, keeping the two namespaces distinct.
- Several `FEEDER_*` comments are copy-pasted ("team members for team voting") for entries that clearly have different purposes (demos, scoreboard, cinematics) — a minor documentation inconsistency in the original source.
- `CG_SHOW_2DONLY (0x10000000)` is non-contiguous with the other `CG_SHOW_*` flags, suggesting it was added late.
- Voice-chat string constants are plain C string literals used as command tokens passed to the game/bot AI layer.
