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

## External Dependencies
- No includes.
- All constants are consumed by external translation units; none are defined or implemented here.


