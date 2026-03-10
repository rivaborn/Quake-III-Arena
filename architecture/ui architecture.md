# Subsystem Overview

## Purpose
The `ui` subsystem provides the symbolic constant layer that drives the Quake III Arena menu and HUD scripting system. It defines the shared vocabulary of widget types, rendering styles, data-feeder IDs, owner-draw IDs, and conditional visibility flags consumed by the `cgame`, `ui`, and menu-script layers at compile time.

## Key Files
| File | Role |
|------|------|
| `ui/menudef.h` | Pure preprocessor-definition header; declares all numeric and string constants for the menu/UI scripting system |

## Core Responsibilities
- Define numeric constants for every UI item widget type (text, button, slider, listbox, and others)
- Define alignment and text-style rendering flags applied to menu items
- Define window border and background-fill style constants
- Enumerate list-box feeder IDs that bind dynamic runtime data sources (maps, servers, players, etc.) to UI widgets
- Provide bitmask flags for conditional HUD/UI element visibility (`CG_SHOW_*`, `UI_SHOW_*`)
- Enumerate owner-draw widget IDs for the cgame HUD layer (`CG_*`, range 1–69) and the UI layer (`UI_*`, range 200–256)
- Declare string constants for all voice-chat commands used in team-play mode

## Key Interfaces & Data Flow
- **Exposes to others:** Preprocessor constants (`#define`) only; no runtime symbols, functions, or data structures are exported. All consuming translation units (`cgame`, `ui`, menu-script compiler) `#include` this header to obtain a shared, stable numeric namespace.
- **Consumes from other subsystems:** Nothing — the file has no `#include` directives and carries no external dependencies.

## Runtime Role
- This file participates solely at compile time. It carries no initialization, per-frame, or shutdown logic of its own.
- At runtime, the constants it defines govern how the menu-script interpreter classifies widgets, selects renderers, resolves feeder callbacks, and evaluates conditional visibility expressions — but that execution occurs in the consuming subsystems (`cgame`, `ui`), not here.

## Notable Implementation Details
- The header is a pure-constant file with no implementation; including it in multiple translation units across `cgame` and `ui` is safe and intentional.
- The owner-draw ID ranges are explicitly partitioned by layer (`CG_*` 1–69 vs. `UI_*` 200–256), preventing collisions between the cgame HUD owner-draw dispatcher and the UI menu owner-draw dispatcher.
- The `CG_SHOW_*` / `UI_SHOW_*` bitmask flags encode conditional display logic directly into the menu-script data, allowing the script interpreter to gate element visibility without game-code changes.
- Voice-chat command strings are co-located here alongside visual constants, indicating this header serves as the single source of truth for the entire menu-scripting contract rather than being split by concern.
